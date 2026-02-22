# SurfQuake Plotting and Interactive Commands

SurfQuake provides a powerful plotting interface designed for seismological workflows. It supports trace visualization, interactive pick annotation, metadata overlays, and integration with processing outputs such as characteristic functions, beamforming results, or filtered waveforms.

This plotting interface is designed for both exploratory data analysis and high-quality figure export.

---

## Overview of Interactive Plotting

When a waveform stream is visualized (e.g., after running a processing pipeline or with the `surfquake quick` command), SurfQuake launches an interactive window showing a page of traces. This interface supports keyboard-driven interactions to facilitate manual picking and review.

Each subplot corresponds to a single seismic trace and includes time axes, amplitude plots, and optional overlays (picks, markers, etc.). Traces are paginated if their count exceeds the configured number per page.

### Core Features

- **Paging**: Use `n` and `b` to move to next and previous pages.
- **Picking**: Press `e` and click to pick a phase arrival. You'll be prompted for phase type and polarity.
- **Reference markers**: Press `w` to insert a green vertical reference line.
- **Pick management**:
  - `d`: Delete the last pick.
  - `c`: Clear all picks.
  - `m`: Remove the most recent reference.
  - `p`: Remove the last reference marker from all traces.
- **Command prompt**: Press `v` to open the SurfQuake command bar within the plotting session.
- **Exit keys**: Use `enter` or `esc` to exit the interactive session.

All interactions update the internal `Trace.stats` metadata. For instance, picks are stored in `Trace.stats.picks`, and reference markers in `Trace.stats.references`.

---

## Pick Metadata Storage

When a user places a pick using the `e` key, a dictionary is appended to the picking list `tr.stats.picks`, with the following structure:

```python
{
    "time": float,          # UTC timestamp
    "phase": str,           # Phase name (e.g., 'P', 'S', 'Pg', etc.)
    "amplitude": float,     # Interpolated amplitude at pick time
    "polarity": str         # 'U', 'D', or '?' (unclear)
}
```

Reference markers (green vertical lines placed via `w`) are stored as timestamps in `tr.stats.references`.

---

## Configuration Options

The plotting interface is governed by a YAML configuration passed via `--plot_config`, which supports:

```yaml
plotting:
  traces_per_fig: 3 # default 6
  sort_by: distance            # options: 'distance', 'backazimuth', or null
  vspace: 0.05                 # vertical space between subplots
  show_legend: true
  title_fontsize: 9
  plot_type: standard        # 'standard', 'record' for record section and overlay for all traces at the same plot 
  sharey: False
  show_arrivals: False
  pick_output_file: ./picks.csv # picks will be written here
  auto_load_pick_file: False   # if user wants that picking file is automatically loaded and plot
  show_info_picks: False # show pick info on screen interactively
  backend: TkAgg # TkAgg (maximum robustness, default), MacOSX (only for mac), Qt5Agg (fast, picking  & interactive commands raise problems)
```

---

## Output and Persistence

- All user picks and references are stored in memory and (optionally) written to disk.
- Picks can be logged to CSV using the `pick_output_file` config key.
- The waveform figures can be exported as `.png` files.
- Header metadata can be updated interactively and saved if exported in HDF5 format.


## Summary Table of Key Commands in picking mode.

| Key | Action                          |
|-----|---------------------------------|
| `1` | P-wave polarity ?               |
| `2` | P-wave polarity Up              |
| `3` | P-wave polarity Down            |
| `4` | S-wave polarity ?               |
| `5` | S-wave polarity Up              |
| `6` | S-wave polarity Down            |
| `e` | Add pick                        |
| `w` | Add reference line              |
| `d` | Delete last pick                |
| `c` | Clear all picks                 |
| `m` | Remove most recent reference    |
| `p` | Remove last reference from all  |
| `n` | Next page                       |
| `b` | Previous page                   |
| `v` | Command prompt                  |
| `enter` / `esc` | Exit plotting       |


---

## Command-Line Prompt (v Key)

SurfQuake provides an embedded terminal-style command prompt for advanced user interaction. Press `v` during plotting to access it.

This prompt allows you to apply real-time processing, visualization, trimming, and export actions without leaving the plot.

### Example Session

```
>> filter bandpass 0.1 1.0 --corners 4 --zerophase True
>> cut --phase P 5 15
>> beam --method FK --fmin 0.8 --fmax 2.0 --grid 0.05
>> write --folder_path ./processed_h5/
>> p
```

### Available Commands (Detailed)

- `filter <type> <fmin> <fmax> [options]`: Filter all displayed traces.
    - Example: `filter bandpass 0.5 8.0 --corners 4 --zerophase True`

- `sp <index>|all [axis_type]`: Plot spectrum (FFT) of a specific or all traces options of axis plot are loglog, xlog, ylog.
    - Example: `spectrum 0`, `spectrum all loglog`

- `spec <index> [win overlap]`: Plot multitaper spectrogram of a trace. win is the sliding window defined in seconds and overlap is the % of sliding windoe step overlap.
    - Example: `spec 0 4 75` # 4 seconds overalp of 75%

- `cwt <index> <wavelet> <param> [fmin fmax]`: Continuous wavelet transform visualization.
    - Example: `cwt 1 cm 6 0.1 10` options are wavelet: cm (complex morlet wavelet), mh (mehican hat) and pa (Paul Wavelet).

- `beam [options]`: Run beamforming in sliding windows (FK method).
    - Example: `beam --fmin 0.5 --fmax 2.5 --win 3 --grid 0.025`

- `smap [options]`: --fmin --fmax --smax --grid --method FK|MTP.COHERENCE|CAPON|MUSIC

    units: frequency in Hz, smax (maximum slowness search) and grid (Slowness grid spacing) in s/km

    - Example:
               `smap --method CAPON --fmin 1.0 --fmax 3.0 --grid 0.01`
               `smap --method MUSIC --fmin 1.0 --fmax 3.0 --grid 0.01  --nsignals 1`

- `stack [options]`: list_traces --method [mean|pw|root]
    - Example:
                `stack`
                `stack 0,3,7`
                `stack 0-5 --method root:4`
                `stack all --method pw:2`

- `xcorr [--ref <i>] [--mode full] [--normalize full]`: Apply cross-correlation.
    - Example: `xcorr --ref 0 --mode full --normalize full`

- `plot_type <type>`: Change plot layout (`standard`, `record`, `overlay`).
    - Example: `plot_type overlay`

- `cut`: Trim traces based on:
    - span time selected: `cut` # cut waveforms according to the starttime and endtime selected by previously draging with right mouse to select start and end time in picking mode.
    - Phase: `cut --phase P 5 10` # cut the waveforms 10 seconds before and 20 seconds after the P wave (for traces with picked P wave)
    - Reference marker: `cut --reference 10 20` # cut the waveforms 10 seconds before and 20 seconds after the common reference.
    - UTC: `cut --start "2024-01-01 12:00:00" --end "2024-01-01 12:01:00"` # cut all waveforms acoording to the span time

- `shift --phase <name>` or `--phase_theo <name>`: Time-align traces.
    - `shift --phase S` # according picked phase S-wave
    - `shift --phase_theo S` # according to the theoretically estimated S-wave. Only if event info is in header. see surfquake processing -e event

- `concat`: Merge overlapping trace segments using interpolation. Merge daily seismograms by net station channel

- `pm`: Perform particle motion analysis (Z-N-E style).

- `write --folder_path <path>`: Export displayed traces to `.h5`.

- `info` : Print header information from displayed traces

- `p`: Return to interactive pick mode.

- `n` / `b`: Navigate to next/previous page.

- `help [command]`: Show help or detailed help for a command.

- `exit`: End command prompt and close plotting.

**Very important:** Exit from auxiliary plots, let's say spectrum, spectrogram or cwt by pressing `enter` / `esc`, NOT closing the GUI. Otherwise the prompt is frozen an you will not able to back to picking mode.

---

This system offers a lightweight yet highly customizable interface for interactive seismogram analysis in SurfQuake. It blends visual inspection with real-time metadata editing to support modern waveform workflows.
