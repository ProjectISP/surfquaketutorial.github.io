# Processing Stream Options

Stream methods use **several traces together**: the three components of a station, a reference trace and the others, the stations of an array, or the whole stream. Typical examples are rotating components, cross-correlating, stacking or beamforming.

They are written in the same `Analysis` section as the trace methods:

```yaml
Analysis:
  process_1:
    name: 'rmean'
    method: 'linear'
  process_2:
    name: 'taper'
    method: 'cosine'
    max_percentage: 0.05
  process_3:
    name: 'filter'
    type: 'bandpass'
    fmin: 0.5
    fmax: 8.0
    corners: 4
    zerophase: True
  process_4:
    name: 'stack'
    method: 'pw'
    order: 2
```

!!! note "Order of execution"
    Trace methods and stream methods run as two blocks. By default (`-op trace_first`) all trace methods are applied first, in their order, and then all stream methods, in their order. With `-op stream_first` the stream methods run first. The position of a stream step among the trace steps in the config file does not change this.

!!! tip "General rules"
    - Each `process_N` key must be **unique**. A repeated key silently replaces the earlier step.
    - Times in seconds, frequencies in Hz. Write decimals with a dot (`0.5`, `2.0`).
    - Most stream methods need traces with the **same sampling rate**. Use `resample` first if needed.
    - Some methods only write results to disk and pass the **same traces** to the next step: this is noted as *stream unchanged*.
    - A template with every stream method is available with `surfquake make_config -c stream_config`.

---

## A. Merging, alignment and stacking

### Concatenate

Joins the time-continuous segments of the same channel into a single trace. Data are often split into several segments (daily files, instrument restarts, transmission delays). Internally it uses ObsPy `Stream.merge()` with linear interpolation:

- Overlapping segments are merged into one continuous trace.
- Gaps are filled by linear interpolation between the adjacent samples.
- Traces with different ids or sampling rates are left unmerged.

```yaml
Analysis:
  process_1:
    name: 'concat'
```

Note: concatenation is done automatically by **surfquake processing** and **surfquake processing_daily**, but not by **surfquake quick**, which gives you full control.

---

### Synchronization

Puts all traces on a common time reference, which is needed before stacking or comparing waveforms.

- **`starttime`**: gives all traces the same (dummy) start time. Use it to overlay or stack records that start at a meaningful point but belong to different events or dates, for example traces already cut around the P arrival.
- **`MCCC`** (Multi-Channel Cross-Correlation): measures the relative delays between all **pairs** of traces by cross-correlation, and solves for the set of delays that best fits all pairs in a least-squares sense. Using all pairs together makes the alignment much more robust than correlating each trace with a single reference, which is very effective with noisy data, or when arrivals differ due to path effects or clock drifts.

```yaml
Analysis:
  process_1:
    name: 'synch'
    method: 'MCCC'
```

> **Scientific Reference**:
>
> VanDecar, J. C., & Crosson, R. S. (1990). Determination of teleseismic relative phase arrival times using multi-channel cross-correlation and least squares. *Bulletin of the Seismological Society of America*, 80(1), 150–169.

---

### Shift

Moves the traces in time. It is used to align waveforms on a phase before stacking, comparing or plotting them. Please review [Trace Header](https://projectisp.github.io/surfquaketutorial.github.io/trace_header_documentation/){: target="_blank" rel="noopener noreferrer" } to fully understand this method. Use **one** of the options:

| Option | Description |
|--------|-------------|
| `phase_theo` | Align on a **theoretical** arrival time, stored in `trace.stats.geodetic['arrivals']` (computed when the traces are processed with an event file). Use it when model-based timing is reliable, e.g. teleseismic body waves. |
| `phase` | Align on an **observed pick**, stored in `trace.stats.picks`. Use it when you trust your picks more than the theoretical times. |
| `time_shifts` | Add a fixed offset (seconds) to the start time of each trace, one value per trace, in stream order. For full manual control. |

With `phase_theo` or `phase`, the samples before the phase are removed and every trace starts at the same time (1970-01-01, i.e. t = 0 at the phase). Traces without that phase are skipped with a warning.

If several options are given, the order of priority is `phase_theo`, then `phase`, then `time_shifts`. If none is valid, the stream is returned unchanged.

```yaml
Analysis:
  process_1:
    name: 'shift'
    time_shifts: [-5.0, 2.0]
```

```yaml
Analysis:
  process_1:
    name: 'shift'
    phase_theo: 'P'
    time_shifts: []
```

ℹ️ The key `time_shifts` must always be present, even when you align by phase. In that case write an empty list.

---

### Cut Stream

Cuts all traces of the stream to the same window. Unlike the trace method `cut`, the traces are **padded with zeros** where they have no data, so all of them end with the same length, which is required by array methods.

| Parameter | Description |
|-----------|-------------|
| `method` | `absolute` (`start`, `end`), `phase` (`phase_name`, `t_before`, `t_after` around each trace's pick) or `reference` (`t_before`, `t_after` around the reference time). |

```yaml
Analysis:
  process_1:
    name: 'cut_stream'
    method: 'absolute'
    start: 2018-08-21T00:27:15
    end: 2018-08-21T00:31:45
```

---

### Stack

Combines all traces into **one** trace (id `NET.STA..STK`). Summing aligned traces reinforces the coherent signal, while the incoherent noise partially cancels: with N traces and random noise, the signal-to-noise ratio improves by about $\sqrt{N}$. The traces must be aligned first (see `shift` and `synch`); they are cut to the shortest length before stacking.

| `method` | Description |
|----------|-------------|
| `linear` | Arithmetic mean of the traces. Preserves amplitudes; the reference method. |
| `pw` | **Phase-weighted stack** (Schimmel & Paulssen, 1997). The linear stack is multiplied by the phase coherence of the traces (computed from their instantaneous phase), raised to the power `order`. Samples where the phases agree are kept; incoherent samples are suppressed. Excellent noise reduction for weak coherent signals, but amplitudes are no longer reliable. |
| `root` | **N-th root stack**: the N-th root of each sample is stacked (keeping its sign) and the result is raised back to the power N. It enhances coherent signals and suppresses incoherent noise, but distorts the waveform. |

| Parameter | Description |
|-----------|-------------|
| `order` | (`pw`, `root`) integer exponent. Higher values give stronger weighting to coherent signals. `pw`: 1–2 are usual. |

```yaml
Analysis:
  process_1:
    name: 'stack'
    method: 'pw'
    order: 2
```

> **Scientific Reference**:
>
> Schimmel, M., & Paulssen, H. (1997). Noise reduction and detection of weak, coherent signals through phase-weighted stacks. *Geophysical Journal International*, 130(2), 497–505.

---

## B. Components: rotation and polarisation

### Rotate Seismograms

The horizontal sensors record in the North and East directions, which are not related to the earthquake. Rotating to the **ray** coordinates separates the wave types: on the **Radial** (R) component, along the great circle from the source, you find P and SV energy and Rayleigh waves; on the **Transverse** (T) component, perpendicular to it, SH energy and Love waves. The 3D rotation **LQT** goes one step further and uses the ray direction itself: L along the P ray, Q (SV) and T (SH) perpendicular to it.

Before rotating, the components of each station are standardised to Z/N/E and cut to their common time window.

- **`type`**: the rotation.
    - `NE->RT`: North and East to Radial and Transverse.
    - `RT->NE`: Radial and Transverse back to North and East.
    - `ZNE->LQT`: Z, North, East to the ray-based L, Q, T system.
    - `LQT->ZNE`: L, Q, T back to Z, North, East.

- **`method`**: how the angles are obtained.
    - `GAC`: Great Arc Circle. The back-azimuth (and the inclination) come from the trace headers, so the event information must be available (e.g. processing with an event file).
    - `FREE`: the angles given by the user.

- **`angle`** (float): back-azimuth from the station to the source, in degrees (`FREE`).
- **`inclination`** (float): inclination of the ray at the station, in degrees (`FREE`). Only used by the LQT rotations; write `0.0` for `NE->RT`.

```yaml
Analysis:
  process_1:
    name: 'rotate'
    type: 'NE->RT'
    method: 'FREE'
    angle: 30.0
    inclination: 0.0
```

---

### Particle Motion

Analyses the **polarisation** of the ground motion at each station: the 3D path followed by a particle of the ground during the arrival. P waves are linearly polarised along the propagation direction, S waves perpendicular to it, and Rayleigh waves elliptical in the vertical-radial plane. The polarisation gives the back-azimuth and incidence angle of a wave from a single station, and helps to identify phases. *Stream unchanged.*

It needs complete Z-N-E (or Z-1-2, Z-Y-X) sets; the traces are cut to their common window. Please review [Particle Motion](https://projectisp.github.io/surfquaketutorial.github.io/particle_motion_documentation/){: target="_blank" rel="noopener noreferrer" }.

```yaml
Analysis:
  process_1:
    name: 'particle_motion'
    output_path: './particle_motion'
```

---

## C. Operations against a reference trace

These methods take the trace at position `reference_idx` as a reference and process every trace of the stream against it: **n traces in, n traces out**. The reference trace is processed with itself (autocorrelation, autodeconvolution).

### Cross Correlation & Convolution

**Cross-correlation** measures the similarity of two signals as a function of the time lag between them. The lag of its maximum is the delay between the signals, which is the basis of relative arrival times, template matching, repeating events and ambient noise interferometry (the correlation of the noise at two stations approximates the Green's function between them).

**Convolution** is the same operation with one of the signals reversed in time. It is the response of a linear system: for example, convolving a source time function with a Green's function gives a synthetic seismogram.

- **`name`**: `cross_correlate` or `convolve`.

- **`mode`**: which lags are returned, following the [NumPy `correlate`](https://numpy.org/doc/stable/reference/generated/numpy.correlate.html){: target="_blank" rel="noopener noreferrer" } method:
    - `full`: all lags, including partial overlaps. Best if you want to scan the entire signal.
    - `valid`: only the lags where the signals fully overlap. Safer when you want to avoid edge effects.
    - `same`: output of the same length as the input, centred around zero lag.

- **`normalize`**: how the correlation is normalised (not applied in convolution).
    - `full`: zero-normalised cross-correlation (ZNCC), normalised by the local energy of the signals. The result is a correlation coefficient between −1 and 1, the most statistically meaningful.
    - `naive`: normalised by the global energy of both signals. Faster, but it may be biased if the amplitude changes along the traces.

- **`reference_idx`**: index of the reference trace (int).
- **`trim`**: if `True`, all traces are first cut to the common time window (latest start, earliest end). Recommended, because then the zero lag means the same time for every pair.

The output traces are named `<ref_station>_<station>` with an `X` in the channel (e.g. `HXZ`).

```yaml
Analysis:
  process_1:
    name: 'cross_correlate'
    mode: 'full'
    normalize: 'full'
    reference_idx: 0
    trim: True
```

---

### Deconvolution

Deconvolution is the inverse of convolution: it **removes the reference signal** from the other traces. Dividing the spectrum of each trace by the spectrum of the reference cancels everything they share (source time function, instrument, common path) and leaves what is specific to each trace.

Typical uses are **receiver functions** (the radial component deconvolved by the vertical, leaving the P-to-S conversions under the station) and **empirical Green's functions** (a large event deconvolved by a small co-located one, leaving the source time function of the large event).

The spectral division is unstable where the reference spectrum is close to zero, so it is stabilised with a **water level** (Clayton & Wiggins, 1976): the reference power spectrum is never allowed below a fraction of its maximum.

| Parameter | Description |
|-----------|-------------|
| `reference_idx` | Index of the reference trace (int). |
| `water_level` | Fraction of the maximum reference power used as floor (float, default 0.001). Typical 0.001–0.1: larger = more stable, but lower resolution. |
| `normalize` | `True`: scale so that the reference deconvolved by itself peaks at 1. |
| `trim` | `True`: cut all traces to the common time window first. |
| `gaussian_filter` | (optional) width *a* of a Gaussian low-pass $G(f) = e^{-(2\pi f)^2 / 4a^2}$ applied to the result, which removes the high-frequency noise amplified by the division. *a* = 2.5 keeps up to about 1 Hz, as in receiver functions. |

```yaml
Analysis:
  process_1:
    name: 'deconv'
    reference_idx: 0
    water_level: 0.0001
    normalize: True
    trim: True
    gaussian_filter: 2.5
```

---

### Cross Spectrum

Compares every trace with the reference trace **in the frequency domain**. *Stream unchanged*: the results are saved in `output_path`.

- **Cross-spectral density**: the spectrum of the cross-correlation. Its phase gives the delay between the traces at each frequency.
- **Coherence**: between 0 and 1, how linearly related the two traces are at each frequency. It shows in which band two stations record the same signal.
- **Transfer function**: the frequency response that turns the reference into the other trace (e.g. site amplification relative to a rock station).

| Parameter | Description |
|-----------|-------------|
| `method` | `welch` (average of overlapping windows) or `multitaper` (several orthogonal tapers, lower variance). |
| `win` | Window length in seconds (`welch`). |
| `overlap_percent` | Overlap between windows, in percent. |
| `reference_idx` | Index of the reference trace (int). |
| `output_path` | Folder for the results. |
| optional | `nw` (multitaper time-bandwidth, default 4.0), `linf` / `lsup` (band limits in Hz), `trim` (default `True`), `compress` (default `True`). |

```yaml
Analysis:
  process_1:
    name: 'cross_spectrum'
    method: 'welch'
    win: 20.0
    overlap_percent: 50.0
    reference_idx: 0
    output_path: './cross_spectrum'
```

---

## D. Array analysis

### Beamforming

Estimates the **back-azimuth** and **apparent slowness** of the waves crossing an array, by testing a grid of slowness vectors, shifting and stacking the traces for each one, and keeping the one with the maximum coherent energy. *Stream unchanged*: the results are saved in `output_folder`. It needs an inventory with the coordinates of every station (`-i`).

Please review [Beamforming](https://projectisp.github.io/surfquaketutorial.github.io/beamforming_documentation/){: target="_blank" rel="noopener noreferrer" } for the description of every parameter.

```yaml
Analysis:
  process_1:
    name: 'beam'
    method: 'FK'
    timewindow: 3.0
    overlap: 0.05
    fmin: 1.0
    fmax: 8.0
    smax: 0.3
    slow_grid: 0.05
    output_folder: './beamforming'
```

---

## E. Arithmetic Operation (algebra)

Computes a new trace from a mathematical expression of the traces of the stream. Some examples of its use in signal processing:

- **Horizontal amplitude**: `sqrt(tr1**2 + tr2**2)` combines N and E into the total horizontal motion.
- **H/V ratios**, differences between co-located sensors (to check their response), sums and differences of components.
- **Energy and power**: `tr1**2`.
- **Instantaneous phase** of an analytic signal: `arctan2(tr2, tr1)`.

```yaml
Analysis:
  process_1:
    name: 'algebra'
    expression: "tr1 + tr2 - tr3"
    trim: True
    fill_gaps: True
```

| Option | Type | Description |
|----------|------|-------------|
| `expression` | `str` | The expression. `tr1` is the first trace of the stream, `tr2` the second, ... |
| `trim` | `bool` | Cut all traces to the common time window `[max(starttimes), min(endtimes)]` before evaluating the expression. Raises an exception if there is no overlap. |
| `fill_gaps` | `bool` | Merge the stream and fill internal gaps or overlaps using linear interpolation. |

All traces are resampled to the highest sampling rate of the stream before evaluating the expression. The output channels are named `ALG`, `ALG1`, ...

- Basic Functions

| Category | Operators | Example |
|-----------|-----------|---------|
| Arithmetic | `+`, `-`, `*`, `/`, `%` | `tr1 + tr2 * 2` |
| Power | `**` | `tr1 ** 2` |
| Floor Division | `//` | `tr1 // 2` |
| Unary | `+`, `-` | `-tr1` |

| Function | Description |
|-----------|-------------|
| `abs(x)` | Absolute value |
| `sqrt(x)` | Square root |
| `exp(x)` | Exponential |
| `log(x)` | Natural logarithm |
| `log2(x)` | Base-2 logarithm |
| `log10(x)` | Base-10 logarithm |
| `power(x, n)` | x to the power n |
| `mod(x, n)` | Remainder of x / n |

- Trigonometric Functions

| Function |
|----------|
| `sin`, `cos`, `tan`, `arcsin`, `arccos`, `arctan`, `arctan2` |
| `sinh`, `cosh`, `tanh` |

- Rounding Functions

| Function | Description |
|-----------|-------------|
| `ceil(x)` | Round up |
| `floor(x)` | Round down |
| `round(x)` | Round to nearest value |

- Signal Processing Helpers

| Function | Description |
|-----------|-------------|
| `sign(x)` | Sign of the signal |
| `diff(x)` | First difference (the output is one sample shorter) |
| `cumsum(x)` | Cumulative sum |
| `gradient(x)` | Numerical gradient |

- Complex Number Helpers

| Function | Description |
|-----------|-------------|
| `real(x)` | Real component |
| `imag(x)` | Imaginary component |

- Constants

| Constant | Value |
|-----------|-------|
| `pi` | π |
| `e` | Euler's number |

- Usage examples

| Goal | Expression |
|------|------------|
| Simple sum of two traces | `tr1 + tr2 * 1e-3` |
| Horizontal amplitude | `sqrt(tr1**2 + tr2**2)` |
| Complex math pipeline | `exp(tr1) + sin(tr2) * sqrt(abs(tr3))` |
| Multi-output (sum and difference) | `tr1 + tr2, tr1 - tr2` |
| Negate a trace | `-tr1` |
| Cumulative integral proxy | `cumsum(tr1)` |
| Numerical derivative | `gradient(tr1)` |
| Signal power | `tr1 ** 2` |
| Normalized ratio | `tr1 / (abs(tr2) + 1e-12)` |
| Phase extraction | `arctan2(tr2, tr1)` |

---

## F. Characteristic functions

A **characteristic function** (CF) transforms a waveform into a curve that rises when a seismic arrival starts. It is the core of automatic detection and picking. They are stream methods: n traces in, n CF traces out, with the same ids.

| Name | Idea |
|------|------|
| `snr` | STA/LTA: ratio of short-term to long-term average energy. The classic detector. |
| `envelope` | Instantaneous amplitude from the Hilbert transform. |
| `kurtosis` | Fourth-order statistics: high when the signal becomes impulsive. Sharp onsets. |
| `entropy` | Spectral entropy: drops when energy concentrates in a few frequencies. |
| `cwt_cf` | Wavelet-based CFs (energy, spectral ratio, entropy, centroid, kurtosis), optionally combined. |

Please review [Characteristic Functions](https://projectisp.github.io/surfquaketutorial.github.io/characteristic_functions_documentation/){: target="_blank" rel="noopener noreferrer" } and the template `surfquake make_config -c ch_functions`.

---

## G. Export

### Chop continuous data

Cuts continuous data into **clock-aligned** files of fixed length (e.g. one MiniSEED file per hour starting at hh:00:00). It is the usual way to prepare continuous archives for ambient noise, PPSD or detection workflows. *Stream unchanged.*

| Parameter | Description |
|-----------|-------------|
| `chunk_length` | Length of each output file, in seconds (e.g. 3600 = 1 h). |
| `min_length` | Minimum amount of **real** recorded data a file must contain, in seconds. Files with less data are not written. |
| `overlap` | Overlap between consecutive files, in seconds (0 = contiguous). |
| `output_dir` | Folder for the MiniSEED files. |
| `max_interpolation_gap` | (optional, default 2) largest gap that is interpolated, in seconds. |

```yaml
Analysis:
  process_1:
    name: 'chop'
    chunk_length: 1800.0
    min_length: 1760.0
    overlap: 0.0
    output_dir: './chop_data'
```
