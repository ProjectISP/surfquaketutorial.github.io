# Time-Frequency Analysis

A seismogram is a function of time, but much of what it tells us is hidden in its **frequency content**: the corner frequency of an earthquake, the resonance of a site, the microseism peak, the tones of a ship's engine, the dispersion of surface waves. Spectral analysis moves the trace to the frequency domain to look at these properties.

SurfQuake offers six tools, each answering a different question:

| Tool | Question it answers | Output file |
|------|---------------------|-------------|
| **Spectrum** | Which frequencies are in the whole trace, and how strong are they? | `.sp` |
| **Spectrogram** | How does the frequency content change with time? | `.spec` |
| **CWT** (Continuous Wavelet Transform) | Same as the spectrogram, with a resolution that adapts to each frequency. Best for short transients. | `.cwt` |
| **Cepstrum** | Does the signal contain a pattern that repeats (echoes, pulse trains, harmonics)? With which period? | `.cep` |
| **Cepstrogram** | How does that repetition period change with time? | `.ceps` |
| **Cross-spectrum** | Are two traces related at each frequency (coherence)? How does one transform into the other (transfer function)? | `.coh` |

Each tool can be used in three ways:

1. **Interactively**, from the command prompt of the plot (`surfquake quick`, then `v`), to explore your data.
2. **In a config file**, to compute and save the results of many traces automatically.
3. **Plotting** the saved results with `surfquake specplot`, or reading them in **Python** for your own analysis.

A step-by-step example is the [Time-Frequency Tutorial](tutorials/tutorial2.md).

---

## Basic concepts

### Time and frequency resolution

A spectrum computed from a window of length $T$ seconds has a frequency resolution of about $\Delta f \approx 1/T$ Hz. Long windows separate close frequencies, short windows locate events precisely in time, but no window does both: this is the **time–frequency uncertainty principle**. Each time-frequency method is a different way of choosing this compromise:

- **Spectrogram**: one window length for all frequencies (you choose it with `win`).
- **CWT**: long windows for low frequencies and short windows for high frequencies, automatically.

### Tapering and spectral leakage

The FFT assumes that the window repeats periodically. If the signal does not start and end at zero, the jump at the edges spreads energy over all frequencies: this is **spectral leakage**, which hides weak peaks next to strong ones. SurfQuake always removes the mean and applies a taper:

- **FFT**: a single 5 % cosine taper on each side of the window.
- **Multitaper**: several orthogonal tapers (Slepian sequences), each one giving an independent estimate; their average has much less variance (less "noisy" spectra) and very little leakage. It is slower.

The data are also **zero-padded** to the next power of 2, which interpolates the spectrum (more frequency samples), but does not improve the real resolution, which depends only on the window length.

### Units: Fourier amplitude or spectral density

The spectrum is given in one of two physical scalings, the same for every method, so spectra computed with different methods can be compared directly:

| `scaling` | Units | Use it for |
|-----------|-------|------------|
| `fourier` (default) | trace unit × s (e.g. m/s · s) | **Transients** (earthquakes, explosions): $|X(f)|\,\Delta t$, the SAC convention. The value does not depend on how much quiet data surrounds the event, so it is the one to use for source spectra. |
| `asd` | trace unit / √Hz | **Continuous signals** (noise, microseism, tremor): amplitude spectral density, $\sqrt{\text{PSD}}$. The value does not depend on the record length, so it can be compared with noise models. |

The spectrograms store the **power spectral density** (PSD, unit²/Hz) of each window. In the figures, spectrograms and scalograms are shown in **dB relative to their maximum**:

$$P_{\mathrm{dB}}(f,t) = 10\log_{10}\left(\frac{P(f,t)}{P_{\max}}\right)$$

so 0 dB is the strongest point, and −60 dB is a million times weaker in power.

!!! tip
    Remove the instrument response first (`remove_response`) if you want the spectra in ground-motion units (m/s, m). Otherwise they are in counts.

---

## Spectrum

The **spectrum** shows how the energy of the whole trace (or of the selected window) is distributed in frequency. It is the tool to measure the corner frequency and the high-frequency decay of an earthquake, to identify the frequency band of a signal before filtering, or to find spectral lines (tones) in continuous data.

### Methods

| `method` | Description |
|----------|-------------|
| `fft` | Fast Fourier transform with a 5 % cosine taper. Fast, keeps the full resolution and also stores the **phase**. The standard choice for transients. |
| `multitaper` | Adaptive multitaper estimate. Smoother, lower variance, very little leakage. The best choice for noise and for weak peaks. |
| `burg`, `yule`, `covar`, `modcovar` | **Parametric** (autoregressive, AR) methods: they fit a model of `order` poles to the samples and draw the spectrum from the model. They give smooth spectra and sharp peaks from short records. `burg` is the recommended one; `yule` is the smoothest and least resolving. |
| `minvar` | Minimum variance (Capon) spectrum. |
| `music`, `ev` | **Pseudo-spectra**: they separate signal and noise subspaces to find the frequencies of pure tones with very high resolution. Their amplitudes have no physical meaning: look at where the peaks are, not at how high they are. |

The parametric methods need the optional package `spectrum` (`pip install spectrum`). The data are not tapered or zero-padded for them.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `method` | One of the methods above (default `fft`). |
| `scaling` | `fourier` (default) or `asd`, see [Units](#units-fourier-amplitude-or-spectral-density). Ignored by `music` and `ev`. |
| `order` | Parametric methods: model order (default 20). Each spectral peak needs about 2 poles, and they are spread from 0 Hz to Nyquist; to study low frequencies, **decimate** the trace first rather than raising the order a lot. Capped at 1/3 of the number of samples. |
| `nsig` | `music` and `ev` only (required): 2 per pure tone you expect in the data (1 tone → 2, 3 tones → 6). Must be smaller than `order`. |
| `output_path` | Folder for the results (created if missing). |

### Config Example

```yaml
Analysis:
  process_1:
    name: 'spectrum'
    method: 'fft'
    scaling: 'fourier'
    output_path: './spectral'
```

```yaml
Analysis:
  process_1:
    name: 'spectrum'
    method: 'multitaper'
    scaling: 'asd'
    output_path: './spectral'
```

```yaml
Analysis:
  process_1:
    name: 'spectrum'
    method: 'music'
    order: 30
    nsig: 4
    output_path: './spectral'
```

### Interactive

```bash
help spectrum
spectrum 0                                  # first trace, log-log axes, FFT
spectrum all loglog --mode multitaper       # all displayed traces
sp 2 xlog --mode burg --order 30            # 'sp' is an alias
sp 0 --mode music --order 20 --nsig 4
```

`axis_type`: `loglog` (default), `xlog`, `ylog` or `linlin`.

---

## Spectrogram

The **spectrogram** cuts the trace into overlapping windows and computes the spectrum of each one (short-time Fourier transform). The result is a picture of power as a function of time and frequency. It shows how a signal evolves: the arrival of P and S waves with their different frequency content, the dispersion of surface waves, the gliding tones of tremor, the passage of a ship, the daily cycle of cultural noise.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `win` | Window length in seconds (required). Longer → finer frequency resolution (about 1/`win` Hz) but coarser time resolution. It should contain several cycles of the lowest frequency of interest. |
| `overlap` | Overlap between consecutive windows, in percent (default 50). Larger values give a smoother image in time (more windows), not more resolution. |
| `method` | `multitaper` (default): smoother, less noisy, slower. `fft`: fast, 5 % cosine taper per window, also stores the phase. |
| `nw` | Multitaper only: time-bandwidth product. Larger = smoother spectrum, lower frequency resolution. If not set, an adaptive low-bias estimate is used (recommended: `nw` < 4). |
| `output_path` | Folder for the results. |

### Config Example

```yaml
Analysis:
  process_1:
    name: 'spectrogram'
    method: 'multitaper'
    win: 5.0
    overlap: 50
    output_path: './spectral'
```

### Interactive

```bash
help spectrogram
spectrogram 0                                       # first trace, defaults (5 s, 50 %)
spec 0 3.0 75                                       # 3 s windows, 75 % overlap
spec 0 5.0 50 -120                                  # clip the colour scale at -120 dB
spectrogram 0 5.0 50 --method fft --dates           # dates on the time axis
spectrogram 0 5.0 50 --method multitaper --nw 3.5
```

---

## Continuous Wavelet Transform (CWT)

The CWT compares the trace with a **wavelet**, a short oscillation, stretched and compressed to many frequencies. At high frequencies the wavelet is short, so the CWT is sharp in time; at low frequencies it is long, so the CWT is sharp in frequency. This adaptive resolution makes it the best tool for **transient** signals: onsets, short bursts, dispersed wave trains, and signals with energy over a wide band. The result is called a **scalogram**.

The frequencies are sampled logarithmically (`voices_per_octave` frequencies per octave, i.e. per doubling of frequency), which matches the adaptive resolution.

**Cone of influence**: near the beginning and the end of the trace, the wavelet partially falls outside the data and the scalogram is unreliable. This edge region is wider at low frequencies (longer wavelets) and is marked in the figure.

### Wavelets

| `wavelet` | Shape | Use it for |
|-----------|-------|------------|
| `cm` | **Complex Morlet**: a sine wave inside a Gaussian envelope. Good balance between time and frequency resolution; with `param: 6` its scales match Fourier frequencies. | The default and recommended choice. |
| `mh` | **Mexican Hat**: real wavelet, second derivative of a Gaussian. Very short, excellent time resolution, poor frequency resolution. | Locating sharp onsets. |
| `pa` | **Paul**: complex, asymmetric, short. Good time resolution. | Impulsive signals. |

### Parameters

| Parameter | Description |
|-----------|-------------|
| `wavelet` | `cm`, `mh` or `pa` (required). |
| `param` | Main wavelet parameter (required). For `cm` it is the number of cycles (6 = standard; larger = better frequency resolution, worse time resolution). |
| `fmin`, `fmax` | Frequency band analysed (Hz). `fmax` must be below Nyquist. Defaults: 4 cycles in the trace length, and Nyquist. |
| `voices_per_octave` | Frequency sampling: 8 = coarse, 16 = standard (default), 32 = fine. |
| `output_path` | Folder for the results (required). |

### Config Example

```yaml
Analysis:
  process_1:
    name: 'cwt'
    wavelet: 'cm'
    param: 6.0
    fmin: 0.5
    fmax: 20.0
    voices_per_octave: 16
    output_path: './spectral'
```

### Interactive

```bash
help cwt
cwt 0 cm 6                         # first trace, complex Morlet
cwt 2 mh 6 0.5 10                  # Mexican hat between 0.5 and 10 Hz
cwt 2 pa 6 0.5 10 -120 --dates     # clip at -120 dB, dates on the time axis
```

---

## Cepstrum and Cepstrogram

The **cepstrum** is the "spectrum of the log-spectrum". When a signal contains a pattern that **repeats** every $T$ seconds (an echo, a reverberation in a water or sediment layer, a train of pulses, the harmonics of a ship's propeller or of volcanic tremor), its spectrum shows regularly spaced ripples, every $1/T$ Hz. The cepstrum turns these ripples into a single **peak at quefrency $T$** (in seconds), which is much easier to detect and measure.

The **cepstrogram** computes the cepstrum in sliding windows, like the spectrogram, to follow how the repetition period changes with time (for example a ship approaching and moving away).

### Kinds

| `kind` | Definition | Use it for |
|--------|------------|------------|
| `real` | $c = \mathrm{IFFT}(\log\lvert X(f)\rvert)$. Keeps the sign. | Harmonic combs (propellers, tremor). |
| `power` | $c = \lvert\mathrm{IFFT}(\log\lvert X(f)\rvert^2)\rvert^2$. Always positive. | **Echoes and multipath**, where the reflection can make the peak negative in `real`. |
| `complex` | Uses the unwrapped phase too. Cepstrum only. | Separating a signal from its echoes (deconvolution). |

The low quefrencies (the first few samples) describe the smooth shape of the spectrum, not repetitions; they are hidden in the plots by default.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `kind` | `real`, `power` (or `complex` for the cepstrum). |
| `win` | Cepstrogram: window length in seconds. It must hold **several repetitions**: the longest detectable period is about `win`. |
| `overlap_percent` | Cepstrogram: overlap between windows, in percent. |
| `qmin`, `qmax` | Cepstrogram (optional): quefrency range kept, in seconds. |
| `output_path` | Folder for the results. |

### Config Example

```yaml
Analysis:
  process_1:
    name: 'cepstrum'
    kind: 'real'
    output_path: './spectral'
```

```yaml
Analysis:
  process_1:
    name: 'cepstrogram'
    kind: 'real'
    win: 20
    overlap_percent: 75.0
    output_path: './spectral'
```

---

## Cross-spectrum: coherence and transfer function

The cross-spectrum compares a **reference** trace with every other trace, frequency by frequency:

- **Coherence** (0–1): how linearly related the two traces are at each frequency. 1 = one trace is a filtered version of the other; 0 = unrelated. It shows the band where two sensors record the same signal, and a dashed line marks the significance threshold.
- **Transfer function** $H(f) = G_{xy}/G_{xx}$: the amplitude and phase of the filter that transforms the reference into the other trace. Use it to compare two sensors (huddle tests), to measure site amplification against a rock station, or to check the response of an instrument.

It needs averaging to be meaningful (coherence of a single window is always 1): `welch` averages overlapping windows, `multitaper` averages over tapers. It is a stream method; see [Processing Stream](processing_stream_config.md) for all its parameters.

```yaml
Analysis:
  process_1:
    name: 'cross_spectrum'
    reference_idx: 0
    method: 'welch'
    win: 20.0
    overlap_percent: 75.0
    output_path: './spectral'
```

Interactive:

```bash
help coh
coh --ref 0 --kind both --win 20
```

---

## Running the analysis

### Interactive commands

Open the plot with `surfquake quick`, select a time window with `u` (start) and `i` (end) if you want to analyse only a part of the traces, and press `v` to open the command prompt. The traces are referred to by their position in the current page (`0`, `1`, ...) or `all`.

| Command | Description |
|---------|-------------|
| `spectrum` / `sp` | Amplitude spectrum |
| `spectrogram` / `spec` | Spectrogram |
| `cwt` | Continuous wavelet transform |
| `coh` / `tf` | Coherence and transfer function |
| `save_tf` | Save one of the above to disk |

These commands only compute and display: the traces are not modified. To keep a result, save it with `save_tf`:

```bash
help save_tf
save_tf spectrum all --folder_path ./spectral
save_tf spectrogram 0 --win 5.0 --overlap 50 --method fft --folder_path ./spectral
save_tf cwt 0 --wavelet cm --param 6 --fmin 0.5 --fmax 10 --folder_path ./spectral
save_tf coh all --ref 0 --folder_path ./spectral
```

If a time window is selected (`u`, `i`), only that window is saved.

### Config file (automatic)

```bash
surfquake make_config -c spectral_config -o ./configs
surfquake quick -w "./data/*" -c ./configs/spectral_config.yaml -a
```

The template contains commented examples of every tool. The spectral steps can follow any pre-processing step (detrend, filter, remove response...) and they do not modify the traces: the next steps receive the same waveforms.

The results are saved in `output_path` as:

| Tool | File name |
|------|-----------|
| spectrum | `<trace_id>.D.<year>.<julday>.sp` |
| spectrogram | `<trace_id>.D.<year>.<julday>.spec` |
| cwt | `<trace_id>.D.<year>.<julday>.cwt` |
| cepstrum | `<trace_id>.D.<year>.<julday>.cep` |
| cepstrogram | `<trace_id>.D.<year>.<julday>.ceps` |
| cross-spectrum | `<ref_id>__<trace_id>.D.<year>.<julday>.coh` |

If a file already exists, a number is added (`_1`, `_2`...).

---

## Plotting and CLI Usage

All the saved results are plotted with `surfquake specplot`. The type of result is recognised from the file extension.

```bash
surfquake specplot -h

# Spectrum
surfquake specplot -f ./spectral/IU.HKT.00.BHZ.D.2024.123.sp -a xlog

# Spectrogram, split into low/high frequency panels, dates on the x axis
surfquake specplot -f ./spectral/IU.HKT.00.BHZ.D.2024.123.spec --clip -120 --split -d

# Scalogram, smoothed, saved to a folder
surfquake specplot -f ./spectral/WM.OBS01..SHZ.D.2015.260.cwt -d --split --smooth --save_path ./figures --cmap gnuplot2

# Cepstrogram, repetition periods between 0.05 and 2 s, with the dominant period tracked
surfquake specplot -f ./spectral/IU.HKT.00.BHZ.D.2024.123.ceps --qmin 0.05 --qmax 2 --track

# Coherence and transfer function
surfquake specplot -f ./spectral/IU.HKT.00.BHZ__IU.HKT.10.BHZ.D.2024.123.coh
```

To save all the figures of a folder, loop over the files in a [bash script](bash_script.md):

```bash
for file in ./spectral/*.spec; do
    surfquake specplot -f "$file" --split --smooth -d --save_path ./figures
done
```

### CLI Options

| Option | Applies to | Description |
|--------|------------|-------------|
| `-f`, `--file` | all | (required) Path to the `.sp`, `.spec`, `.cwt`, `.cep`, `.ceps` or `.coh` file. |
| `-a`, `--axis_type` | `.sp`, `.coh` | Axes: `loglog` (default), `xlog`, `ylog`, `linlin`. |
| `-c`, `--clip` | `.spec`, `.cwt`, `.ceps` | Minimum displayed power, in dB (e.g. `-120`). |
| `-v`, `--vmax_db` | `.spec`, `.cwt`, `.ceps` | Maximum displayed power, in dB (default 0). |
| `--split` | `.spec`, `.cwt` | Split the plot into low and high frequency panels, each with its own colour scale. Cutoff 1 Hz by default, or `--split 0.5`. Useful when the microseism dominates the colour scale. |
| `-s`, `--smooth` | `.spec`, `.cwt`, `.ceps` | Smooth the time-frequency image (slower, prettier). |
| `-d`, `--dates` | `.spec`, `.cwt`, `.ceps` | Dates on the x axis instead of seconds. |
| `--cmap` | `.spec`, `.cwt`, `.ceps` | Matplotlib colormap (default `rainbow`). |
| `--qmin`, `--qmax` | `.cep`, `.ceps` | Quefrency range shown, in seconds. |
| `--track` | `.ceps` | Overlay the dominant quefrency of every window. |
| `--save_path` | all | **Folder** where the figure is saved (named automatically); the figure is then not shown. |
| `--extension` | all | Image format: `png` (default), `pdf`, `svg`... |

The plotting options change the figure only, never the stored values.

---

## Accessing the results in Python

SurfQuake does not store only an image. Each file contains a Python object with the numerical arrays and the original [ObsPy `Trace`](https://docs.obspy.org/packages/autogen/obspy.core.trace.Trace.html){: target="_blank" rel="noopener noreferrer" }. You can load it, select frequencies or time windows, and continue your analysis without extracting data from a figure.

All the classes have `from_pickle(filepath)` to load a file, `to_pickle(folder_path)` to save it, and a plotting method. The files are gzip-compressed by default.

| File | Class | Import |
|------|-------|--------|
| `.sp` | `TraceSpectrumResult` | `from surfquakecore.spectral.specrun import TraceSpectrumResult` |
| `.spec` | `TraceSpectrogramResult` | `from surfquakecore.spectral.specrun import TraceSpectrogramResult` |
| `.coh` | `TraceCrossSpectralResult` | `from surfquakecore.spectral.specrun import TraceCrossSpectralResult` |
| `.cwt` | `TraceCWTResult` | `from surfquakecore.spectral.cwtrun import TraceCWTResult` |
| `.cep` | `TraceCepstrumResult` | `from surfquakecore.spectral.ceprun import TraceCepstrumResult` |
| `.ceps` | `TraceCepstrogramResult` | `from surfquakecore.spectral.ceprun import TraceCepstrogramResult` |

### 1. Spectrum (`.sp`)

| Attribute | Contents |
|-----------|----------|
| `trace`, `stats` | The analysed ObsPy `Trace` and its header. |
| `freq` | Frequencies (Hz). |
| `spectrum` | Spectral amplitudes, paired with `freq` by index, in the units given by `scaling`. |
| `phase` | Phase (radians) of each frequency, `fft` method only (otherwise `None`). |
| `method`, `scaling` | Method and scaling used. `order`, `nsig` for the parametric methods. |

```python
import numpy as np
from surfquakecore.spectral.specrun import TraceSpectrumResult

result = TraceSpectrumResult.from_pickle("./spectral/IU.HKT.00.BHZ.D.2024.123.sp")

print(result.trace.id, result.method, result.scaling)
freq = np.asarray(result.freq)
amp = np.asarray(result.spectrum)

# Amplitude at the frequency nearest to 5 Hz
i = np.argmin(np.abs(freq - 5.0))
print(f"{freq[i]:.3f} Hz: {amp[i]:.6g}")

# Frequency of the spectral peak between 1 and 20 Hz
band = (freq >= 1) & (freq <= 20)
print("Peak at", freq[band][np.argmax(amp[band])], "Hz")
```

### 2. Spectrogram (`.spec`)

| Attribute | Contents |
|-----------|----------|
| `trace`, `stats` | The analysed ObsPy `Trace` and its header. |
| `time` | Centre of each window, in seconds from the trace start. |
| `freq` | Frequencies (Hz). |
| `spectrogram` | 2D array of **power spectral density** (linear, unit²/Hz), indexed `[frequency, time]`. |
| `phase` | 2D phase array (radians), `fft` method only. |
| `num_steps` | Number of windows. |

```python
import numpy as np
from surfquakecore.spectral.specrun import TraceSpectrogramResult

result = TraceSpectrogramResult.from_pickle("./spectral/IU.HKT.00.BHZ.D.2024.123.spec")

power = np.asarray(result.spectrogram)
freq = np.asarray(result.freq)
time = np.asarray(result.time)
assert power.shape == (len(freq), len(time))

# Power at the nearest frequency and time
fi = np.argmin(np.abs(freq - 2.0))
ti = np.argmin(np.abs(time - 60.0))
print(f"PSD at {freq[fi]:.2f} Hz and {time[ti]:.1f} s:", power[fi, ti])

# Energy in the 1-5 Hz band as a function of time
band = (freq >= 1.0) & (freq <= 5.0)
band_energy = np.trapezoid(power[band, :], freq[band], axis=0)

# Absolute UTC times of the windows
utc_times = [result.stats.starttime + float(t) for t in time]

# Relative dB, as in the figure
with np.errstate(divide="ignore"):
    db = 10 * np.log10(power / np.nanmax(power))
```

### 3. Continuous Wavelet Transform (`.cwt`)

The result is stored in the tuple `cwt_data`:

| Element | Contents |
|---------|----------|
| `cwt_data[0]` | `t`: time in seconds from the trace start. |
| `cwt_data[1]` | `f`: frequencies (Hz), logarithmically spaced. |
| `cwt_data[2]` | Scalogram: **complex** wavelet coefficients, indexed `[frequency, time]`. The power is `abs(...)**2`. |
| `cwt_data[3]`, `cwt_data[4]` | Left and right limits of the cone of influence (times, in seconds). |

```python
import numpy as np
from surfquakecore.spectral.cwtrun import TraceCWTResult

result = TraceCWTResult.from_pickle("./spectral/WM.OBS01..SHZ.D.2015.260.cwt")
t, f, W, coi_left, coi_right = result.cwt_data

power = np.abs(W) ** 2
assert power.shape == (len(f), len(t))

# Power between 1 and 5 Hz
band = (f >= 1.0) & (f <= 5.0)
band_power = power[band, :]

# Instantaneous phase at 2 Hz (complex wavelets only)
fi = np.argmin(np.abs(f - 2.0))
phase = np.angle(W[fi, :])

# Plot again with other options
result.plot_cwt(split=1.0, clip=-80)
```

The cone-of-influence limits are times computed with a linear frequency axis, not boolean masks: use them as an approximate guide of the region affected by the edges.

### 4. Cepstrum (`.cep`) and Cepstrogram (`.ceps`)

| Class | Attributes |
|-------|------------|
| `TraceCepstrumResult` | `quefrency` (s), `cepstrum`, `kind`, and `find_peaks()` |
| `TraceCepstrogramResult` | `time` (window centres, s), `quefrency` (s), `cepstrogram` (2D, `[quefrency, time]`), `kind`, `win`, and `track_peak()` |

```python
from surfquakecore.spectral.ceprun import TraceCepstrumResult, TraceCepstrogramResult

cep = TraceCepstrumResult.from_pickle("./spectral/IU.HKT.00.BHZ.D.2024.123.cep")
for peak in cep.find_peaks(qmin=0.05, qmax=2.0, n_peaks=3):
    print(f"Repetition every {peak['quefrency']:.3f} s ({peak['frequency']:.2f} Hz), SNR {peak['snr']:.1f}")

ceps = TraceCepstrogramResult.from_pickle("./spectral/IU.HKT.00.BHZ.D.2024.123.ceps")
time, period, value = ceps.track_peak(qmin=0.05, qmax=2.0)   # dominant period in each window
```

`find_peaks()` returns the strongest repetition periods, as dictionaries with `quefrency` (s), `frequency` (= 1 / quefrency, Hz), `value` and `snr` (height above the median, in robust standard deviations).

### 5. Cross-spectrum (`.coh`)

| Attribute or property | Contents |
|-----------------------|----------|
| `ref_trace`, `trace` | Reference (input) and target (output) traces. |
| `freq` | Frequencies (Hz). |
| `gxx`, `gyy` | Auto-spectra of the reference and the target. |
| `gxy` | Complex cross-spectrum, $E[X^* Y]$. |
| `n_avg` | Number of averages, used for the significance threshold. |
| `coherence` | Magnitude-squared coherence (0–1). |
| `transfer` | Complex transfer function $G_{xy}/G_{xx}$. |
| `amplitude`, `phase_deg` | Amplitude and phase (degrees) of the transfer function. |
| `coherence_threshold(alpha)` | Coherence below which the result is not significant at level `alpha` (default 0.05). |

```python
from surfquakecore.spectral.specrun import TraceCrossSpectralResult

result = TraceCrossSpectralResult.from_pickle("./spectral/IU.HKT.00.BHZ__IU.HKT.10.BHZ.D.2024.123.coh")

threshold = result.coherence_threshold(alpha=0.05)
reliable = result.coherence > max(threshold, 0.8)
print("Reliable band:", result.freq[reliable].min(), "-", result.freq[reliable].max(), "Hz")
print("Mean amplification there:", result.amplitude[reliable].mean())
```

`coherence`, `transfer`, `amplitude` and `phase_deg` are computed from `gxx`, `gyy` and `gxy` when you access them. High coherence means a strong linear relationship, not necessarily causality.

### Computing in Python

The same classes compute the results directly from ObsPy traces:

```python
from obspy import read
from surfquakecore.spectral.specrun import TraceSpectrumResult, TraceSpectrogramResult

tr = read("./data/IU.HKT.00.BHZ.D.2024.123")[0]

sp = TraceSpectrumResult(tr)
sp.compute_spectrum(method="multitaper", scaling="asd")
sp.to_pickle("./spectral")

sg = TraceSpectrogramResult(tr)
sg.compute_spectrogram(win=5.0, overlap_percent=50, method="fft")
sg.plot_spectrogram(clip=-100, split=1.0)
```

---

## Scientific References

- Percival, D. B., & Walden, A. T. (1993). *Spectral Analysis for Physical Applications: Multitaper and Conventional Univariate Techniques*. Cambridge University Press.
- Thomson, D. J. (1982). Spectrum estimation and harmonic analysis. *Proceedings of the IEEE*, 70(9), 1055–1096.
- Torrence, C., & Compo, G. P. (1998). A practical guide to wavelet analysis. *Bulletin of the American Meteorological Society*, 79(1), 61–78.
- Bogert, B. P., Healy, M. J. R., & Tukey, J. W. (1963). The quefrency alanysis of time series for echoes. *Proceedings of the Symposium on Time Series Analysis*, 209–243.
- Bendat, J. S., & Piersol, A. G. (2010). *Random Data: Analysis and Measurement Procedures* (4th ed.). Wiley.
