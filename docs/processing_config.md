# Processing Individual Traces Options

## Detrending

Remove unwanted trends from the raw signal:

- `linear`: Remove a best-fit line
- `demean`: Subtract the mean
- `polynomial`: Fit and subtract a polynomial (e.g., degree 3)
- `spline`: Subtract a smooth spline fit

---

## Tapering

Apply a window to reduce spectral leakage near signal edges.

- **Parameter**: `max_percentage` (e.g., 0.05 for 5%)
- **Supported Windows**:
  `cosine`, `hann`, `hamming`, `blackman`, `kaiser`, `flattop`, `slepian`.

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

---

## Normalize

Rescale amplitude:

- `0`: Normalize by maximum
- Custom value: Divide signal by provided constant


```yaml
Analysis:
  process_1:
    name: 'normalize'
    norm: True
```

---

## Filtering

Apply filters to isolate specific frequency content.

- **Parameters**: `freqmin`, `freqmax`, `zerophase`, `corners` (poles)
- **Filter Types**:
  - `bandpass`, `lowpass`, `highpass`, `bandstop`
  - Parametric options: `cheby1`, `cheby2`, `elliptic`, `bessel`

ℹ️ Ensure `freqmin < freqmax` for band filters.

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
    name: 'normalize'
    norm: True
  process_4:
    name: 'filter'
    type: 'bandpass'
    fmin: 0.5
    fmax: 8.5
    corners: 4
    zerophase: True
```

---

## Denoising

### **Wiener Filter**
- **Parameters**: `time_window` (seconds), `noise_power`  
- Tip: If `noise_power=0`, it’s estimated from local variance.

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
    name: "wiener_filter"
    time_window: 1.0
    noise_power: 0
```

### **Wavelet Denoise**
- **Parameters**: `wavelet_family` (e.g., `sym8`), `threshold` (e.g., 0.05)

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

## Resampling

Adjust the sampling rate (e.g., for standardization):

- **Parameters**: `new_sampling_rate`, `pre_filter` (recommended: `True`)

```yaml
Analysis:
  process_1:
    name: 'resample'
    sampling_rate: 20
    pre_filter: True
```

---

## Quick Resampling

This strategy makes a very fast resampling specially designg to speed up the plot of seismograms. Warning this resampling not prevents from aliasing effects.

- **Parameters**: `factor`, `integers` (recommended: `True`)

```yaml
Analysis:
  process_1:
    name: 'raw'
    factor: 20
    integers: True
```

---

## Fill Gaps

- **Methods**: `interpolate`, `latest` (repeat last value)

```yaml
Analysis:
  process_1:
    name: 'fill_gaps'
    method: 'latest'
```

---

## Differentiate

Estimate the signal’s derivative:

- **Methods**: `gradient`, `spectral`

```yaml
Analysis:
  process_9:
    name: 'differentiate'
    method: 'spectral'
```

---

## Integrate

Reconstruct displacement or velocity:

- **Methods**: `cumtrapz`, `spline`, `spectral`

```yaml
Analysis:
  process_1:
    name: 'integrate'
    method: 'spectral'
```

---

## Time Shifting

Shift traces forward or backward in time.

- **Parameter**: List of time shifts in seconds

```yaml
Analysis:
  process_1:
    name: 'shift'
    time_shifts: [-5, 2] # list of shift seconds of the starttime. 
    phase: P #  if pick is vailable. 
    phase_theo: P #  if your trace header has event information.
```

---

## Remove Instrument Response

Convert raw counts to physical units (e.g., velocity or displacement):

⚠️ WARNING: Remember to includ your inventory: e.g., `surfquake quick -w "waveform_file" -i "inventory_file.xml"`

- **Parameters**: `pre_filt` (list), `water_level`, `units`, `inventory`

```yaml
  process_1:
    name: 'remove_response'
    water_level: 90
    units: 'VEL' # [ACC Wood Anderson]
    pre_filt: [0.005,0.008,45,48] # Hz, incremental frequencies defining the pre-filt
```

---

## Add Noise

**White Noise**:
- **Parameter**: `SNR_dB` amount of noise relative to maximum amplitude of the original trace.

**Colored Noise** (optional, not in base config):
- Use `exponent` for control: 0 (white), 1 (pink), 2 (brown), -1 (blue), etc.

```yaml
Analysis:
  process_1:
    name: 'add_noise'
    noise_type: 'white'
    SNR_dB: 1
```

---

## Spectral Whitening

Flattens signal spectrum while preserving phase:

- **Parameters**: `freq_width` (Hz), `taper_edge` (True/False)

```yaml
Analysis:
  process_11:
    name: 'whitening'
    freq_width: 0.02
    taper_edge: True
```

---

## Time Normalization

Standardize temporal energy distribution:

- **Methods**: `time-normalization`, `1bit`, `clipping` and  `clipping iteration`.
- **Parameters**: `norm_win`, `iterations`

```yaml
Analysis:
  process_1:
    name: 'time_normalization'
    method: 'time normalization'
    norm_win: 10
```

---

## Smoothing

Reduce high-frequency noise or jitter:

- **Parameters**: `time_window`, `fwhm`
- **Methods**: `mean`, `gaussian`, `tkeo`

```yaml
Analysis:
  process_1:
    name: 'smoothing'
    method: 'gaussian'
    time_window: 5
    FWHM: 0.05
```

---

## Spike Removal

Removes isolated outliers using hampel algorithm:

- **Parameters**: `window_size`, `sigma` (standard deviations from the avarage noise)

```yaml
Analysis:
  process_1:
    name: 'remove_spikes'
    window_size: 10
    sigma: 4
```

---

