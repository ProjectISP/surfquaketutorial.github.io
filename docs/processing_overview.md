# Data Processing with SurfQuake

Welcome to the **Signal Processing Module** of SurfQuake — a scalable command-line tool for applying signal processing pipelines to large collections of seismic waveform data.

Whether you're analyzing earthquake catalogs, preparing continuous data for ambient noise or array analysis, or building datasets for machine learning models, this module lets you define, control, and automate your signal processing workflow with precision: the same config file processes one trace or ten years of data.

📘 Learn more in our [**Signal Processing Tutorial**](https://projectisp.github.io/ISP_tutorial.github.io/sp_filtering/){: target="_blank" rel="noopener noreferrer" } and in the [Tutorial Book](tutorials/tutorial1.md).

Remember that you can use real data files and metadata to learn how to use surfquake processing commands with [<span style="color:#5DADE2;">Case of study</span>](https://github.com/rcabdia/test_surfquake){: target="_blank" rel="noopener noreferrer" }.

In the following link, you will find <a href="https://github.com/rcabdia/SurfQuakeCore/tree/main/examples/configs" target="_blank" rel="noopener noreferrer">config file templates</a>. You can also generate them in your working folder:

```bash
surfquake make_config -h
surfquake make_config -c config_all -o ./configs      # every trace method, as a reference
surfquake make_config -c stream_config -o ./configs   # stream methods
surfquake make_config -c ch_functions -o ./configs    # characteristic functions
```

---

## How the processing works

A processing workflow is a list of **steps** written in a YAML config file. Each step names a method and its parameters:

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
    fmax: 8.0
    corners: 4
    zerophase: True
```

The steps run in the order of their number: `process_1`, `process_2`, ... Before anything is computed, the config is **validated**: a misspelled method or a missing parameter stops the run with a message, so a long job does not fail halfway through.

There are two kinds of methods:

| Kind | Works on | Examples | Documentation |
|------|----------|----------|---------------|
| **Trace methods** | each trace independently | `rmean`, `taper`, `filter`, `remove_response`, `resample`, `whitening` | [Processing Traces](processing_config.md) |
| **Stream methods** | several traces together | `rotate`, `cross_correlate`, `stack`, `shift`, `beam`, `algebra` | [Processing Stream](processing_stream_config.md) |

!!! note "Order of execution"
    Trace methods and stream methods run as two blocks. By default (`-op trace_first`) all trace methods are applied first, and then all stream methods, each block in its own order. With `-op stream_first`, the stream methods run first. This is the natural order for most workflows: first clean each trace, then combine them.

!!! tip "General rules for config files"
    - Each `process_N` key must be **unique**. A repeated key silently replaces the earlier step.
    - Times in seconds, frequencies in Hz.
    - Write decimals with a dot (`0.5`, `2.0`): some options require a float, and `fmin: 1` fails while `fmin: 1.0` works.
    - Quote text values (`'linear'`) and station codes (`'00'`).

### Why this order? A basic signal processing recipe

Most workflows start with the same preparation, and the order matters:

1. **Merge and fill gaps** (`concat`, `fill_gaps`): FFTs and filters need continuous, evenly sampled data.
2. **Remove the mean and trend** (`rmean`): an offset or a drift would leak into the band of interest.
3. **Taper** (`taper`): brings the trace ends smoothly to zero, avoiding spectral leakage and filter ringing at the edges.
4. **Remove the instrument response** (`remove_response`): converts counts to ground motion, so that stations can be compared.
5. **Filter** (`filter`): keeps the band of your signal.
6. **Resample** (`resample`), if needed, after the anti-alias filter.
7. **Analysis**: rotation, correlation, stacking, beamforming, spectral analysis...

Some example pipelines:

| Goal | Typical steps |
|------|---------------|
| Local earthquakes, picking | `rmean` → `taper` → `filter` (bandpass 1–20 Hz, `zerophase: True`) |
| Magnitudes / ground motion | `rmean` → `taper` → `remove_response` (`VEL`, `DISP` or `Wood Anderson`) → `filter` |
| Teleseismic body waves | `rmean` → `taper` → `remove_response` → `filter` (0.02–2 Hz) → `rotate` (`NE->RT`, `GAC`) → `shift` (`phase_theo`) → `stack` |
| Ambient noise | `rmean` → `taper` → `remove_response` → `filter` → `resample` → `time_normalization` → `whitening` → `cross_correlate` |
| Receiver functions | `rmean` → `taper` → `filter` → `rotate` (`ZNE->LQT`) → `deconv` (reference: L component) |
| Array analysis | `rmean` → `taper` → `cut_stream` → `beam` |

---

## Running the Process

This tool runs via the terminal. There are three processing commands: `quick`, `processing` and `processing_daily`. They share the same config files and differ in **how they read the data**:

| Command | Input | Use it when |
|---------|-------|-------------|
| `quick` | waveform files (`-w`) | You want to process files directly, without a project. Fast workflows and exploration. |
| `processing` | a project (`-p`) or files (`-w`) and an event file (`-e`) | You want to cut and process the waveforms of a list of **events**. |
| `processing_daily` | a project (`-p`) | You work with **continuous** data: merging daily files, or processing long time spans. |

First, in the terminal you can ask for help:

```bash
surfquake quick -h
```

All of them have two modes:

- **Interactive** (default): the waveforms are plotted and you can pick, inspect and process them with the interactive command prompt (see [Plotting](plotting_interaction_documentation.md)).
- **Automatic** (`-a`): no plots and no questions, the output is written to disk. Use it for big datasets and bash scripts.

And all of them accept a **post-script**: your own Python script applied to each stream (see [Scripts](scripts.md) and `surfquake make_config -c script_template`). With `--post_script_stage` you choose when it runs: `before` plotting (good for filtering or editing headers) or `after` plotting (good to act on the picks or metadata you added).

---

### surfquake quick

    Overview:
        Process seismic traces.
        You can:
            - Apply processing steps (filtering, normalization, etc.)
            - Optionally visualize the waveforms (interactive mode)
            - Apply a user-defined post-processing script before or after plotting

    Modes:
        Default  : Interactive mode (plotting + user prompts)
        --auto   : Non-interactive mode (no plots, no prompts, outputs written automatically)

    Post-Script Logic:
        Use `--post_script` to apply a custom script to each event stream.
        Use `--post_script_stage` to control **when** it runs:
            • before : script runs before plotting (good for filtering, editing headers)
            • after  : script runs after plotting (good if picks or metadata are added)

    Examples:
        Process waveform files with a config and plot interactively:
            surfquake quick -w "./data/*.mseed" -c ./config.yaml -i ./inventory.xml -o ./out --plot_config plot.yaml

    Key Arguments:
        -w, --wave_files         [REQUIRED | OPTIONAL] Glob pattern or path to waveform files, (not needed if the config has a 'Source' section)
        -f, --fast               [OPTIONAL] Quick-and-Dirty plotting
        -c, --config_file        [OPTIONAL] YAML config defining processing steps
        -i, --inventory_file     [OPTIONAL] Station metadata (StationXML or RESP)
        -o, --output_folder      [OPTIONAL] Directory to save processed traces
        -a, --auto               [OPTIONAL] Run in automatic (non-interactive) mode
        -m, --merge              [OPTIONAL] If merge traces as first action
        -op, --order_preference  [OPTIONAL] 'trace_first' (default) or 'stream_first' — controls
                                  whether trace-level steps (rmean, filter, taper...) run before
                                  or after stream-level steps (cross_correlate, rotate, stack...)
            --min_date           [OPTIONAL] Filter Start date (format: YYYY-MM-DD HH:MM:SS), DEFAULT None
            --max_date           [OPTIONAL] Filter End date   (format: YYYY-MM-DD HH:MM:SS), DEFAULT None
            --plot_config        [OPTIONAL] Plotting settings YAML
            --post_script        [OPTIONAL] Python script to apply to each stream
            --post_script_stage  [OPTIONAL] When to run post-script: 'before' or 'after' (default: before)

With `quick` all the selected files are read and processed together as **one stream**, which is what stream methods (rotation, correlation, beamforming...) need.

!!! tip "Synthetic test signals"
    A config file with a `Source` section instead of `-w` generates synthetic signals (impulses, sines, chirps...). They are very useful to understand what a method does before applying it to real data: for example, filter an impulse to see the impulse response of the filter. See `surfquake make_config -c test_sign_config`.

---

### surfquake processing

    Overview:
        Process or cut waveforms associated with seismic events.
        You can:
            - Cut traces using event times and headers
            - Apply processing steps (filtering, normalization, etc.)
            - Optionally visualize the waveforms (interactive mode)
            - Apply a user-defined post-processing script before or after plotting

    Modes:
        Default  : Interactive mode (plotting + user prompts)
        --auto   : Non-interactive mode (no plots, no prompts, outputs written automatically)

    Post-Script Logic:
        Use `--post_script` to apply a custom script to each event stream.
        Use `--post_script_stage` to control **when** it runs:
            • before : script runs before plotting (good for filtering, editing headers)
            • after  : script runs after plotting (good if picks or metadata are added)

    Usage Example:
        surfquake processing -p "./project.pkl" -i inventory.xml -e events.txt -c config.yaml -o ./output_folder --phases P,S --plot_config plot_settings.yaml --post_script custom_postproc.py

    Key Arguments:
        -p, --project_file         [OPTIONAL] Path to an existing project file
        -w, --wave_files           [OPTIONAL] Path or glob pattern to waveform files
        -f, --fast                 [OPTIONAL] Quick-and-Dirty plotting
        -i, --inventory_file       [OPTIONAL] Station metadata file (XML, RESP)
        -e, --event_file           [OPTIONAL] Event catalog in txt format (columns: date;hour;latitude;longitude;depth;magnitude)
        -c, --config_file          [OPTIONAL] Processing configuration file (YAML)
        -a, --auto                 [OPTIONAL] Run in automatic mode
        -o, --output_folder        [OPTIONAL] Folder where processed files are saved
        -n, --net                  [OPTIONAL] Project net filter Example: NET1,NET2,NET3
        -s, --station              [OPTIONAL] Project station filter Example: STA1,STA2,STA3
        -ch, --channel             [OPTIONAL] Project channel filter Example: HHZ,BHN
        -t, --cut_time             [OPTIONAL] pre & post first arrival in seconds (symmetric)
        -cs, --cut_start_time      [OPTIONAL] Cut pre-first arrival in seconds
        -ce, --cut_end_time        [OPTIONAL] Cut post-first arrival in seconds
        -r, --reference            [OPTIONAL] The pick time for cutting the waveforms is the origin time of the event
        -op, --order_preference    [OPTIONAL] trace_first (default) | stream_first. Controls whether trace-level
                                   steps (rmean, filter, taper...) run before or after stream-level steps
                                   (cross_correlate, rotate, stack...)
        --phases                   [OPTIONAL] Comma-separated list of phases for arrival estimation (e.g., P,S)
        --vel                      [OPTIONAL] Phase speed to estimate the first arrival time (reference for cutting waveform)
        --plot_config              [OPTIONAL] Optional plot configuration file (YAML)
        --post_script              [OPTIONAL] Python script to apply per event stream
        --post_script_stage        [OPTIONAL] When to apply the post-script: before | after (default: before)

    Notes:
    You might need to set -w for individual files or -p for loading your project file

For each event of the event file, surfquake estimates the arrival times of the `--phases` at every station (or uses the origin time with `-r`), cuts the waveforms around them (`-t`, `-cs`, `-ce`) and processes **one stream per event**. The event and arrival information is stored in the trace headers, so methods like `rotate` (`GAC`) and `shift` (`phase_theo`) can use it.

---

### surfquake processing_daily

This command is different from `surfquake processing` because it is intended to **merge daily seismograms** and to work with big continuous datasets. For example, imagine that you need three days of seismograms for the normal-mode analysis of a large event, or that you would like to process all your daily MiniSEED files for ambient noise.

    Overview:
        Cut seismograms and apply processing to continuous waveforms.
        You can perform either or both of these operations.

    Modes:
        Default : Interactive mode (with plotting and prompts)
        --auto  : Non-interactive mode (no plots, no prompts, automatic output)

    Post-Script Logic:
        Use --post_script to apply a custom script per station/day stream.
        Use --post_script_stage to control when the script is applied:
            • before : apply script before plotting (e.g., for filtering or cleanup)
            • after  : apply script after plotting (e.g., to act on manual picks)

    Usage Example:
        surfquake processing_daily \
            -p ./project.pkl \
            -i inventory.xml \
            -c config.yaml \
            -o ./output_folder \
            --min_date "2024-01-01 00:00:00" \
            --max_date "2024-01-02 00:00:00" \
            --span_seconds 86400 \
            --plot_config plot_settings.yaml \
            --post_script my_custom.py \
            --post_script_stage after

    Key Arguments:
        -p, --project_file        [REQUIRED] Path to a saved project file
        -f, --fast                [OPTIONAL] Quick-and-Dirty plotting
        -o, --output_folder       [OPTIONAL] Directory for processed output
        -i, --inventory_file      [OPTIONAL] Station metadata (XML/RESP)
        -c, --config_file         [OPTIONAL] Processing configuration (YAML)
        -a, --auto                [OPTIONAL] Run in automatic mode
        -n, --net                 [OPTIONAL] Network code filter
        -s, --station             [OPTIONAL] Station code filter
        -ch, --channel            [OPTIONAL] Channel filter
        -op, --order_preference   [OPTIONAL] trace_first (default) | stream_first. Controls whether trace-level
                                  steps (rmean, filter, taper...) run before or after stream-level steps
                                  (cross_correlate, rotate, stack...)
        --min_date                [OPTIONAL] Filter Start date (format: YYYY-MM-DD HH:MM:SS), DEFAULT min date of the project
        --max_date                [OPTIONAL] Filter End date   (format: YYYY-MM-DD HH:MM:SS), DEFAULT max date of the project
        --time_tolerance          [OPTIONAL] Tolerance in seconds for time filtering, excluded files with smaller time span
        --span_seconds            [OPTIONAL] Select and merge files in sets of time spans, DEFAULT 86400
        --time_segment            [OPTIONAL] If set, process entire time window as a single merged stream of traces
        --plot_config             [OPTIONAL] Optional plotting configuration (YAML)
        --post_script             [OPTIONAL] Path to Python script for custom post-processing
        --post_script_stage       [OPTIONAL] When to apply the post-script: before | after (default: before)

The project is divided into time spans of `--span_seconds` (one day by default); the files of each span are merged and processed as one stream. With `--time_segment`, the whole window between `--min_date` and `--max_date` is merged and processed as a single stream.

---

### Event File Format (`events.txt`)

This is the input of `surfquake processing -e ./events.txt`. One event per line, columns separated by `;`:

```
date;hour;latitude;longitude;depth;magnitude
2022-02-02;23:35:29.7;42.5089;1.4293;20.7;1.71
2022-02-03;12:01:21.6;42.3047;2.2741;0.0;1.65
```

Depth in km. A template is available with `surfquake make_config -c events`.

---

### Sample Processing Configuration (`config.yaml`)

The config file defines your signal processing pipeline step by step. You can process files directly, full daily waveform files, or segments extracted around events, and the interactive plotting tool helps you to visually inspect and validate every step.

```bash
surfquake quick -w ./mseed_file -c ./config.yaml                         # reading directly mseed files
surfquake processing -p ./project.pkl -c ./config.yaml -e ./events.txt    # loading a project and using event files
```

Detailed information on every method is in [Processing Traces](processing_config.md) and [Processing Stream](processing_stream_config.md).

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
    fmax: 8.0
    corners: 4
    zerophase: True
```
