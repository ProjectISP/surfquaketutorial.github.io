# Data Processing with SurfQuake

Welcome to the **Signal Processing Module** of SurfQuake — a powerful and scalable command-line tool for applying signal processing pipelines to large collections of seismic waveform data.

Whether you're analyzing earthquake catalogs or preparing data for machine learning models, this module allows you to define, control, and automate your signal processing workflow with precision.

📘 Learn more in our [**Signal Processing Tutorial**](https://projectisp.github.io/ISP_tutorial.github.io/sp_filtering/)

Remember that you can use real data files and metadata to learn how to use surfquake processing commands with [<span style="color:#5DADE2;">Case of study</span>](https://github.com/rcabdia/test_surfquake).


In the following link, you will find <a href="https://github.com/rcabdia/SurfQuakeCore/tree/main/examples/configs" download> config file templates</a>.

---

## Running the Process

This tool runs via the terminal. First in the terminal you can ask for help. Available commands are quick, processing and processing_daily.

```bash
surfquake command -h
```

### surfquake quick

Let's start explaining **quick** command:

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
                surfquake quick -w "./data/*.mseed" -c ./config.yaml -i ./inventory.xml --plot_config plot.yaml

        Key Arguments:
            -w, --wave_files         [REQUIRED] Glob pattern or path to waveform files
            -c, --config_file        [OPTIONAL] YAML config defining processing steps
            -i, --inventory_file     [OPTIONAL] Station metadata (StationXML or RESP)
            -o, --output_folder      [OPTIONAL] Directory to save processed traces
            -a, --auto               [OPTIONAL] Run in automatic (non-interactive) mode
            --plot_config            [OPTIONAL] Plotting settings YAML
            --post_script            [OPTIONAL] Python script to apply to each stream
            --post_script_stage      [OPTIONAL] When to run post-script: 'before' or 'after' (default: before)
        
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
        surfquake processing -p "./project.pkl" -i inventory.xml -e events.xml -c config.yaml -o ./output_folder --phases P,S --plot_config plot_settings.yaml --post_script custom_postproc.py

    Key Arguments:
        -p, --project_file         [OPTIONAL] Path to an existing project file
        -w, --wave_files           [OPTIONAL] Path or glob pattern to waveform files
        -i, --inventory_file       [OPTIONAL] Station metadata file (XML, RESP)
        -e, --event_file           [OPTIONAL] Event catalog in QuakeML format
        -c, --config_file          [OPTIONAL] Processing configuration file (YAML)
        -a, --auto                 [OPTIONAL] Run in automatic mode
        -o, --output_folder        [OPTIONAL] Folder where processed files are saved
        -n, --net                  [OPTIONAL] project net filter Example: NET1,NET2,NET3
        -s, --station              [OPTIONAL] project station filter Example: STA1,STA2,STA3
        -ch, --channel,            [OPTIONAL]  project channel filter Example: HHZ,BHN
        -t, --cut_time             [OPTIONAL] pre & post first arrival in seconds (symmetric)
        -cs, --cut_start_time      [OPTIONAL] cut pre-first arrival in seconds
        -ce, --cut_end_time        [OPTIONAL] cut ppost-first arrival in seconds
        -r, --reference            [OPTIONAL] The pick time for cutting the waveforms is the origin time of the event
        --phases                   [OPTIONAL] Comma-separated list of phases for arrival estimation (e.g., P,S)
        --plot_config              [OPTIONAL] Optional plot configuration file (YAML)
        --post_script              [OPTIONAL] Python script to apply per event stream
        --post_script_stage        [OPTIONAL] When to apply the post-script: before | after (default: before)

    Notes:
    You might need to set -w for individual files or -p for loading your project file

---

### surfquake processing_daily

This command is different to simply surfquake processing because is intended to merge daily seismograms and work for big data sets. For xample, imagine that you need three days of seismograms for normal modes analysis of an event. Or imagine you would like to process datayo your daily mseed files.

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
        surfquake processing_daily \\
            -i inventory.xml \\
            -c config.yaml \\
            -o ./output_folder \\
            --min_date "2024-01-01 00:00:00" \\
            --max_date "2024-01-02 00:00:00" \\
            --span_seconds  86400\\
            --plot_config plot_settings.yaml \\
            --post_script my_custom.py \\
            --post_script_stage after

    Key Arguments:
        -p, --project_file        [REQUIRED] Path to a saved project files
        -o, --output_folder       [OPTIONAL] Directory for processed output
        -i, --inventory_file      [OPTIONAL] Station metadata (XML/RESP)
        -c, --config_file         [OPTIONAL] Processing configuration (YAML)
        -a, --auto                [OPTIONAL] Run in automatic mode
        -n, --net                 [OPTIONAL] Network code filter
        -s, --station             [OPTIONAL] Station code filter
        -ch, --channel            [OPTIONAL] Channel filter
        --min_date                [OPTIONAL] Filter Start date (format: YYYY-MM-DD HH:MM:SS), DEFAULT min date of the project
        --max_date                [OPTIONAL] Filter End date   (format: YYYY-MM-DD HH:MM:SS), DEFAULT max date of the project
        --time_tolerance          [OPTIONAL] Tolerance in seconds for time filtering, excluded files with smaller time span
        --span_seconds            [OPTIONAL] Select and merge files in sets of time spans, DEFAULT 86400
        --time_segment            [OPTIONAL] If set, process entire time window as a single merged stream of traces
        --plot_config             [OPTIONAL] Optional plotting configuration (YAML)
        --post_script             [OPTIONAL] Path to Python script for custom post-processing
        --post_script_stage       [OPTIONAL] When to apply the post-script: before | after (default: before)
    """


### Event File Format (`event.txt`)

Please find here a very simple event file format. This is typical input for surfquake processing -e ./event.txt

```
date;hour;latitude;longitude;depth;magnitude
2022-02-02;23:35:29.7;42.5089;1.4293;20.7;1.71
2022-02-03;12:01:21.6;42.3047;2.2741;0.0;1.65
```

---

### Sample Processing Configuration (`config.yaml`)

---

The macro configuration (provided via a YAML file) defines your signal processing pipeline in a structured, step-by-step format. Each `process` entry specifies a method, its parameters, and the order in which it will be applied to each trace. You can process either full daily waveform files or extract and process specific segments based on an event file. Additionally, an interactive plotting tool is available to help you visually inspect and validate the processing steps.

Please find here a very simple event file format. This is the typical input file for this commands:


```bash
surfquake quick -w ./mseed_file -c ./config.yaml # reading directly mseed files

surfquake processing -p ./project.pkl -c ./config.yaml -e ./event.txt # loading a project and using event files
```

`config.yaml`: Detaled information of this file in Processing traces and Processing Stream.

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


