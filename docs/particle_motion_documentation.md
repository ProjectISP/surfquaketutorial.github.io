# Processing Options

## Particle Motion Analysis

The `pm` (particle motion) method analyzes the motion of a particle as recorded by a three-component seismic station (e.g., Z, N, E components). It provides geometrical parameters such as **azimuth**, **incidence angle**, **rectilinearity**, and **planarity**, which are critical for understanding wave polarization and propagation direction.

---

### Description

Particle motion analysis plots the trajectory of ground motion in 3D (or in 2D projections like Z-N, Z-E, and N-E planes). By examining these plots, users can determine:

- **Azimuth (°):** The horizontal direction of wave propagation.
- **Incidence (°):** The vertical angle of wave arrival relative to the surface.
- **Rectilinearity:** A measure of how straight (linear) the motion is, often associated with body waves.
- **Planarity:** A measure of how planar (flat) the motion is, often associated with surface waves.

The method automatically detects and maps three-component combinations (e.g., ZNE, Z12, ZYX) and processes them accordingly.

---

### Parameters

- **`output_path`** (optional): Directory where particle motion plots and results will be saved.  
  - If provided, a text file `pm.txt` will record azimuth, incidence, rectilinearity, and planarity for each analyzed trace.
  - A `.png` plot is generated for each station.

---

### Config Example

```yaml
Analysis:
  process_1:
    name: 'particle_motion'
    output_path: './pm_results'
```

---

### Outputs

For each valid three-component station:
- **Plots:**
  - Z vs. N projection
  - Z vs. E projection
  - N vs. E projection (with azimuth arrow)
  - Information box with computed parameters
- **Text file (pm.txt):** Appends results in CSV format:
  ```
  NET.STA.LOC.CHAN, YYYY-MM-DD HH:MM:SS, Azimuth, Incidence, Rectilinearity, Planarity
  ```

---

### Example Output Plot

The particle motion figure contains:
1. **Z vs. N** plane with incidence angle overlay.
2. **Z vs. E** plane with incidence angle overlay.
3. **N vs. E** plane with azimuth direction indicated by an arrow.
4. A summary panel showing:
   - Azimuth
   - Incidence
   - Rectilinearity
   - Planarity

---

### Scientific Background

Particle motion analysis is widely used in seismology for polarization studies, wavefield separation, and source characterization. For more details, see:

- Flinn, E. A. (1965). *Signal Analysis Using Rectilinearity and Direction of Particle Motion*. Proceedings of the IEEE, **53(12)**, 1874–1876.
- Vidale, J. E. (1986). *Complex polarization analysis of particle motion*. Bulletin of the Seismological Society of America, **76(5)**, 1393–1405.

---

### Notes

- Ensure that three-component data are present for each station (ZNE or equivalent).
- The traces are trimmed to a common time window before analysis.
- If no valid 3-component set is found, the process is skipped for that station.

