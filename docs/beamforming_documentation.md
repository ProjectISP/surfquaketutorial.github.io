## Description

The `beamforming` method performs array-based slowness and back-azimuth analysis of seismic waveforms. It is based on frequency-wavenumber (FK) analysis and supports additional methods like MUSIC, CAPON, and Multitaper Coherence. This process is useful for identifying wave arrival directions and apparent velocities in local, regional, or teleseismic arrays.

This method computes slowness and azimuth values over time windows by stacking energy across an array of stations. It is particularly effective in resolving wavefront parameters and isolating coherent seismic phases in noisy environments.

Internally, it calls the FK analysis provided by `surfquakecore.arrayanalysis`, and uses ObsPy’s data structures for waveforms and metadata.

> **Scientific Reference**:
> 
> Rost, S., & Thomas, C. (2002). Array seismology: Methods and applications. *Reviews of Geophysics*, 40(3).  


---

### Parameters

- **`timewindow`** (float): Duration of each sliding window for analysis (in seconds). Smaller windows improve temporal resolution; larger windows improve frequency resolution.
- **`overlap`** (float): Fractional overlap between consecutive time windows. Values close to 1 provide smoother results.
- **`fmin`, `fmax`** (float): Frequency band for filtering before beamforming. Filters out irrelevant noise and focuses on desired signal bands.
- **`smax`** (float): Maximum slowness value (s/km) to consider in the slowness grid.
- **`slow_grid`** (float): Resolution of the slowness grid (s/km). Smaller values yield higher-resolution slowness maps but are computationally more intensive.

---

### Config Example

```yaml
Analysis:
  process_1:
    name: 'beam'
    timewindow: 3.0
    overlap: 0.05
    fmin: 0.8
    fmax: 2.2
    smax: 0.3
    slow_grid: 0.05
    output_folder: "/documents/beam_results"
```

---

---

### Outputs

After running the beamforming step, the following attributes are available in the result object:

- **`relpower`**: Relative power for each time window.
- **`abspower`**: Absolute power estimates (if computed).
- **`AZ`**: Back-azimuth (degrees from North) per window.
- **`Slowness`**: Apparent slowness (s/km).
- **`T`**: Center times of each window.

You may visualize the results using the built-in `plot_beam()` method, which produces:
- Time series of relative power
- Time evolution of slowness and azimuth
- Optional FK/slowness maps on keypress

---

### Peak Detection

To extract beamforming-derived arrivals, use the `detect_beam_peaks()` method. It supports:

- **Slowness filtering** by phase type (regional/teleseismic)
- **Minimum power** thresholding
- **Azimuth constraints**
- **Minimum time separation** between detected peaks

This is useful for automated or semi-automated phase picking and event localization.

---

### Notes

- Requires an `ObsPy Stream` and corresponding `Inventory`.
- Trims traces to overlapping time intervals.
- Traces are filtered to the defined bandpass before analysis.
- Outputs can be saved and reloaded with `to_pickle()` and `from_pickle()` methods.


---

## Plotting and CLI Usage

Beamforming results can be visualized and analyzed using the `surfquake beamplot` command-line utility. This tool reads a serialized `.beam` file and provides rich options for interactive visualization and automated peak detection.

### Basic Usage

```bash
# Plot a saved FK beamforming result
surfquake beamplot --file ./output/2024.123.beam
```

### Peak Detection Examples

```bash
# Detect phase peaks with regional constraints
surfquake beamplot --file event.beam --find_solutions regional --write_path solutions.txt

# Apply custom slowness constraints (JSON format)
surfquake beamplot --file beam.beam --find_solutions '{"Pn": [0.05, 0.08], "Lg": [0.18, 0.30]}'

# Add azimuth filtering
surfquake beamplot -f beam.beam --find_solutions regional --baz_range 100 150

# Set minimum relative power threshold
surfquake beamplot -f beam.beam --find_solutions teleseismic --min_power 0.2
```

### CLI Options

- `--file`, `-f` (required): Path to the `.beam` file (gzip-pickled `TraceBeamResult`).
- `--save_path`: File path to save the plotted figure.
- `--find_solutions`: Define phase constraints for peak detection. Accepts `'regional'`, `'teleseismic'`, or a custom JSON dictionary.
- `--baz_range`: Filter solutions within a specific backazimuth range (in degrees).
- `--min_power`: Minimum relative power to accept a peak (default: `0.6`).
- `--write_path`: Output file to append detected peak information.

### Slowness map

- The figure includes time evolution of relative power, slowness, and backazimuth.
- Clicking or keypresses in the GUI will trigger slowness map popups for selected time points.
- This visualization is highly interactive and useful for manual phase inspection.

Point with the mouse to the power peaks and press a key number to select a method to plot the slowness map:

- **`Available methods`**: Beamforming algorithm to use:
    - 1 `FK`: Classical frequency-wavenumber method.
    - 2 `CAPON`: Adaptive beamforming with improved resolution.
    - 3 `MTP.COHERENCE`: Multitaper-based coherence stacking.
    - 4 `MUSIC`: High-resolution direction-of-arrival estimation method. Using one cluster.

---
