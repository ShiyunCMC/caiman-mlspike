# MLSpike Workflow From CaImAn Calcium Traces

This document records the agreed workflow for converting CaImAn calcium outputs into MLSpike-ready traces and saving MLSpike's native binned spike output.

## Initial Inputs

- `calcium_timestamps`: actual calcium frame timestamps, in seconds.
- `C`: CaImAn temporal component array.
- `YrA`: CaImAn residual/background-corrected component array.

For each cell/neuron `i`, start with:

```matlab
Fraw = C(i,:) + YrA(i,:);
t = calcium_timestamps(:)';
Fraw = Fraw(:)';
```

`C + YrA` is the calcium trace passed to MLSpike.

## 1. Estimate Nominal Sampling Interval

MLSpike assumes a uniform sampling interval, so use the timestamp vector only to estimate the nominal frame interval and identify irregular gaps.

```matlab
dt = median(diff(t));
```

## 2. Split Around Large Gaps

Small dropped-frame gaps can be interpolated. Larger gaps should not be filled, because interpolation across long missing intervals can invent calcium dynamics.

Use this practical threshold:

```matlab
large_gap = diff(t) > 2.5 * dt;

seg_start = [1, find(large_gap) + 1];
seg_end   = [find(large_gap), numel(t)];
```

Each resulting segment is processed separately.

## 3. Interpolate Each Segment Onto A Regular Grid

For each segment:

```matlab
idx = seg_start(s):seg_end(s);

t_seg = t(idx);
F_seg = Fraw(idx);

tq = t_seg(1):dt:t_seg(end);
Fq = interp1(t_seg, F_seg, tq, 'linear');
```

Use linear interpolation. Avoid spline interpolation because it can create overshoot or ringing around dropped frames.

## 4. Run MLSpike Per Segment

Run MLSpike separately on each regularized segment using the same nominal `dt`.

Conceptually:

```matlab
n_seg = run_mlspike(Fq, dt, params);
```

`n_seg` is the MLSpike MAP estimate of spike count per regular imaging bin. It is not a continuous-time electrophysiology spike timestamp list.

## 5. Save Native Binned Output Only

The agreed output should save only:

- `t_grid`: regular time grid for the MLSpike bins.
- `n_binned`: MLSpike inferred spike count per bin.

Do not save timestamp-expanded spike trains as the primary output.

For each cell, concatenate segment outputs:

```matlab
t_grid_all = [];
n_binned_all = [];

for s = 1:numel(seg_start)
    % build tq and Fq for this segment
    % run MLSpike to obtain n_seg

    t_grid_all = [t_grid_all; tq(:)];
    n_binned_all = [n_binned_all; n_seg(:)];
end

mlspike_output(i).t_grid = t_grid_all;
mlspike_output(i).n_binned = n_binned_all;
```

If downstream code later needs timestamp-style events, those can be derived from `t_grid` and `n_binned`, but they should not be the saved output of this workflow.
