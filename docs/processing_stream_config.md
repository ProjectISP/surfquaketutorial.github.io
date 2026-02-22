# Processing Stream Options

## Rotate Seismograms

This option rotates the three-component traces in the stream to either the Great Arc Circle (GAC) or a user-defined angle and inclination.

- **`type`**: Defines the rotation transformation.
    - `NE->RT`: Rotates North and East components to Radial and Transverse.
    - `RT->NE`: Rotates Radial and Transverse components back to North and East.
    - `ZNE->LQT`: Rotates from a left-handed Z, North, East system to an LQT (ray-based, right-handed) coordinate system.
    - `LQT->ZNE`: Rotates from an LQT system back to a left-handed Z, North, East system.

- **`method`**: Specifies the rotation method.
    - `GAC`: Rotates according to the Great Arc Circle. Requires event metadata.
    - `FREE`: Rotates to a user-defined angle and inclination (used for `LQT` transformations).


- **`angle`** (float) – The back azimuth from station to source in degrees.
- **`inclination`** (float). Inclination of the ray at the station in degrees. Only necessary for three component rotations.


```yaml
Analysis:
  process_1:
    name: 'rotate'
    type: 'NE->RT'
    method: 'GAC'
    angle: 30.0
    inclination: 0.0
```

---

## Cross Correlation

Performs cross-correlation between waveforms.

- **`mode`**: Defines how the cross-correlation is computed, following the [NumPy `correlate`](https://numpy.org/doc/stable/reference/generated/numpy.correlate.html) method:
    - `full`: Returns the complete cross-correlation function. Includes all shifts, even partial overlaps. Best if you want to scan the entire signal.
    - `valid`: Returns only correlation values where the signals fully overlap. Safer when you want to avoid edge effects.
    - `same`: Returns output of the same length as the input data, centered around zero lag.

- **`normalize`**: Specifies how traces are normalized before correlation.
    - `full`: Uses zero-normalized cross-correlation (ZNCC). Each point is normalized based on local signal energy and optional demeaning. This provides a correlation coefficient between -1 and 1 and is the most statistically meaningful
    - `naive`:Normalizes both signals by their global standard deviation. Quicker but may introduce bias if signals have amplitude variation.
    - `none`: No normalization is applied. Raw amplitudes are used, which may be sensitive to trace scaling or noise.
- **`reference_idx`**: Index of the trace to use as the reference for correlation (typically one of the traces in the stream).
- **`strict`**: If `True`, enforces strict alignment and compatibility between trace lengths and sampling rates.


```yaml
Analysis:
  process_1:
    name: 'cross_correlate'
    mode: 'full'
    normalize: 'full'
    reference_idx: 0
    strict: True
```

---

## Stack

Stacks multiple traces using a specified method.

- **`method`**: Specifies the stacking method:
    - `linear`: Simple arithmetic mean across traces.
    - `pw`: Phase-weighted stacking, emphasizing coherent phase information. Order 1 is the parameter of this method. Recommended 1.
    - `root`: nth-root stack

- **`order`**: The exponent applied in phase-weighted stacking. A higher value gives stronger weighting to coherent signals.

```yaml
  process_1:
    name: 'stack'
    method: 'pw'
    order: 1
```

---


## Synchronization

In many processing workflows, it is necessary to synchronize all traces to a common reference point in time. This ensures consistent alignment for further processing steps such as stacking, correlation, or rotation.

Two synchronization methods are supported:

- **`starttime`**: Aligns all traces to a common absolute start time as defined in their metadata. This method assumes all traces are recorded simultaneously and no additional time correction is needed.


- **`MCCC`**: (Multi-Channel Cross-Correlation): Dynamically aligns traces based on their waveform similarity by computing relative delays. This method maximizes coherence across all channels and is especially effective in noisy conditions or when arrival times differ slightly due to path effects or instrument drift. The MCCC approach is based on: VanDecar, J. C., & Crosson, R. S. (1990).

```yaml
Analysis:
  process_1:
    name: 'synch'
    method: 'starttime' # or MCCC
```

---


## shift

This method applies temporal alignment to traces in a stream using one of several strategies. It is useful when aligning waveforms for stacking, correlation, or interpretation, and offers flexible options including theoretical models, observed picks, or manual time shifts. Please review **traces header info** to fully understand this method.

phase_theo:
Aligns traces based on theoretical arrival times. The expected arrival must be stored in trace.stats.geodetic['arrivals'] under the specified phase name.
Use case: When synthetic or model-based timing is reliable (e.g., teleseismic body waves).

- **`phase`**: Aligns traces to observed picks, typically generated from prior picking routines or manual annotation. Pick information must be stored in trace.stats.picks. Use case: When you trust the quality of observed picks more than theoretical estimates.

- **`time_shifts`**: Applies manual time shifts to each trace, using a list of per-trace time offsets in seconds (float).
Use case: When you want full control over the alignment or need to apply empirical or experiment-specific shifts.

Note: If both phase and time_shifts are provided, phase takes precedence.
If none of the options are valid, the stream is returned unchanged. 

```yaml
Analysis:
  process_1:
    name: 'shift'
    phase_theo: 'P'
    # Optional alternatives:
    # phase: 'P'
    # time_shifts: [0.1, -0.2, 0.05] in seconds
```

---

## Concatenate

This method concatenates multiple time-continuous trace segments within the same stream. It is commonly used to join split traces that were originally recorded as separate files or segments (e.g., due to daily files, instrument restarts, or transmission delays). Internally, it uses ObsPy's Stream.merge() functionality with:

method=1: Linear interpolation for filling gaps

fill_value='interpolate': Gaps are filled using interpolated values

Behavior
Overlapping traces are merged into a single continuous trace. Gaps are filled using linear interpolation between adjacent segments. Non-overlapping or incompatible traces (e.g. different IDs or sampling rates) are left unmerged.

Note: Trace concatenation is automatically done when command is **surfquake process** or **surfquake process_daily** but not for **surfquake quick** for full control. 


```yaml
Analysis:
  process_1:
    name: 'concat'
```



