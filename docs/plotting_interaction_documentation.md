# SurfQuake Plotting and Interactive Commands

SurfQuake includes an interactive plotting tool designed for seismological work. It is much more than a viewer: in the same window you can **look at your waveforms, pick phases, mark reference times, select time windows, process the traces, and run analyses** (spectra, spectrograms, wavelets, beamforming, particle motion, cross-correlation...) without leaving the plot. Everything you do is stored in the trace headers and can be saved with the processed data.

The same tool also produces publication figures, and it can run **without a screen** to save the figures of thousands of traces automatically.

This page explains:

1. [How to open the plot](#opening-the-plot) and how a session works.
2. [Picking mode](#picking-mode): keys and mouse actions on the figure.
3. [Plot types](#plot-types): standard pages, record sections and overlays.
4. [Command prompt](#command-prompt-v-key): processing and analysis commands typed in the terminal.
5. [Configuration file](#configuration-file): every option of the plot.
6. [Saving figures and data](#saving-figures-and-data).

---

## Opening the plot

The plot opens with the processing commands `quick`, `processing` and `processing_daily` when they run in interactive mode (without `-a`). The plotting options are given with `--plot_config`:

```bash
surfquake make_config -c plotting_config -o ./configs

surfquake quick -w "./data/*" -c ./configs/typical_config.yaml --plot_config ./configs/plotting_config.yaml
```

The plot is optional for these commands: without `--plot_config` the default options are used. If you also give an inventory (`-i`), the array commands (`beam`, `smap`) can use the station coordinates.

!!! tip "Fast plotting"
    For long or numerous traces, `-f` / `--fast` draws every trace as a decimated min/max envelope (SAC QDP style). It is much faster, with less detail in the drawn line. With a number, e.g. `-f 10`, the traces are also decimated before processing, keeping one sample out of 10 (without anti-alias filter: for viewing only, not for analysis).

### How a session works

A session alternates between two modes:

- **Picking mode** (the default): you interact with the **figure** using the keyboard and the mouse. Picks, references and time windows are set here.
- **Command prompt mode**: press `v` on the figure, and type commands in the **terminal** (`>>` prompt). Type `p` to return to picking mode.

```
           ┌──────────────── v ────────────────┐
           │                                   ▼
   ┌───────────────┐                  ┌─────────────────┐
   │ Picking mode  │                  │ Command prompt  │
   │ (figure)      │                  │ (terminal, >>)  │
   └───────────────┘                  └─────────────────┘
           ▲                                   │
           └──── p, or after a command that ───┘
                 modifies the traces
```

When you close the figure window, the terminal asks what to do next:

```
[Prompt] Finished processing. Type 'c' to continue, 'r' to reprocess this event, or 'e' to exit:
```

- `c`: continue with the next event or time span (`processing`, `processing_daily`), or finish (`quick`).
- `r`: reprocess the same data again, re-reading the config file. Useful to try new parameters.
- `e`: exit.

---

## Picking mode

Each subplot shows one trace with its id, and the traces are split into **pages** of `traces_per_fig` traces. The keyboard and mouse act on the subplot under the mouse pointer, so **place the mouse over a trace before pressing a key**.

### Keys and mouse actions

| Key / action | What it does |
|--------------|--------------|
| `n` / `b` | Next / previous page. |
| **Double left-click** | Add a pick at that time. The terminal asks for the phase and polarity, e.g. `P,U`, `S,D` or just `Pg`. |
| `1` / `2` / `3` | Quick **P** pick with polarity `?` / `U` (up) / `D` (down). |
| `4` / `5` / `6` | Quick **S** pick with polarity `?` / `U` / `D`. |
| `e` | Pick at the mouse position, typing the phase and polarity in the terminal (as the double click). Not available with the `Qt5Agg` backend. |
| `d` / `Del` / `Backspace` | Delete the pick under the cursor. |
| `c` | Clear all picks. |
| `w` | Add a **reference time** (green dashed line) at the mouse position, on **all** the traces. |
| `p` | Remove the last reference time (from all the traces). `m` does the same. |
| `u` / `i` | Set the **start** / **end** of the analysis time window at the mouse position. |
| Right-click drag | Select the analysis time window by dragging. |
| `v` | Open the command prompt in the terminal. |
| `Esc` / `Enter` | Close the current auxiliary figure (spectrum, spectrogram, slowness map...). |

The table of shortcuts is printed in the terminal when the figure opens (`show_help: true`).

### Picks

Every pick is stored in the header of its trace, in `trace.stats.picks`, as a dictionary:

```python
{
    "time": 1727123830.321,   # UTC timestamp of the pick
    "phase": "P",             # phase name: P, S, Pg, Sn...
    "amplitude": 3.21e-05,    # amplitude of the trace at the pick time
    "polarity": "U"           # U (up), D (down) or ? (unknown)
}
```

They are also written, each time you pick, to the file set by `pick_output_file` (default `./picks.csv`). This file is a phase table in the NonLinLoc observation format (station, component, phase, first motion, date and time, amplitude...), which is the input of `surfquake locate`. A pick file can be loaded back on the plot with `auto_load_pick_file: true` or with the command `load_picks`.

See [Trace Header](trace_header_documentation.md) for all the header fields and how to use them in Python.

### Reference times

A reference time (`w`) is a time mark common to all the traces, stored in `trace.stats.references` as a list of UTC timestamps. Use it to cut all the traces around the same time (`cut --reference`), or as a time tag for your own [scripts](scripts.md).

### The analysis time window

Many commands work on a **time window**: the spectral analysis (`spectrum`, `spectrogram`, `cwt`, `coh`), `save_tf`, `beam`, `smap`, `pm`, `wiener` and `cut` without arguments. Set it with `u` (start) and `i` (end), or by dragging with the right mouse button. If no window is set, the whole traces are used where possible.

---

## Plot types

| `plot_type` | Layout | Use it for |
|-------------|--------|------------|
| `standard` | One trace per subplot, `traces_per_fig` subplots per page. | Picking, inspection, all the interactive work. |
| `record` | **Record section**: all the traces in one axis, ordered by epicentral distance (needs event information in the headers, e.g. from `surfquake processing -e`). | Following phases across a network, moveout, EGFs and cross-correlations. |
| `overlay` | All the traces superimposed in one axis. | Comparing waveforms: co-located sensors, repeating events, aligned traces. |

Change it in the configuration file or from the prompt with `plot_type record`.

Record section options:

- `record_option`: `dates` (absolute UTC time, with the theoretical arrivals drawn as curves), `seconds` (seconds from the start of each trace), or `egf` (lag time, with a red dashed line at t = 0, for empirical Green's functions and cross-correlations).
- `scale_factor`: amplitude multiplier of the traces. Increase it to make the traces overlap their neighbours.

---

## Command prompt (`v` key)

Press `v` on the figure to open the SurfQuake console in the terminal. It lets you process and analyse the displayed traces with short commands, and see the result immediately.

```
=======================================================
 Command Line Interface Activated
=======================================================
[INFO] Type 'help' to see command line options or 'p' return to picking mode
>>
```

### Rules of the prompt

- `help` lists all the commands; `help <command>` shows its syntax and examples. **Always check `help <command>`**: it is the most up-to-date reference.
- **Trace indices** refer to the traces of the **current page**: `0` is the first subplot, `1` the second... Several traces: a list (`0,2`) or a range (`1-3`). `all` means all the displayed traces.
- **Commands that modify the traces** (filter, rmean, taper, normalize, flip, reverse, remove_spikes, wiener, cut, rotate, shift, concat, algebra, xcorr, conv, deconv, runpy, back2future, plot_type) **redraw the figure and return to picking mode**. Look at the result and press `v` again to continue.
- **Analysis commands** (spectrum, spectrogram, cwt, coh, beam, smap, stack, pm) open an auxiliary figure and do not modify the traces. You stay in the prompt.
- **Close auxiliary figures with `Esc` or `Enter`**, not with the window's close button: otherwise the prompt may freeze and you cannot return to picking mode.
- The arrow keys recall previous commands; `history` lists them.
- Values with spaces are written in quotes, e.g. `cut --start "2024-01-01 12:00:00" --end "2024-01-01 12:01:00"`.
- `back2future` undoes all the processing of the session (it needs `back2future: true` in the configuration).

### Navigation and session

| Command | Description |
|---------|-------------|
| `n` / `b` | Next / previous page. |
| `p` | Return to picking mode. |
| `back2future` / `b2f` | Restore the traces as they were loaded, undoing all the processing. Needs `back2future: true` in the configuration. |
| `history [N]` | Show the last N commands (default 10). |
| `info` | Print the headers of the traces (as `surfquake info`), including picks and event information. |
| `exit` | Close the interactive session. |

### Signal processing

| Command | Syntax | Description |
|---------|--------|-------------|
| `rmean` | <code>rmean [--method simple&#124;linear&#124;constant&#124;demean&#124;polynomial&#124;spline]</code> | Remove the mean or trend (default `simple`). |
| `taper` | `taper [--type <window>] [--max_percentage <f>]` | Taper the trace ends (default cosine, 0.05). |
| `filter` | <code>filter &lt;type&gt; &lt;fmin&gt; &lt;fmax&gt; [--corners N] [--zerophase True&#124;False] [--ripple R] [--rp R] [--rs R]</code> | Butterworth `bandpass`, `highpass`, `lowpass`, or `cheby1`, `cheby2`, `elliptic`, `bessel`. For highpass and lowpass give both frequencies; only the relevant one is used. |
| `normalize` / `norm` | <code>normalize [--mode trace&#124;global]</code> | `trace`: every trace peaks at 1. `global`: all traces divided by the largest amplitude, keeping relative amplitudes (use it with `sharey: true`). |
| `remove_spikes` | `remove_spikes [--window <s>] [--sigma <f>]` | Hampel filter: samples further than `sigma` robust deviations from the moving median are replaced by it (defaults 1 s, 5). |
| `wiener` | <code>wiener [--ncoeff n] [--mu off&#124;on&#124;v] [--epsilon off&#124;on&#124;e] [--update sac&#124;lms] [--start t --end t]</code> | Prediction-error (SAC WIENER) filter, designed on the selected time window, which must contain noise only. Whitens the noise and enhances onsets. |
| `flip` | <code>flip [&lt;index&gt;&#124;all]</code> | Invert the polarity. |
| `reverse` | <code>reverse [&lt;index&gt;&#124;all]</code> | Reverse the traces in time. |
| `cut` | `cut` <br> `cut --phase <name> <before> <after>` <br> `cut --reference <before> <after>` <br> `cut --start "<UTC>" --end "<UTC>"` | Trim the traces to the selected time window, around a pick, around the last reference time, or between two dates (`YYYY-MM-DD HH:MM:SS`). |
| `concat` | `concat` | Merge the segments of the same channel (gaps interpolated). |
| `rotate` | `rotate NE->RT` <br> `rotate NE->RT <baz>` <br> `rotate ZNE->LQT <baz> [inc]` <br> `rotate ->ZNE` | Rotate the components: with the back-azimuth from the headers (great circle), or with a given back-azimuth and inclination. |
| `shift` | `shift --phase <name>` <br> `shift --phase_theo <name>` | Align the traces on a picked phase, or on a theoretical arrival (needs event information, `surfquake processing -e`). |
| `algebra` | `algebra --expression <formula>` | New traces from an expression of `tr1`, `tr2`..., e.g. `algebra --expression sqrt(tr1**2 + tr2**2)`. See [Processing Stream](processing_stream_config.md). |
| `write` | `write --folder_path <path> [--all]` | Write the displayed traces (or all with `--all`) in HDF5, keeping picks, references and event information. |

### Spectral analysis

These commands use the selected time window if there is one. See [Time-Frequency Analysis](spectral_analysis_documentation.md).

| Command | Syntax | Description |
|---------|--------|-------------|
| `spectrum` / `sp` | <code>spectrum &lt;index&gt;&#124;all [loglog&#124;xlog&#124;ylog&#124;linlin] [--mode m] [--order n] [--nsig n]</code> | Amplitude spectrum. Modes: `fft` (default), `multitaper`, `burg`, `yule`, `covar`, `modcovar`, `minvar`, `music`, `ev`. |
| `spectrogram` / `spec` | <code>spectrogram &lt;index&gt; [&lt;win_s&gt; &lt;overlap_%&gt;] [&lt;clip_dB&gt;] [--method multitaper&#124;fft] [--nw n] [--dates]</code> | Spectrogram (default 5 s windows, 50 % overlap, multitaper). |
| `cwt` | <code>cwt &lt;index&gt; &lt;cm&#124;mh&#124;pa&gt; &lt;param&gt; [&lt;fmin&gt; &lt;fmax&gt;] [&lt;clip_dB&gt;] [--dates]</code> | Continuous wavelet transform (scalogram). |
| `coh` / `tf` | <code>coh [--ref i] [--kind both&#124;transfer&#124;coherence] [--method welch&#124;multitaper] [--win s] [--overlap %] [--fmin f] [--fmax f]</code> | Coherence and transfer function with respect to a reference trace. |
| `save_tf` | <code>save_tf &lt;spectrum&#124;spectrogram&#124;cwt&#124;coh&gt; &lt;index&gt;&#124;all --folder_path &lt;path&gt; [options]</code> | Save the result to disk, to plot it later with `surfquake specplot` or read it in Python. |

### Array processing and correlation

| Command | Syntax | Description |
|---------|--------|-------------|
| `beam` | `beam [--fmin f] [--fmax f] [--smax s] [--grid g] [--win s] [--overlap r]` | FK beamforming in sliding windows over the selected time window: relative power, slowness and back-azimuth. Needs the inventory (`-i`). In the figure, point at a power peak, press `1`–`6` to choose the method and `e` to see its slowness map. See [Beamforming](beamforming_documentation.md). |
| `smap` | <code>smap [--method FK&#124;CAPON&#124;MTP.COHERENCE&#124;MUSIC] [--fmin f] [--fmax f] [--smax s] [--grid g] [--nsignals n]</code> | Slowness map of the selected time window. |
| `stack` | <code>stack [&lt;indices&gt;&#124;all] [--method mean&#124;sum&#124;pw:k&#124;root:k]</code> | Stack traces (cut to their common window) and show the result: linear mean, sum, phase-weighted (`pw:2`) or n-th root (`root:4`). |
| `xcorr` | <code>xcorr [--ref i] [--mode full&#124;same&#124;valid] [--normalize full&#124;naive] [--trim True&#124;False]</code> | Cross-correlate the traces with a reference trace. |
| `conv` | <code>conv [--ref i] [--mode full&#124;same&#124;valid] [--trim True&#124;False]</code> | Convolve the traces with a reference trace. |
| `deconv` / `deconvolve` | <code>deconv [--ref i] [--water c] [--gauss a] [--tshift s] [--normalize True&#124;False] [--trim True&#124;False]</code> | Water-level deconvolution by a reference trace (e.g. receiver functions). |

### Polarization, picks and display

| Command | Syntax | Description |
|---------|--------|-------------|
| `pm` | `pm` | Particle motion of every complete three-component station (ZNE, Z12, ZYX) in the selected time window. See [Particle Motion](particle_motion_documentation.md). |
| `load_picks` | `load_picks --file <path>` | Load picks from a phase file (NonLinLoc format) and draw them. |
| `plot_type` | <code>plot_type standard&#124;record&#124;overlay</code> | Change the layout. |

### Your own scripts

| Command | Syntax | Description |
|---------|--------|-------------|
| `runpy` | `runpy <script.py>` | Run your own Python function `process(stream)` on the loaded traces; what it returns replaces them. See [Scripts](scripts.md). |

### Example session

```
>> filter bandpass 1.0 10.0 --corners 4 --zerophase True
   (the figure is redrawn: look at it, set a window with 'u' and 'i', then press 'v')
>> spectrum all loglog --mode multitaper
   (close the spectrum with Esc)
>> spectrogram 0 3.0 75 --dates
>> save_tf spectrogram 0 --win 3.0 --overlap 75 --folder_path ./time_freq
>> cut --reference 10 60
   (back in picking mode: pick the P and S waves with the keys 1-6, then press 'v')
>> write --folder_path ./processed_h5
>> p
```

---

## Configuration file

Create a fully commented configuration file with:

```bash
surfquake make_config -c plotting_config -o ./configs
```

Every key is optional: a key you remove takes its default value. All keys go under the `plotting:` section.

```yaml
plotting:
  plot_type: standard
  traces_per_fig: 6
  sort_by: false
  sharey: false
  show_legend: true
  record_option: dates
  scale_factor: 1
  show_arrivals: false
  pick_output_file: ./picks.csv
  auto_load_pick_file: false
  show_info_picks: false
  show_crosshair: false
  plot_output_path: null
  format_save: png
  show_plot: true
  show_help: true
  fast: false
  fast_points: 1000
  back2future: false
  backend: TkAgg
  rcparams:
    lines.linewidth: 0.5
```

!!! warning "YAML reminders"
    Booleans are `true` / `false`, and "nothing" is `null`. Do not end a line with a comma: `fast: false,` is read as the text `"false,"`, which counts as **true**.

### Layout

| Key | Default | Description |
|-----|---------|-------------|
| `plot_type` | `standard` | `standard`, `record` or `overlay` (see [Plot types](#plot-types)). |
| `traces_per_fig` | 6 | Traces (subplots) per page, `standard` only. |
| `sort_by` | `false` | Order of the traces: `distance`, `backazimuth` (both need event information in the headers) or `false` (input order). |
| `sharey` | `false` | `true`: the same amplitude scale in all the subplots, to compare amplitudes between traces. `false`: each trace is scaled independently. |
| `show_legend` | `true` | Show the trace id in each subplot. |

### Record section

| Key | Default | Description |
|-----|---------|-------------|
| `record_option` | `dates` | X axis: `dates` (UTC), `seconds` (from the start of each trace) or `egf` (lag time with a line at t = 0). |
| `scale_factor` | 1 | Amplitude multiplier of the traces. |

### Arrivals and picks

| Key | Default | Description |
|-----|---------|-------------|
| `show_arrivals` | `false` | Draw the origin time (blue) and the theoretical arrivals (green, dashed, labelled) stored in the headers. |
| `pick_output_file` | `./picks.csv` | File where the picks are written (NonLinLoc phase format). |
| `auto_load_pick_file` | `false` | Load the picks of `pick_output_file` when the figure opens. |
| `show_info_picks` | `false` | Show a box with the information of the current pick. |
| `show_crosshair` | `false` | Dashed lines following the mouse. `single`: vertical line in all subplots and horizontal line in the one under the mouse (recommended; `true` means this). `vertical`, `horizontal` (useful with `sharey: true`), `all`, or `false`. |

### Saving figures

| Key | Default | Description |
|-----|---------|-------------|
| `plot_output_path` | `null` | Folder where the figures are saved. `null`: not saved. If the folder does not exist, you are offered to create it. |
| `format_save` | `png` | Image format: `png`, `pdf`, `svg`, `jpg`... |
| `show_plot` | `true` | `true`: interactive window. `false`: **headless (batch) mode**: nothing is displayed, all the pages are written to `plot_output_path`, and no question is asked. |

### Performance and interaction

| Key | Default | Description |
|-----|---------|-------------|
| `show_help` | `true` | Print the table of shortcuts when the figure opens. |
| `fast` | `false` | Quick-and-dirty plotting: min/max decimated lines, much faster for long traces. The option `-f` of `quick` activates it too. |
| `fast_points` | 1000 | Points (min/max pairs) per trace in fast mode. |
| `back2future` | `false` | Keep a copy of the loaded traces so `back2future` can undo all the processing. It doubles the memory used by the traces. |

### Backend

| Key | Default | Description |
|-----|---------|-------------|
| `backend` | `TkAgg` | Graphical engine: `TkAgg` (most robust, recommended), `MacOSX` (macOS only, recommended there), `Qt5Agg` (fast, but picking and interactive commands may fail; the `e` key is disabled). Ignored in headless mode, which uses `Agg`. |

### Style (Matplotlib rcParams)

Any [Matplotlib rcParam](https://matplotlib.org/stable/users/explain/customizing.html){: target="_blank" rel="noopener noreferrer" } can be set under `rcparams` to adapt the figures to your publication style. They apply to the `standard`, `overlay` and `record` plots; unknown keys are ignored with a warning. The time axis format, the scientific notation of the Y axis and the figure size are fixed by SurfQuake.

```yaml
plotting:
  rcparams:
    xtick.labelsize: 7
    ytick.labelsize: 7
    xtick.direction: in
    xtick.minor.visible: true
    lines.linewidth: 0.5
    legend.fontsize: 8
    font.family: serif
    savefig.dpi: 300
```

---

## Saving figures and data

### Figures

| `show_plot` | `plot_output_path` | Result |
|-------------|--------------------|--------|
| `true` | `null` | Displayed, not saved. |
| `true` | `./figures` | Displayed and saved. |
| `false` | `./figures` | Not displayed, all pages saved (**batch mode**, for scripts and servers). |
| `false` | `null` | Nothing is shown or saved. |

In automatic mode (`-a`), the figures are only produced in batch mode (`show_plot: false` with a `plot_output_path`). The figures are named `sqProject_page<NNN>_<date-time>.<format>`.

### Picks, references and processed traces

- **Picks** are written to `pick_output_file` each time you pick, and stored in the trace headers.
- **Picks, references and event information** travel with the traces in their headers. To keep them, write the traces in **HDF5**: from the prompt with `write --folder_path <path>`, or automatically with `-o <folder>` and `-a`. See [Trace Header](trace_header_documentation.md).
- **Analysis results** (spectra, spectrograms, scalograms, coherence) are saved with `save_tf`, and beamforming and particle motion results with their config steps.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| A key does nothing | Place the mouse **over a trace** (inside a subplot) before pressing it, and make sure the figure window has the focus. |
| The prompt is frozen after closing a spectrum or other figure | Close auxiliary figures with `Esc` / `Enter`, not with the window button. |
| After a command, the prompt disappeared | Commands that modify the traces redraw the figure and return to picking mode. Press `v` again. |
| `back2future` does not work | Set `back2future: true` in the configuration file. |
| `sort_by: distance` or `record` does nothing | The traces need event information in their headers: use `surfquake processing` with an event file and an inventory. |
| Picking or the prompt fail with `Qt5Agg` | Use `backend: TkAgg` (or `MacOSX` on macOS). |
| Slow plots | Use `fast: true` (or `-f`), fewer `traces_per_fig`, or cut the traces first. |
