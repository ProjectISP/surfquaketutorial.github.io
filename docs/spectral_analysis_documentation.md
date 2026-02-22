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

### Config Example

```yaml
Analysis:
  process_1:
    name: 'spectrogram'
    win: 5.0
    overlap_percent: 50.0
    linf: 0 # optional
    lsup: 20 # optional
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

- **`wavelet_type`**: 
  - `"cm"`: Complex Morlet *(default and recommended for phase coherence)*
  - `"mh"`: Mexican Hat
  - `"pa"`: Paul wavelet
- **`param`**: Wavelet parameter (e.g., 6.0)
- **`fmin`, `fmax`**: Frequency range
- **`nf`**: Number of frequency bins

### Config Example

```yaml
Analysis:
  process_1:
    name: 'cwt'
    wavelet_type: 'cm'
    param: 6.0
    fmin: 0.5 # optional
    fmax: 20.0 #optional
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

### Scientific References

- Percival, D. B., & Walden, A. T. (1993). *Spectral Analysis for Physical Applications: Multitaper and Conventional Univariate Techniques*.
- Torrence, C., & Compo, G. P. (1998). A practical guide to wavelet analysis. *Bulletin of the American Meteorological Society*, 79(1), 61–78.

