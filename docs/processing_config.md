# Processing Individual Traces Options

Trace methods are applied to **each trace independently**: the result of one trace never depends on the others. They are the building blocks of any processing workflow: preparing the waveforms (gaps, trends, edges), changing their physical meaning (instrument response, derivative, integral), selecting the frequency content (filters, resampling) and cleaning them (denoising, spikes, normalisation).

Each method is a step under the `Analysis` section of your config file. Steps run in order: `process_1`, `process_2`, ...

```yaml
Analysis:
  process_1:
    name: 'rmean'
    method: 'linear'
  process_2:
    name: 'taper'
    method: 'cosine'
    max_percentage: 0.05
```

!!! tip "General rules"
    - Each `process_N` key must be **unique**. A repeated key silently replaces the earlier step.
    - Times are in seconds, frequencies in Hz.
    - Write decimals with a dot (`0.5`, `2.0`). Some options require a float: `fmin: 1` fails, `fmin: 1.0` works.
    - A full reference with every method is available with `surfquake make_config -c config_all`.

**A typical workflow** follows this order:

1. **Prepare** the data: `fill_gaps`, `cut`, `rmean`, `taper`.
2. **Physical units**: `remove_response`, `differentiate` / `integrate`.
3. **Frequency content**: `filter`, `resample`.
4. **Clean**: `remove_spikes`, `wavelet_denoise`, `wiener_smoothing`, `wiener_filter`, `smoothing`.
5. **Normalise**: `normalize`, `time_normalization`, `whitening`.

---

## Data preparation

### Fill Gaps

Real data often contain **gaps** (missing samples due to telemetry problems, instrument restarts...) or are split into several segments. Most signal processing operations (FFT, filters, correlations) need a continuous, evenly sampled series. `fill_gaps` merges the segments of the same channel and fills the missing samples.

| Parameter | Values | Description |
|-----------|--------|-------------|
| `method` | `latest` | Repeat the last valid value across the gap (a flat line). |
|  | `interpolate` | Linear interpolation between the samples on both sides of the gap. |

```yaml
Analysis:
  process_1:
    name: 'fill_gaps'
    method: 'latest'
```

!!! warning
    Filling a gap invents data. For short gaps (a few samples) it is harmless; for long gaps the filled segment can create artefacts after filtering (steps at the edges of the gap ring like an impulse). Avoid interpreting signals inside long filled gaps.

---

### Cut

Keep only a time window of the trace. Cutting reduces the computation time and focuses the analysis on the signal of interest (an event, a phase, a noise window).

| Parameter | Description |
|-----------|-------------|
| `method` | `absolute`: between two dates. `phase`: around a pick of the trace. `reference`: around the last reference time saved in the trace header. |
| `start`, `end` | (`absolute`) UTC date-times, e.g. `2018-10-26T05:20:03`. |
| `phase_name` | (`phase`) name of the picked phase, e.g. `'P'`. |
| `t_before`, `t_after` | (`phase`, `reference`) seconds before and after the pick or reference (floats). |
| `pad` | (optional, default `False`) if `True`, the window can go beyond the data; the extra samples are filled with `fill_value`. |
| `fill_value` | (optional, default `None`) value for the padded samples. With `None`, traces with gaps become NumPy masked arrays. |

```yaml
Analysis:
  process_1:
    name: 'cut'
    method: 'absolute'
    start: 2018-10-26T05:20:03
    end: 2018-10-26T06:25:03
```

```yaml
Analysis:
  process_1:
    name: 'cut'
    method: 'phase'
    phase_name: 'P'
    t_before: 10.5
    t_after: 120.0
```

!!! tip
    Picks and references are stored in the trace header, see [Trace Header](https://projectisp.github.io/surfquaketutorial.github.io/trace_header_documentation/){: target="_blank" rel="noopener noreferrer" }. To cut all traces of a stream to a common window, use `cut_stream` in [Processing Stream](processing_stream_config.md).

---

### Detrending (rmean)

Seismic records usually carry an **offset** (DC level) and slow **trends** (sensor drift, temperature, tilt). They are not part of the seismic signal, and they are harmful for later steps: an offset becomes a huge peak at 0 Hz in the spectrum, and a trend behaves like a very long-period signal that leaks energy into the band of interest after filtering.

| `method` | What it removes |
|----------|-----------------|
| `simple` | A straight line joining the first and the last sample. |
| `linear` | The best-fit (least-squares) straight line. The usual choice. |
| `constant` / `demean` | The mean value only. |
| `polynomial` | A best-fit polynomial of degree `order` (int). For curved drifts. |
| `spline` | A smoothing spline of degree `order` (1–5) with nodes every `dspline` samples (int). For complex, slow drifts. |

```yaml
Analysis:
  process_1:
    name: 'rmean'
    method: 'linear'
```

```yaml
Analysis:
  process_1:
    name: 'rmean'
    method: 'polynomial'
    order: 3
```

---

### Tapering

The FFT assumes that the signal repeats periodically. If the trace does not start and end at zero, this repetition creates a **sudden jump** at the edges, which spreads energy over all frequencies (**spectral leakage**) and makes filters ring at the beginning and end of the trace. A taper is a window that smoothly brings the trace ends to zero.

| Parameter | Description |
|-----------|-------------|
| `method` | Window shape: `cosine`, `hann`, `hamming`, `blackman`, `kaiser`, `flattop`, `slepian`, `bartlett`, `triang`, ... (ObsPy window types). |
| `max_percentage` | Fraction of the trace tapered at **each** end, 0.0–0.5 (float). `0.05` = 5 %. |
| `max_length` | (optional) maximum taper length at each end, in seconds. |
| `side` | (optional, default `both`) `both`, `left` or `right`. |

```yaml
Analysis:
  process_1:
    name: 'rmean'
    method: 'linear'
  process_2:
    name: 'taper'
    method: 'cosine'
    max_percentage: 0.05
```

!!! tip
    Always detrend **before** tapering, and taper **before** filtering, resampling or any FFT-based method. A 5 % cosine taper is a good default. Keep in mind that the tapered ends are attenuated: do not place your signal of interest there.

---

## Physical units

### Remove Instrument Response

A seismometer does not record ground motion directly: it records **counts**, the output of the sensor and the digitizer. Their response depends on frequency (a broadband sensor, for example, is flat in velocity only between its corner periods). Removing the response **deconvolves** the instrument and converts the counts into ground displacement, velocity or acceleration, so that records of different stations can be compared.

Deconvolution means dividing by the instrument spectrum. Where the instrument is not sensitive (very low and very high frequencies) this division amplifies noise enormously, so two stabilisation tools are used:

| Parameter | Description |
|-----------|-------------|
| `units` | Output: `DISP` (m), `VEL` (m/s), `ACC` (m/s²) or `Wood Anderson` (simulates a Wood-Anderson seismograph, used for local magnitudes). |
| `pre_filt` | Four frequencies `[f1, f2, f3, f4]` (Hz) of a cosine band-pass applied in the frequency domain: flat between `f2` and `f3`, tapered to zero below `f1` and above `f4`. Choose `f4` below the Nyquist frequency. |
| `water_level` | Water level in dB below the maximum of the response. It sets a floor to the instrument spectrum, so the division never explodes where the response is very small. |

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
    name: 'remove_response'
    water_level: 90
    units: 'VEL'
    pre_filt: [0.005, 0.008, 45, 48]
```

⚠️ WARNING: Remember to include your inventory: e.g., `surfquake quick -w "waveform_file" -i "inventory_file.xml"`. If the response of a trace is not found, its data are removed and a message is printed.

---

### Differentiate

Computes the time derivative of the trace, for example to convert displacement into velocity, or velocity into acceleration. In the frequency domain, differentiation multiplies the spectrum by $i\omega$: it **amplifies high frequencies**.

| `method` | Description |
|----------|-------------|
| `spectral` | Multiplies the spectrum by $i 2\pi f$ (after demean, taper and zero-padding). Exact for band-limited signals. |
| `gradient` | Finite differences in the time domain (NumPy gradient). Simple and robust. |

```yaml
Analysis:
  process_1:
    name: 'differentiate'
    method: 'spectral'
```

!!! tip
    Since high-frequency noise is amplified, apply a low-pass or band-pass filter before or after differentiating.

---

### Integrate

Computes the time integral of the trace, for example to convert acceleration into velocity, or velocity into displacement. In the frequency domain, integration divides the spectrum by $i\omega$: it **amplifies low frequencies**, so any small offset or long-period noise grows into a drift.

| `method` | Description |
|----------|-------------|
| `spectral` | Divides the spectrum by $i 2\pi f$ and removes the 0 Hz component (after demean, taper and zero-padding). |
| `cumtrapz` | Cumulative trapezoidal rule in the time domain. |
| `spline` | Integral of a spline interpolation of the data. |

```yaml
Analysis:
  process_1:
    name: 'integrate'
    method: 'spectral'
```

!!! tip
    Detrend and high-pass filter before integrating, otherwise the result drifts.

---

## Frequency content

### Filtering

A filter keeps the frequencies where your signal lives and attenuates the others: microseismic noise (0.1–0.5 Hz), cultural noise (> 1 Hz), long-period drifts... It is the most used tool in seismology.

| Parameter | Description |
|-----------|-------------|
| `type` | `bandpass`, `bandstop`, `lowpass`, `highpass` (Butterworth), or the band filters `cheby1`, `cheby2`, `elliptic`, `bessel`. |
| `fmin`, `fmax` | Corner frequencies in Hz (floats). `highpass` uses `fmin`, `lowpass` uses `fmax`. |
| `corners` | Filter order (number of poles). Higher = sharper cut, but more ringing. 2–4 is usual. |
| `zerophase` | `True`: the filter is applied forwards and backwards, so there is **no phase shift** and the onsets do not move (recommended for picking). The effective order is doubled. `False`: causal filter, the waveform is delayed and distorted, but nothing appears before the onset. |

About the filter families:

- **Butterworth** (`bandpass`, `lowpass`, ...): maximally flat pass-band, smooth roll-off. The standard choice.
- **Chebyshev I** (`cheby1`): sharper roll-off at the cost of ripple in the pass-band.
- **Chebyshev II** (`cheby2`): flat pass-band, ripple in the stop-band, sharp roll-off.
- **Elliptic** (`elliptic`): the sharpest roll-off, ripple in both bands.
- **Bessel** (`bessel`): the smoothest roll-off, but it preserves the waveform shape best (almost linear phase).

ℹ️ Ensure `fmin < fmax` for band filters, and `fmax` below the Nyquist frequency (half the sampling rate). The traces are detrended and tapered (5 %) internally before filtering.

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
    fmax: 8.5
    corners: 4
    zerophase: True
```

---

### Resampling

Changes the sampling rate of the trace, for example to bring all stations to the same rate, or to reduce the size of long datasets.

When **downsampling**, frequencies above the new Nyquist frequency (half the new sampling rate) fold back into the band and contaminate the signal: this is **aliasing**. To avoid it, the trace must be low-pass filtered before reducing the rate.

| Parameter | Description |
|-----------|-------------|
| `sampling_rate` | New sampling rate in Hz. |
| `pre_filter` | `True` (recommended): anti-alias low-pass filter at 90 % of the new Nyquist before each downsampling step. Large factors are done in several steps (at most ×10 each). |

```yaml
Analysis:
  process_1:
    name: 'resample'
    sampling_rate: 20
    pre_filter: True
```

---

### Quick Resampling (raw)

A very fast decimation: it keeps one sample out of every `factor`. It is designed to **speed up the plot** of long seismograms, not for analysis.

⚠️ No anti-alias filter is applied: high frequencies fold into the new band. Do not use the output for spectral analysis or picking.

| Parameter | Description |
|-----------|-------------|
| `factor` | Decimation factor (int). |
| `integers` | `True`: scale the data and convert them to int32, which reduces memory. |

```yaml
Analysis:
  process_1:
    name: 'raw'
    factor: 20
    integers: True
```

---

## Cleaning the signal

### Spike Removal

Spikes are isolated samples with huge amplitude (digitizer glitches, telemetry errors). They are very harmful: a single spike is broadband, so after filtering it becomes a wavelet that looks like a real arrival.

`remove_spikes` uses the **Hampel filter**: for each sample, it computes the median and the MAD (median absolute deviation) of a window centred on it. Since median and MAD are robust statistics, they are not affected by the spike itself. If the sample is further than `sigma` robust standard deviations (1.4826 × MAD) from the median, it is replaced by the median.

| Parameter | Description |
|-----------|-------------|
| `window_size` | Total length of the moving window, in seconds. It must be clearly longer than the spikes, but short compared to the signals you want to keep. |
| `sigma` | Threshold in robust standard deviations. 3–4 is usual; lower values remove more samples (and may clip real signals). |

```yaml
Analysis:
  process_1:
    name: 'remove_spikes'
    window_size: 10
    sigma: 4
```

---

### Wavelet Denoise

Decomposes the trace with the **Discrete Wavelet Transform** (DWT) into an approximation (the smooth trend) and several levels of details (one per octave of frequency). A coherent signal is concentrated in a few large coefficients, while random noise is spread over many small ones. Shrinking the small coefficients (soft thresholding) and reconstructing the trace removes much of the noise while keeping the sharp features of the signal (onsets), which a simple low-pass filter would smooth.

| Parameter | Description |
|-----------|-------------|
| `dwt` | Wavelet family: `db2`–`db20` (Daubechies), `sym2`–`sym20` (Symlets), `coif2`–`coif16` (Coiflets), `bior...` (biorthogonal). `sym8` is a good default. |
| `threshold` | Fraction (0–1) of the largest coefficient of each level below which the coefficients are shrunk. Larger = stronger denoising, but also more signal removed. |

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
    fmax: 8.5
    corners: 4
    zerophase: True
  process_4:
    name: 'wavelet_denoise'
    dwt: 'sym8'
    threshold: 0.1
```

---

### Wiener Smoothing

An **adaptive (local) Wiener filter** in the time domain (SciPy). For each sample, it estimates the local mean and variance in a short window. Where the local variance is much larger than the noise (a signal is present), the sample is kept almost unchanged; where the variance is close to the noise level, the sample is replaced by the local mean. The result is a smoothing that adapts to the signal: strong in quiet parts, weak during arrivals.

| Parameter | Description |
|-----------|-------------|
| `time_window` | Local window, in seconds. `0` = SciPy default (3 samples). |
| `noise_power` | Noise level as a multiple of the trace standard deviation. `0` = estimated from the data (mean of the local variances). |

```yaml
Analysis:
  process_1:
    name: 'wiener_smoothing'
    time_window: 1.0
    noise_power: 0
```

---

### Wiener Filter (SAC WIENER)

The equivalent of the SAC `WIENER` command: an **adaptive prediction-error filter**. A linear predictor with `ncoeff` coefficients is designed on a window of the trace (normally pre-event noise). The filter predicts what the trace "should" be from its past samples and outputs the **prediction error**. Everything that is predictable (stationary noise, narrow-band hum, microseisms) is removed, while the new, unpredictable information (an arrival) remains. This is also called **prewhitening**, because the output noise has a flat spectrum. It is very effective to sharpen onsets before picking.

| Parameter | Description |
|-----------|-------------|
| `window_start`, `window_end` | Design window: seconds from the trace start, or UTC date-times. Choose a window of noise before the event. |
| `ncoeff` | (optional, default 30, range 2–100) filter length including the leading 1 (prediction order = `ncoeff` − 1). More coefficients describe more complex noise spectra. |
| `mu` | (optional, float) adaptation step. If not given, the filter is fixed (designed once on the window). If given, the coefficients keep adapting along the trace, as in SAC. |
| `epsilon` | (optional, float) relative ridge added to the zero-lag autocorrelation to stabilise the design. If not given, the values 0, 1e-5, ..., 1e-2 are tried until the filter is stable (SAC default). |

```yaml
Analysis:
  process_1:
    name: 'wiener_filter'
    window_start: 0
    window_end: 20
```

---

### Smoothing

Reduces high-frequency jitter by averaging neighbouring samples. Smoothing is a low-pass filter in disguise, useful for envelopes and characteristic functions rather than for raw waveforms.

| `method` | Description |
|----------|-------------|
| `mean` | Moving average over `time_window` seconds. Simple, but its spectrum has side lobes. |
| `gaussian` | Convolution with a Gaussian of full width at half maximum `FWHM` seconds. Smooth response, no side lobes. |
| `adaptive` | Savitzky–Golay filter (polynomial of order 3) over `time_window` seconds: smooths while preserving peak heights and widths. |
| `tkeo` | Teager–Kaiser Energy Operator $x_n^2 - x_{n-1}x_{n+1}$: not a smoother but an **instantaneous energy** estimator, sensitive to both amplitude and frequency. It highlights sudden onsets. |

Both `time_window` and `FWHM` must be given, even if the selected method uses only one of them.

```yaml
Analysis:
  process_1:
    name: 'smoothing'
    method: 'gaussian'
    time_window: 5
    FWHM: 0.05
```

---

## Amplitude normalisation

### Normalize

Rescales the amplitude of the traces. It is useful to compare waveform shapes of stations with different gains or distances, for plotting record sections, and before machine learning.

| Parameter | Description |
|-----------|-------------|
| `norm` | `True`: divide by the absolute maximum (the trace peaks at ±1). A number (float): divide by that value. |
| `global_max` | (optional, default `False`) `True`: divide every trace by the absolute maximum of the **whole stream**, which keeps the relative amplitudes between traces. Use it with `norm: True`. Since it needs all traces, it runs together with the stream methods. |

```yaml
Analysis:
  process_1:
    name: 'normalize'
    norm: True
```

---

### Time Normalization

Used mainly in **ambient noise** processing. Earthquakes and other large transients dominate a cross-correlation of continuous data, while we want the contribution of the diffuse noise. Time normalisation reduces the weight of those high-amplitude windows.

| `method` | Description |
|----------|-------------|
| `1bit` | Keep only the sign of each sample (+1 / −1). The most aggressive: all amplitude information is lost, but it is very robust. |
| `clipping` | Clip the samples above 6 × the standard deviation. |
| `clipping iteration` | Repeatedly divide the samples above 6 × the standard deviation by 10, until none remains above it. |
| `time normalization` | Running-absolute-mean normalisation (Bensen et al., 2007): each sample is divided by the mean absolute amplitude in a window of `norm_win` seconds centred on it. |

| Parameter | Description |
|-----------|-------------|
| `norm_win` | Window of the running mean, in seconds (`time normalization` only). About half the longest period of interest. |

```yaml
Analysis:
  process_1:
    name: 'time_normalization'
    method: 'time normalization'
    norm_win: 10
```

> **Scientific Reference**:
>
> Bensen, G. D., et al. (2007). Processing seismic ambient noise data to obtain reliable broad-band surface wave dispersion measurements. *Geophysical Journal International*, 169(3), 1239–1260.

---

### Spectral Whitening

Flattens the **amplitude spectrum** inside a frequency band while keeping the **phase**. The spectrum is divided by a smoothed version of its own amplitude, so all frequencies of the band get the same weight. Outside the band the spectrum is strongly attenuated.

In ambient noise processing it broadens the band of the cross-correlations (otherwise dominated by the microseism peak) and removes monochromatic sources. Before cross-correlation or array analysis it also sharpens the correlation peaks.

| Parameter | Description |
|-----------|-------------|
| `fmin`, `fmax` | Whitening band, in Hz. |
| `freq_width` | Width (Hz) of the running average used to smooth the amplitude spectrum. Too small = noisy whitening; too large = incomplete whitening. |
| `taper_edge` | `True`: taper the band edges to reduce ringing in the time domain. |

```yaml
Analysis:
  process_1:
    name: 'whitening'
    fmin: 0.05
    fmax: 24.5
    freq_width: 0.02
    taper_edge: True
```

---

## Other trace operations

### Reverse

Reverses the order of the samples (time reversal): the last sample becomes the first. Useful to build anticausal operations, to check the symmetry of correlation functions, or to apply a causal filter backwards.

```yaml
Analysis:
  process_1:
    name: 'reverse'
```

---

### Flip

Multiplies the samples by −1, which inverts the **polarity** of the trace. Use it to correct sensors installed with reversed polarity, or to compare first-motion polarities.

```yaml
Analysis:
  process_1:
    name: 'flip'
```

---

### Add Noise

Adds synthetic random noise to the trace. It is used to test the robustness of a method (picking, location, magnitudes) at decreasing signal-to-noise ratios. The noise level is set **relative to the trace** (an SNR). To add noise with the level and spectrum of real seismic noise, use [Add Seismic Noise](#add-seismic-noise).

The noise is generated in the frequency domain with a power spectral density proportional to $1/f^{\alpha}$:

| `noise_type` | $\alpha$ | Character |
|--------------|----------|-----------|
| `white` | 0 | Flat spectrum, all frequencies equal. |
| `pink` | 1 | More energy at low frequencies. |
| `brown` | 2 | Dominated by low frequencies (random walk). |
| `blue` | −1 | More energy at high frequencies. |
| `violet` | −2 | Dominated by high frequencies. |

| Parameter | Description |
|-----------|-------------|
| `SNR_dB` | Signal-to-noise ratio of the result, in dB, as the ratio of the mean power of the trace to the mean power of the noise. 0 dB = same power; 20 dB = noise power 100 times smaller. |

```yaml
Analysis:
  process_1:
    name: 'add_noise'
    noise_type: 'white'
    SNR_dB: 1
```

---

### Add Seismic Noise

Adds **realistic seismic background noise**, with the spectrum and the absolute level of the noise recorded by real stations. While `add_noise` uses a mathematical spectrum and a level relative to the trace (an SNR), `add_seismic_noise` uses the **Peterson (1993) noise models**: the **New Low Noise Model** (NLNM), the quietest noise observed at the stations of a global network, and the **New High Noise Model** (NHNM), the noisiest. The real noise of almost every station lies between the two. They include the main features of the seismic noise: the **microseism peaks** between about 3 and 20 s and the increase of noise at long periods.

It is useful to test a method with the noise that a quiet or a noisy station would really record, for example to see down to which magnitude an event is still detected or picked, or to add noise to synthetic seismograms that have none.

How the noise is built (a Python port of R. Herrmann's `sacnoise`):

1. Gaussian white noise is generated, with the same length and sampling rate as the trace.
2. Its spectrum is shaped so that its power spectral density $P(f)$, in acceleration, follows the noise model: $|X(f)| = |W(f)|\sqrt{P(f)/(2\,\Delta t)}$, where $W(f)$ is the spectrum of the white noise and $\Delta t$ the sampling interval.
3. The noise, in acceleration, is converted to the **units of the trace** (see `units`).
4. The noise is added to the trace.

The level between the two models is set with `noise_level`. It is interpolated in decibels: $P_{dB}(f) = (1 - \text{noise\_level})\, \text{NLNM}(f) + \text{noise\_level}\, \text{NHNM}(f)$.

| Parameter | Description |
|-----------|-------------|
| `noise_level` | (optional, default `0.5`) `0` = NLNM (very quiet station), `0.5` = intermediate noise, `1` = NHNM (very noisy station). Values above 1 give noise above the NHNM, and below 0 noise under the NLNM. |
| `units` | (optional, default `counts`) units of the trace data. `counts`: raw data; the noise goes through the **instrument response**, as if recorded by the real sensor and digitizer. `ACC` (m/s²), `VEL` (m/s) or `DISP` (m): data already corrected for the instrument; the noise is converted by integration in the frequency domain, and no metadata is needed. |
| `seed` | (optional, default `None`) integer seed of the random generator. The same seed gives the same noise, so the test can be repeated. |

```yaml
Analysis:
  process_1:
    name: 'add_seismic_noise'
    noise_level: 0.5
    units: 'counts'
    seed: 12345
```

Example with data already in velocity: the response is removed first, then a quiet-station noise is added:

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
    name: 'remove_response'
    water_level: 90
    units: 'VEL'
    pre_filt: [0.005, 0.008, 45, 48]
  process_4:
    name: 'add_seismic_noise'
    noise_level: 0.2
    units: 'VEL'
```

⚠️ WARNING: With `units: 'counts'` the metadata is **required**: include your inventory, e.g. `surfquake quick -w "waveform_file" -i "inventory_file.xml"`. If there is no inventory, or the response of a trace is not found, a warning is printed and the noise is **not added** to that trace (the trace is kept unchanged).

!!! tip
    - Choose `units` according to the **current** state of the trace: `counts` before `remove_response`, and the output units of `remove_response` after it.
    - The models are defined for periods between 0.1 and 100000 s. At higher frequencies (above 10 Hz, when the sampling rate is above 20 Hz), the value of the model at 0.1 s is used.
    - The noise is added to the whole trace, also to the signal: compare the result with the original trace to see how the noise affects your method.

> **Scientific Reference**:
>
> Peterson, Jon R. "Observations and modeling of seismic background noise." U.S. Geological Survey Open-File Report 93-322 (1993): 1-95.

---

### Rename

Changes the network, station, location or channel codes of the traces. It is useful to harmonise data from different sources, or to change `1/2` components into `N/E`.

All maps are optional; write them as `old: new`.

| Parameter | Description |
|-----------|-------------|
| `networks`, `stations`, `locations` | Maps of whole codes. |
| `channels` | Map of whole channel codes (e.g. `HHZ` → `BHZ`). |
| `components` | Map of the last channel letter (e.g. `1` → `N`). Used only when the channel is not matched in `channels`. |

Special keys: `'*'` = any code not matched explicitly; `''` = empty code.

```yaml
Analysis:
  process_1:
    name: 'rename'
    networks:
      'XX': 'ES'
    locations:
      '': '00'
    components:
      '1': 'N'
      '2': 'E'
```

⚠️ Always quote the codes: unquoted `00` becomes `0`, and `NO` (Norway) becomes `False`.

---

## Time-frequency products

The methods `spectrum`, `spectrogram`, `cwt`, `cepstrum` and `cepstrogram` are also trace methods: they compute a time-frequency object of each trace and save it to disk, without modifying the trace. Please review [Time-Frequency Analysis](https://projectisp.github.io/surfquaketutorial.github.io/spectral_analysis_documentation/){: target="_blank" rel="noopener noreferrer" }.

---

## Time Shifting

Shifting traces in time is a stream method, see [Processing Stream](processing_stream_config.md).
