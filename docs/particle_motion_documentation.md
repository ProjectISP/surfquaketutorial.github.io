# Particle Motion Analysis

## Description

As a seismic wave passes, each particle of the ground moves along a path in three dimensions. A three-component station (Z, N, E) records this path, and its **shape** tells us what kind of wave is passing and where it comes from:

- **P waves** push and pull the ground along the direction of propagation: the motion is **linear**, pointing towards (or away from) the source and tilted by the incidence angle.
- **S waves** move the ground perpendicular to the propagation: also linear, but perpendicular to the P direction.
- **Rayleigh waves** move the ground along an **ellipse** in the vertical plane that contains the source and the station.
- **Love waves** move the ground horizontally, perpendicular to the propagation.
- **Noise** usually has no preferred direction: a cloud of points.

**Particle motion** (or polarisation) analysis quantifies this shape. With a single station you can estimate the direction of a wave, its incidence angle and its type, which helps to identify phases, check the orientation of the sensors, and separate body waves from surface waves.

SurfQuake plots the particle motion in three planes (Z–N, Z–E and N–E) and computes the polarisation parameters of **Flinn (1965)**: **azimuth**, **incidence**, **rectilinearity** and **planarity**.

---

## How it works

The three components in a time window form a cloud of points in 3D. SurfQuake computes the **covariance matrix** of the three components (after removing the mean) and its eigenvalues $\lambda_1 \geq \lambda_2 \geq \lambda_3$ and eigenvectors. The eigenvectors are the axes of the ellipsoid that best fits the cloud, and the eigenvalues their squared lengths:

- The **first eigenvector** (largest eigenvalue) is the main direction of the motion.
- The ratio between the eigenvalues describes the **shape**: a needle (linear), a disc (planar, elliptical) or a sphere (no polarisation).

| Parameter | Definition | Meaning |
|-----------|------------|---------|
| **Azimuth (°)** | Horizontal direction of the first eigenvector, clockwise from North. | Direction of the main horizontal motion. Reported between 0° and 180° (see note below). |
| **Incidence (°)** | Angle between the first eigenvector and the vertical, 0°–90°. | 0° = vertical motion, 90° = horizontal motion. For a P wave it is the apparent incidence angle at the surface. |
| **Rectilinearity** | $1 - \sqrt{\lambda_2 / \lambda_1}$ | 1 = perfectly linear motion (body waves); 0 = no dominant direction. |
| **Planarity** | $1 - \dfrac{2\lambda_3}{\lambda_1 + \lambda_2}$ | 1 = the motion lies in a plane (e.g. elliptical Rayleigh motion, or linear motion); 0 = 3D cloud. |

!!! note "The 180° ambiguity"
    A linear motion back and forth along a line has no sign: the polarisation tells the direction of the **line**, not which way the wave is going. For this reason the azimuth is given between 0° and 180°, and the back-azimuth of the source is either the azimuth or the azimuth + 180°. For a P wave, the first-motion polarity solves it: a compressional first motion (Z up) moves the ground **away** from the source.

### Interpreting the results

| Wave | Rectilinearity | Planarity | Incidence | Azimuth |
|------|----------------|-----------|-----------|---------|
| P | high (> 0.8) | high | small to moderate | along the source direction |
| S (SV, SH) | high | high | large (mostly horizontal) | perpendicular to the source direction (SH) |
| Rayleigh | low to moderate | high | — | elliptical motion in the vertical plane towards the source |
| Love | high | high | ~90° (horizontal) | perpendicular to the source direction |
| Noise | low | low | — | not meaningful |

---

## Requirements

- Three components per station, recognised automatically by the last letter of the channel: **Z-N-E**, **Z-1-2** or **Z-Y-X**. They are mapped to Z, N, E. Note that `1` and `2` are taken as North and East: if your sensor is not oriented to North, the azimuth is relative to its own axes.
- The traces are cut to a common time window before the analysis.
- Stations without a complete three-component set are skipped.

!!! tip "Getting good results"
    - **Filter** the traces in a narrow band where the phase is clear (e.g. 1–5 Hz for local P waves): noise at other frequencies spreads the cloud and lowers the rectilinearity.
    - **Use a short window around the phase**: a few periods of the dominant frequency, starting at the onset. A long window mixes P, coda and S, and the result describes none of them.
    - If the instrument response is different between components, remove it first (`remove_response`).

---

## Usage

### Interactive

In the plot of `surfquake quick` (or `processing`), select a short window around the phase with the keys `u` (start) and `i` (end), open the command prompt with `v` and type:

```bash
help pm
pm
```

The particle motion of each station shown in the plot is computed in the selected window and displayed.

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
    name: 'filter'
    type: 'bandpass'
    fmin: 1.0
    fmax: 5.0
    corners: 4
    zerophase: True
  process_4:
    name: 'cut_stream'
    method: 'reference'
    t_before: 0.5
    t_after: 2.0
  process_5:
    name: 'particle_motion'
    output_path: './pm_results'
```

In this example, the traces are filtered and cut around the last reference time before the analysis. A reference time (key `w` on the plot) is added to **all** the traces, so the three components are cut to the same window. In automatic mode, read files that keep the reference in their header, i.e. HDF5 files saved after marking it (see [Trace Header](trace_header_documentation.md)).

ℹ️ With `method: 'phase'`, each trace is cut around its **own** pick, and traces without that pick are skipped. Use it only if the three components of each station are picked; otherwise the horizontal components are dropped.

The particle motion is a stream method that writes its results to disk: the traces passed to the next steps are unchanged.

### Parameters

- **`output_path`**: folder where the plots and the results are saved. Set it always.

---

## Outputs

For each valid three-component station:

- **Figure** `<trace_id>.D.<year>.<julday>.png` in `output_path`, with four panels:
    1. **Z vs. N** plane, with the incidence angle.
    2. **Z vs. E** plane, with the incidence angle.
    3. **N vs. E** plane (map view), with an arrow along the azimuth.
    4. A summary with azimuth, incidence, rectilinearity and planarity.
- **Text file** `pm.txt` in `output_path`. One line per station is appended:

  ```
  NET.STA.LOC.CHAN, YYYY-MM-DD HH:MM:SS, Azimuth°, Incidence°, Rectilinearity, Planarity
  ```

  The time is the start of the analysed window.

---

## Scientific Background

Particle motion analysis is widely used in seismology for polarisation studies, phase identification, wavefield separation, sensor orientation and source characterization. For more details, see:

- Flinn, E. A. (1965). Signal analysis using rectilinearity and direction of particle motion. *Proceedings of the IEEE*, 53(12), 1874–1876.
- Montalbetti, J. F., & Kanasewich, E. R. (1970). Enhancement of teleseismic body phases with a polarization filter. *Geophysical Journal International*, 21(2), 119–129.
- Vidale, J. E. (1986). Complex polarization analysis of particle motion. *Bulletin of the Seismological Society of America*, 76(5), 1393–1405.
- Jurkevics, A. (1988). Polarization analysis of three-component array data. *Bulletin of the Seismological Society of America*, 78(5), 1725–1743.
