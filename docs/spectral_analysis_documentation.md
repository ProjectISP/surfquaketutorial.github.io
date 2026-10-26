# Processing Options

## Spectral Analysis Tools

SurfQuake provides three main tools for spectral analysis of seismic data: **spectrum**, **spectrogram**, and **CWT** (Continuous Wavelet Transform). These tools enable users to inspect frequency content in different forms—static, time-evolving, and high-resolution wavelet representations.

---

## Spectrum

The `spectrum` tool computes the frequency-domain representation of a single seismic trace using the **multitaper method** by default, which is robust against noise and spectral leakage.

### Parameters

- **`method`**: Spectral estimation method. Options:
  - `multitaper` (default)
  - `fft` or others if extended

### Config Example

```yaml
Analysis:
  process_1:
    name: 'spectrum'
    method: 'multitaper' # 'fft' for conventional spectrum with single taper
    output_path: /Users/surf_test/spectral
```

### Output

- Amplitude vs. Frequency plot
- Optional log-log or semi-log axes (`loglog`, `xlog`, `ylog`)
- Serialized result saved as `.sp`

---

## Spectrogram

The `spectrogram` method produces a time-frequency representation of a seismic trace using short-time Fourier Transform (STFT).

### Parameters

- **`win`** (float): Window length in seconds (e.g., 5.0)
- **`overlap_percent`** (float): Percent overlap between windows (e.g., 50.0)
- **`linf`, `lsup`**: Lower and upper frequency limits for plotting
- **`method`**: Spectral estimation method. Options:
  - `multitaper` (default)
  - `fft`

### Config Example

```yaml
Analysis:
  process_1:
    name: 'spectrogram'
    method: 'multitaper'
    win: 5.0
    overlap_percent: 50.0
    output_path: /Users/surf_test/spectral
```

### Output

- Power over time and frequency in decibels (dB)
- Plotted as a color-coded matrix
- Serialized result saved as `.spec`

---

## Continuous Wavelet Transform (CWT)

CWT provides high-resolution time-frequency analysis using scalable wavelets. Ideal for transient signals and high-precision timing.

### Parameters

- **`wavelet`**: 
  - `"cm"`: Complex Morlet *(default and recommended for phase coherence)*
  - `"mh"`: Mexican Hat
  - `"pa"`: Paul wavelet
- **`param`**: Wavelet parameter (e.g., 6.0)
- **`fmin`, `fmax`**: Frequency range (optional)
- **`nf`**: Number of frequency bins

### Config Example

```yaml
Analysis:
  process_1:
    name: 'cwt'
    wavelet: 'cm'
    param: 6.0
    fmin: 0.5
    fmax: 20.0
    nf: 80
    output_path: /Users/surf_test/spectral
```

### Output

- Scalogram in decibel power
- Prediction masks showing reliable analysis range
- Serialized result saved as `.cwt`

---

## Plotting and CLI Usage

All spectral results can be visualized via the CLI:

```bash
# Plot a spectrum
surfquake specplot --file path/to/file.sp

# Plot a spectrogram
surfquake specplot --file path/to/file.spec 

# Plot a wavelet scalogram with optional clipping
surfquake specplot --file path/to/file.cwt --type cwt --clip -120

# Save output to file
surfquake specplot -f file.spec --save_path output.png
```

### CLI Options

- `--file`, `-f`: Path to serialized `.sp`, `.spec`, or `.cwt` file
- `--type`, `-t`: Type of spectral result to plot (`spectrum`, `spectrogram`, `cwt`)
- `--clip`, `-c`: Optional clipping in dB for spectrogram or CWT
- `--save_path`: Output path to save the figure

---

## Understanding and Accessing Spectral Result Objects in Python

SurfQuake does not store only an image of the spectral analysis. Its serialized output files contain Python result objects that preserve the numerical arrays and the original [ObsPy `Trace`](https://docs.obspy.org/packages/autogen/obspy.core.trace.Trace.html). This allows users to inspect spectral values, select particular frequencies or time windows, and perform further analysis without extracting data from a figure.

The examples below describe the classes implemented in `specrun.py`. The import path assumes that this module is installed as `surfquakecore.data_processing.specrun`; adjust it if your installation uses a different module layout.

### 1. Spectrum object (`.sp`)

A spectrum saved as `.sp` contains a `TraceSpectrumResult` instance. Its main attributes are:

| Attribute | Contents |
| --- | --- |
| `trace` | Original (or trimmed) ObsPy `Trace`, including waveform samples in `trace.data`. |
| `stats` | ObsPy trace metadata (`starttime`, `sampling_rate`, `network`, `station`, etc.). |
| `freq` | One-dimensional array of frequency samples in Hz. |
| `spectrum` | One-dimensional array of spectral amplitudes associated with `freq`. |
| `method` | Estimation method used, such as `multitaper` or `fft`. |

```python
import numpy as np
from surfquakecore.data_processing.specrun import TraceSpectrumResult

result = TraceSpectrumResult.from_pickle("path/to/trace.sp")

print(result.trace.id)
print(result.stats.starttime)
print("Method:", result.method)
print("Frequency bins:", result.freq.shape)

frequency_hz = np.asarray(result.freq)
amplitude = np.asarray(result.spectrum)

# Read the amplitude at the frequency bin nearest to 5 Hz.
idx = np.argmin(np.abs(frequency_hz - 5.0))
print(f"{frequency_hz[idx]:.3f} Hz: {amplitude[idx]:.6g}")
```

The two arrays are paired by index: `spectrum[i]` corresponds to `freq[i]`. The stored values are spectral amplitudes, not a time-frequency matrix.

### 2. Spectrogram object (`.spec`)

A spectrogram saved as `.spec` contains a `TraceSpectrogramResult` instance. It preserves the original power matrix rather than only the decibel values displayed in the figure.

| Attribute | Contents |
| --- | --- |
| `trace` | Original ObsPy `Trace`. |
| `stats` | Trace metadata. |
| `time` | One-dimensional array of spectrogram time coordinates, in seconds relative to trace start. |
| `freq` | One-dimensional array of spectrogram frequencies, in Hz. |
| `spectrogram` | Two-dimensional **power** array, indexed as `[frequency, time]`. |
| `num_steps` | Number of computed time steps returned by the spectral calculation. |

```python
import numpy as np
from surfquakecore.data_processing.specrun import TraceSpectrogramResult

result = TraceSpectrogramResult.from_pickle("path/to/trace.spec")

power = np.asarray(result.spectrogram)
frequencies = np.asarray(result.freq)
times = np.asarray(result.time)

print("Power matrix shape:", power.shape)
print("Frequency bins:", len(frequencies))
print("Time steps:", len(times))
assert power.shape == (len(frequencies), len(times))

# Power at the nearest available frequency and time bin.
fi = np.argmin(np.abs(frequencies - 2.0))
ti = np.argmin(np.abs(times - 60.0))
print(f"Power at {frequencies[fi]:.2f} Hz and {times[ti]:.1f} s:", power[fi, ti])

# Extract a frequency band (1–5 Hz) for subsequent calculations.
band = (frequencies >= 1.0) & (frequencies <= 5.0)
band_power = power[band, :]
print("Selected band matrix:", band_power.shape)

# Obtain absolute UTC times for the spectrogram columns.
absolute_times = [result.stats.starttime + float(t) for t in times]
print("First spectrogram time:", absolute_times[0])
```

**Important distinction:** `result.spectrogram` contains linear power values. The plotting method converts them to **relative decibels**, taking the maximum finite power as its reference:

\[
P_{\mathrm{rel}}(f,t) = 10\log_{10}\left(\frac{P(f,t)}{P_{\max}}\right).
\]

You can reproduce the conversion in Python:

```python
max_power = np.nanmax(power)
if not np.isfinite(max_power) or max_power <= 0:
    raise ValueError("Spectrogram has no positive finite maximum")

with np.errstate(divide="ignore", invalid="ignore"):
    relative_db = 10.0 * np.log10(power / max_power)

# Zero-power cells become -inf dB; filter them when needed.
finite_db = relative_db[np.isfinite(relative_db)]
print("Finite relative dB range:", finite_db.min(), finite_db.max())
```

The frequency and time coordinates describe the **computed bins**; they should not be confused with the waveform's individual sample times. Plotting options such as `clip`, `split`, and `smooth` change the presentation, not the stored power matrix.

### 3. Cross-spectral result object (`.coh`)

The same module also provides `TraceCrossSpectralResult` for analyses between a **reference** (input) trace and a **target** (output) trace. These objects can be saved as `.coh` files.

| Attribute or property | Contents |
| --- | --- |
| `ref_trace`, `trace` | Reference and target ObsPy traces. |
| `ref_stats`, `stats` | Metadata for the respective traces. |
| `freq` | Frequency samples in Hz. |
| `gxx`, `gyy` | Reference and target autospectral estimates. |
| `gxy` | Complex cross spectrum, defined as `E[conj(X) * Y]`. |
| `n_avg` | Averaging information used by the coherence-threshold calculation. |
| `method` | Spectral estimation method. |
| `coherence` | Derived magnitude-squared coherence (between 0 and 1). |
| `transfer` | Derived complex transfer function `gxy / gxx`. |
| `amplitude`, `phase_deg` | Derived transfer-function magnitude and phase in degrees. |

```python
from surfquakecore.data_processing.specrun import TraceCrossSpectralResult

result = TraceCrossSpectralResult.from_pickle("path/to/pair.coh")

print("Input:", result.ref_trace.id)
print("Output:", result.trace.id)

frequency_hz = result.freq
coherence = result.coherence
transfer_function = result.transfer
magnitude = result.amplitude
phase_degrees = result.phase_deg

# Identify frequencies with coherence of at least 0.8.
reliable = coherence >= 0.8
print("High-coherence frequencies:", frequency_hz[reliable])
```

These last four quantities are computed from the stored spectral estimates when accessed; they are Python properties, not separate arrays serialized independently. High coherence measures a strong linear frequency-domain relationship, but does not by itself establish causality.

### 4. CWT result object (`.cwt`)

A Continuous Wavelet Transform is represented by `TraceCWTResult`, defined in `surfquakecore.data_processing.cwtrun`. Unlike the spectrogram object, the wavelet result keeps its transform and associated coordinates in a **five-element tuple** called `cwt_data`.

| Attribute | Contents |
| --- | --- |
| `trace` | Original ObsPy `Trace`, including waveform samples. |
| `stats` | Trace metadata (`starttime`, `sampling_rate`, `channel`, etc.). |
| `cwt_data[0]` | `t`: relative time coordinates in seconds. |
| `cwt_data[1]` | `f`: wavelet frequency coordinates in Hz, logarithmically spaced. |
| `cwt_data[2]` | `scalogram`: wavelet-transform values, indexed as **`[frequency, time]`**. |
| `cwt_data[3]` | `pred`: left-hand edge-effect boundary, expressed as relative time coordinates. |
| `cwt_data[4]` | `pred_comp`: right-hand edge-effect boundary, expressed as relative time coordinates. |

Load a previously saved CWT result (the default serialization is gzip-compressed pickle):

```python
import numpy as np
from surfquakecore.data_processing.cwtrun import TraceCWTResult

result = TraceCWTResult.from_pickle("path/to/waveform.cwt")

print("Trace:", result.trace.id)
print("Sampling rate:", result.stats.sampling_rate, "Hz")

times, frequencies, coefficients, left_edge, right_edge = result.cwt_data

print("Time coordinates:", times.shape)
print("Frequency coordinates:", frequencies.shape)
print("CWT matrix:", coefficients.shape)
assert coefficients.shape == (len(frequencies), len(times))

# Inspect a coefficient at the nearest sampled frequency and time.
fi = np.argmin(np.abs(frequencies - 2.0))
ti = np.argmin(np.abs(times - 60.0))
print("CWT coefficient:", coefficients[fi, ti])
```

**Amplitude versus power:** The stored `scalogram` is the output of `ConvolveWaveletScipy.scalogram()`. The CWT plotting method computes **power** as `abs(scalogram)**2` and then displays power in decibels relative to the maximum:

```python
power = np.abs(coefficients) ** 2
max_power = np.nanmax(power)
if not np.isfinite(max_power) or max_power <= 0:
    raise ValueError("CWT power must have a positive finite maximum")

with np.errstate(divide="ignore", invalid="ignore"):
    relative_db = 10.0 * np.log10(power / max_power)

# Extract CWT power between 1 and 5 Hz.
selected = (frequencies >= 1.0) & (frequencies <= 5.0)
band_power = power[selected, :]
```

The `left_edge` and `right_edge` arrays delimit regions near the beginning and end of the trace where wavelet edge effects can influence the interpretation. They are **boundary coordinates**, not Boolean masks. The implementation computes them using an auxiliary linearly spaced frequency vector, whereas the stored `frequencies` vector is logarithmically spaced; avoid treating the boundary arrays as exact per-bin validity flags without considering this distinction.

```python
# Display the saved CWT using its built-in plotting method.
result.plot_cwt(split=1.0, clip=-80)

# Save a result object; the method chooses the .cwt filename.
result.to_pickle("./spectral_results", compress=True)
```

As with the other result objects, plotting parameters modify the figure presentation rather than replacing `result.cwt_data`. For the supported wavelet choices and computation parameters, see the **Continuous Wavelet Transform (CWT)** section above.

### 5. Inspecting an object and saving results

All four classes implement `to_pickle(folder_path, compress=True)` and `from_pickle(filepath, compress=True)`. **Compression is enabled by default**, so use the same `compress` setting when loading that was used when saving. File extensions (`.sp`, `.spec`, `.coh`, `.cwt`) identify the result type; they do not by themselves indicate whether gzip compression was enabled.

```python
# Explore attributes of an object already loaded as `result`.
print(type(result))
print(vars(result).keys())

# Save the full object to a directory (file name assigned automatically).
result.to_pickle("./spectral_results", compress=True)
```

---

### Scientific References

- Percival, D. B., & Walden, A. T. (1993). *Spectral Analysis for Physical Applications: Multitaper and Conventional Univariate Techniques*.
- Torrence, C., & Compo, G. P. (1998). A practical guide to wavelet analysis. *Bulletin of the American Meteorological Society*, 79(1), 61–78.

