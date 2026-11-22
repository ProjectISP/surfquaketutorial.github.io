# Beamforming

## Description

A seismic **array** is a group of stations close enough to record the same wavefront, but far enough apart to see it arrive at different times. From those small delays, beamforming measures two properties of each wave that crosses the array:

- **Back-azimuth (BAz)**: the direction the wave comes from, in degrees clockwise from North (station → source).
- **Apparent slowness (s)**: how fast the wavefront sweeps across the surface, in s/km. Its inverse is the apparent velocity.

Together they identify the wave: where it comes from, and which phase it is (a steep teleseismic P crosses the array almost at once and has a small slowness, while a crustal Lg travels along the surface and has a large one). With them you can detect and locate events with a single array, separate phases arriving at the same time from different directions, and improve the signal-to-noise ratio by stacking.

SurfQuake computes the beamforming over sliding time windows, so you follow how power, slowness and back-azimuth evolve along the record, and lets you open the **slowness map** of any window with four different estimators: FK, Capon, Multitaper Coherence and MUSIC.

> **Scientific Reference**:
>
> Rost, S., & Thomas, C. (2002). Array seismology: Methods and applications. *Reviews of Geophysics*, 40(3).

A complete worked example is the [Beamforming Tutorial](tutorials/beamforming_tutorial.md) (North Korea nuclear test, ILAR array).

---

## How it works

Far from the source, the wavefront that crosses an array is approximately a **plane**. It is described by the horizontal **slowness vector** $\mathbf{s} = (s_x, s_y)$: its length is the slowness and its direction points along the propagation. A station at position $\mathbf{r}_i$ (relative to the array centre) records the wave with a delay

$$\tau_i = \mathbf{s} \cdot \mathbf{r}_i$$

Beamforming tests a **grid** of slowness vectors between $-s_{max}$ and $+s_{max}$ in both directions. For each one, the traces are shifted by their delays and stacked (in practice this is done in the frequency domain, with phase shifts). If the tested vector matches the real wave, the traces add up **in phase** and the beam power is maximal; otherwise they partially cancel. The grid point with the maximum power gives the slowness and the back-azimuth of the dominant wave:

$$s = \sqrt{s_x^2 + s_y^2}, \qquad \text{BAz} = \operatorname{atan2}(s_x, s_y) \;(\text{towards the source})$$

The power of all grid points forms the **slowness map**: a picture of where the energy comes from in a time window.

### Methods

| Method | Idea | When to use it |
|--------|------|----------------|
| **FK** (conventional, Bartlett) | Delay-and-sum in the frequency domain. Every direction is weighted the same. | Robust, the reference method. Its resolution is limited by the array aperture: a broad peak and side lobes given by the array response. |
| **CAPON** (minimum variance) | Adaptive weights: for each direction, it minimises the power coming from all other directions. | Sharper peaks, better separation of close arrivals. More sensitive to noise and to errors in station positions. |
| **MTP.COHERENCE** | Multitaper estimate of the coherence between stations instead of power. | Low-variance estimate, insensitive to amplitude differences between stations (site effects, gains). |
| **MUSIC** | Subspace method: separates the cross-spectral matrix into a signal subspace (1, 2 or 3 signals) and a noise subspace. | Highest resolution, can resolve several simultaneous waves. You must choose the number of signals; the values are a pseudo-spectrum, not a power. |

The time evolution (sliding windows) is computed with **FK** or **CAPON**. The slowness maps can be computed with any of the four methods.

### What the array can resolve

- **Aperture** (largest distance between stations): the larger the aperture compared with the wavelength, the sharper the slowness peak. Small arrays resolve high frequencies well but give broad peaks at low frequencies.
- **Station spacing**: if the stations are further apart than half a wavelength, the slowness map shows **aliases** (several peaks of equal power). The minimum spacing sets the highest usable frequency.
- **Geometry**: irregular, two-dimensional layouts give cleaner array responses than lines or regular grids.

This is why the frequency band (`fmin`, `fmax`) must be chosen for your array: a band where the wavelengths are comparable to the aperture, but longer than about twice the station spacing.

---

## Requirements

- An ObsPy `Stream` with one trace per station, usually the **vertical components**, with the same sampling rate.
- An **inventory** (StationXML or dataless) with the coordinates of every station (`-i`).
- The traces are cut to their common time window (padding with zeros where needed).

---

## Parameters

| Parameter | Description | How to choose it |
|-----------|-------------|------------------|
| `method` | `FK` or `CAPON`. | `FK` by default. |
| `timewindow` | Length of each sliding window (s). | It must contain at least a few cycles of `fmin` (e.g. 3 s at 1 Hz). Shorter windows follow fast changes but give noisier estimates. |
| `overlap` | **Step** between consecutive windows, as a fraction of `timewindow`. `0.05` moves the window by 5 % of its length (95 % overlap). `1.0` gives contiguous windows without overlap. | Small values give a dense, smooth time evolution but take longer. 0.05–0.2 is usual. |
| `fmin`, `fmax` | Frequency band of the analysis (Hz). Only the frequencies in this band enter the cross-spectral computation. | The band where your signal is above the noise, adapted to the array size (see above). |
| `smax` | Maximum slowness of the grid (s/km). The grid covers $-s_{max}$ to $+s_{max}$ in $s_x$ and $s_y$. | Larger than the slowest phase you want to see. 0.3–0.4 s/km covers body waves and Lg. |
| `slow_grid` | Spacing of the slowness grid (s/km). | Fine enough to separate the phases you are interested in: 0.01 s/km for teleseismic P, 0.02–0.05 s/km for regional phases. Computation time grows with $(2 s_{max} / \text{slow\_grid})^2$. |
| `output_folder` | Folder where the result is saved. | |

### Typical apparent slowness of seismic phases

Use these values to choose `smax` and `slow_grid`, and to interpret your results (they vary with distance and with the Earth structure under the array):

| Phase | Apparent velocity (km/s) | Slowness (s/km) |
|-------|--------------------------|-----------------|
| Teleseismic P (30°–90°) | 12–25 | 0.04–0.08 |
| PKP (core phases) | > 25 | < 0.04 |
| Pn | ~8 | ~0.12–0.13 |
| Pg | ~6–6.5 | ~0.15–0.17 |
| Sn | ~4.5 | ~0.21–0.23 |
| Lg | ~3.5 | ~0.27–0.30 |
| Rg (short-period Rayleigh) | ~3 | ~0.33 |

---

## Usage

### Config file (automatic)

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
    name: 'beam'
    method: 'FK'
    timewindow: 3.0
    overlap: 0.05
    fmin: 0.8
    fmax: 2.2
    smax: 0.3
    slow_grid: 0.05
    output_folder: "./beam_results"
```

```bash
surfquake quick -w "./data/*Z*" -i ./metadata/inventory.xml -c ./configs/beam_config.yaml -a
```

The beamforming is a stream method that writes its result to disk; the traces passed to the next steps are unchanged. To analyse only a time window, add a `cut_stream` step before `beam` (see [Processing Stream](processing_stream_config.md)).

The result is saved in `output_folder` as `<year>.<julday>.beam` (year and julian day of the first trace), a compressed file that contains the complete result: power, slowness, back-azimuth, the parameters, the traces and the inventory.

### Interactive

In the plot of `surfquake quick` (with `-i`), select a time window with the keys `u` (start) and `i` (end), open the command prompt with `v` and type:

```bash
help beam
beam --fmin 0.8 --fmax 2.2 --smax 0.3 --grid 0.01 --win 3 --overlap 0.05

help smap
smap --method CAPON --fmin 0.8 --fmax 2.2 --smax 0.3 --grid 0.01
```

- `beam`: FK time evolution over the selected window, with the same figure and keys as `beamplot`.
- `smap`: slowness map of the whole selected window. `--method`: `FK`, `CAPON`, `MTP.COHERENCE` or `MUSIC` (in capital letters).

---

## Outputs

The result is a `TraceBeamResult` object with one value per time window:

| Attribute | Description |
|-----------|-------------|
| `relpower` | Relative power (0–1): the coherence of the traces for the best slowness vector. Close to 1 = a coherent plane wave crossing the array; low values = incoherent noise. |
| `abspower` | Absolute power of the beam. It depends on the amplitude of the signal. |
| `AZ` | Back-azimuth (°, 0–360, clockwise from North). |
| `Slowness` | Apparent slowness (s/km). |
| `T` | Start time of each window, as Matplotlib date numbers. |

You can load and use a result in Python:

```python
from matplotlib.dates import num2date
from surfquakecore.arrayanalysis.beamrun import TraceBeamResult

beam = TraceBeamResult.from_pickle("./beam_results/2017.246.beam")

best = beam.relpower.argmax()
print(num2date(beam.T[best]), beam.AZ[best], beam.Slowness[best], beam.relpower[best])
```

---

## Plotting and CLI Usage

Beamforming results can be visualized and analyzed using the `surfquake beamplot` command-line utility. It reads a `.beam` file and provides interactive visualization and automated peak detection.

### Basic Usage

```bash
surfquake beamplot -h

# Plot a saved FK beamforming result
surfquake beamplot -f ./beam_results/2024.123.beam

# Save the figure instead of showing it
surfquake beamplot -f ./beam_results/2024.123.beam --save_path ./beam_results/2024.123.png
```

The figure shows three panels sharing the time axis: relative power, slowness and back-azimuth, all coloured by relative power. A wave crossing the array appears as a group of high-power points with stable slowness and back-azimuth; noise appears as scattered, low-power points.

### Slowness maps

The figure is interactive. Point with the mouse at a time (for example a power peak), press a number to choose the method, and then press `e` to plot the slowness map of the window starting at that time:

| Key | Method |
|-----|--------|
| `1` | FK (conventional) |
| `2` | Capon |
| `3` | Multitaper Coherence |
| `4` | MUSIC, 1 signal |
| `5` | MUSIC, 2 signals |
| `6` | MUSIC, 3 signals |
| `e` | Compute and plot the slowness map |

The map shows the power over the $(s_x, s_y)$ plane, with the slowness, back-azimuth and power of the maximum. The distance from the centre to the maximum is the slowness; its direction is the back-azimuth.

### Peak Detection

`--find_solutions` searches the time evolution for arrivals of given phases:

1. For each phase, only the windows whose slowness is inside its slowness range are kept.
2. The peaks of relative power are found among them.
3. Peaks below `--min_power`, or outside `--baz_range`, are rejected.
4. Peaks closer than 10 s to an already accepted peak are rejected, so the same arrival is not assigned to two phases.
5. The **strongest** remaining peak of each phase is reported.

This is useful for automated or semi-automated phase identification and event location.

```bash
# Detect phase peaks with the regional shortcut
surfquake beamplot -f event.beam --find_solutions regional --write_path solutions.txt

# Custom slowness ranges (JSON format)
surfquake beamplot -f event.beam --find_solutions '{"Pn": [0.11, 0.14], "Lg": [0.26, 0.31]}'

# Add back-azimuth filtering
surfquake beamplot -f event.beam --find_solutions regional --baz_range 100 150

# Set the minimum relative power
surfquake beamplot -f event.beam --find_solutions teleseismic --min_power 0.4
```

The shortcuts use these slowness ranges (s/km):

| `regional` | | `teleseismic` | |
|------------|---|---------------|---|
| Pn | 0.11–0.14 | P | 0.04–0.07 |
| Pg | 0.14–0.18 | PP | 0.06–0.08 |
| Sn | 0.20–0.24 | PKP | 0.05–0.07 |
| Lg | 0.26–0.31 | PcP | 0.05–0.07 |
| Rg | 0.31–0.40 | S | 0.07–0.12 |
| | | SS | 0.09–0.13 |
| | | SKS | 0.07–0.10 |
| | | ScS | 0.09–0.13 |
| | | Rayleigh | 0.20–0.40 |
| | | Love | 0.20–0.35 |

!!! tip
    The best ranges depend on your array and distance. Compare the shortcut ranges with the [typical apparent slowness](#typical-apparent-slowness-of-seismic-phases) of the phases, and use your own JSON ranges when needed.

### CLI Options

| Option | Description |
|--------|-------------|
| `--file`, `-f` | (required) Path to the `.beam` file. |
| `--save_path` | File path to save the figure (e.g. `beam.png`). The figure is then not shown. |
| `--find_solutions` | Phase constraints for peak detection: `regional`, `teleseismic`, or a JSON dictionary `'{"phase": [smin, smax], ...}'`. |
| `--baz_range` | Accept only solutions with back-azimuth between `MIN` and `MAX` (degrees). |
| `--min_power` | Minimum relative power to accept a peak (default: `0.6`). |
| `--write_path` | Text file where the detected peaks are appended, one line per phase: `phase, time, back-azimuth, slowness, power`. |

---

## Tips

- **Filter and choose the band first** by looking at the spectra of your traces: the beamforming only works where the signal is coherent across the array.
- **Check the array response**: a slowness map of pure noise, or of a single clear arrival, shows the side lobes of your array; do not interpret them as waves.
- **Relative power** is your quality measure: values above ~0.6 indicate a well-defined plane wave.
- **Back-azimuth deviations** of a few degrees from the theoretical value are normal (lateral heterogeneity under the array and along the path), and they are systematic for a given direction: they can be calibrated with known events.
- **Small slowness = steep incidence**: at very small slowness (core phases, near-vertical P) the back-azimuth becomes poorly constrained.
