# <span style="color:darkcyan;"> **Basic Signal Processing Tutorial 1** </span>

In this tutorial, we will learn how to work with the commands: `make_config`, `explore`, `info` and  `quick` to process your seismic waveforms.

In this tutorial we will follow this structure
- *Seismic files:* tutorial1/data
- *Metadata:* tutorial1/metadata
- *Config Files:* tutorial1/configs

    --tutorial
        --data
            --CA.ARBS..HH1.D.2022.32
            --CA.ARBS..HH2.D.2022.32
            --CA.ARBS..HHZ.D.2022.32
            ......
        --metadata
            --inv_all.xml
        --configs
            --typical_config.yaml
            --plotting_config.yaml


For this tutorial you need to download the [seismic data here](https://github.com/rcabdia/test_surfquake/tree/main/Andorra/inputs/waveforms_cut){: target="_blank" rel="noopener noreferrer" }. Optionally, download the corresponding [metadata](https://github.com/rcabdia/test_surfquake/tree/main/Andorra/inputs/metadata){: target="_blank" rel="noopener noreferrer" }.

Please, first open a terminal and type the following command to overviewall surfquake capabilities:

```bash
surfquake -h
```

## Generate Config Files

When working with seismic waveforms it might be necessary to follow a signal processing workflow and plotting configuration. Let's prepare the necessary files.

```bash
surfquake make_config -h
surfquake make_config -c typical_config -o "./tutorial1/configs"
surfquake make_config -c plotting_config -o "./tutorial1/configs"
```
The [plotting configuration](https://projectisp.github.io/surfquaketutorial.github.io/plotting_interaction_documentation/){: target="_blank" rel="noopener noreferrer" } is commented to explain the functionality. The parametrization for signal processing can be found [here for traces](https://projectisp.github.io/surfquaketutorial.github.io/processing_config/){: target="_blank" rel="noopener noreferrer" } and [here for streams](https://projectisp.github.io/surfquaketutorial.github.io/processing_stream_config/){: target="_blank" rel="noopener noreferrer" }.

---

## Explore your data

Sometimes we have really big datasets, A first view of your data coverage, including gaps.  

```bash
surfquake explore -h
surfquake explore -w './tutorial1/data/*'
```

In this scenario, you are using the wildcard `*` that means all files. No worries because only seismic traces will be readed.

Useful wildcards `*Z*` (all vertical components), `*VALC*` (only read station VALC). Wildcards are nice, but only understand file names no file headers.

![Seismic Waveform](img/explore.png)

---

## Look at the files header


Now let see the header of your files. This tell you a lot of infomation about what is inside the files, such as sampling rate, number of points, start time of the header...

Please review the following documentation:

[Trace Header](https://projectisp.github.io/surfquaketutorial.github.io/trace_header_documentation/){: target="_blank" rel="noopener noreferrer" }

```bash
surfquake info -h
surfquake info -w './tutorial1/data/*' -c 3
```

    [INFO] 30 files retained after filtering.
    File                 | CA.ARBS..HH1.D.2022.32    | CA.ARBS..HH2.D.2022.32    | CA.ARBS..HHZ.D.2022.32
    Network              | CA                        | CA                        | CA
    Station              | ARBS                      | ARBS                      | ARBS
    Location             |                           |                           |
    Channel              | HH1                       | HH2                       | HHZ
    Start Time           | 2022-02-01T01:15:00.6200  | 2022-02-01T01:15:00.6200  | 2022-02-01T01:15:00.6200
    End Time             | 2022-02-01T02:35:00.2100  | 2022-02-01T02:35:00.2100  | 2022-02-01T02:35:00.2100
    Sampling Rate (Hz)   | 100.0                     | 100.0                     | 100.0
    Delta (s)            | 0.01                      | 0.01                      | 0.01
    Npts                 | 479960                    | 479960                    | 479960
    Picks                | None                      | None                      | None
    References           | None                      | None                      | None

---

## Process your waveform

Finally it is time to play with processing. In the example we filter the traces between [0.5-8.0] Hz with a butterworth filter, zero-phase.
Please review the next documentation:

[Trace processing](https://projectisp.github.io/surfquaketutorial.github.io/processing_config/){: target="_blank" rel="noopener noreferrer" } and [plotting](https://projectisp.github.io/surfquaketutorial.github.io/plotting_interaction_documentation/){: target="_blank" rel="noopener noreferrer" }

```bash
surfquake quick -h
surfquake quick -w "./tutorial1/data/*" -c "./configs/typical_config.yaml" --plot_config "./configs/plotting_config.yaml"
```

![Seismic Waveform](img/waveform1.png)

Navigate and process interatively your data, see the menu on the terminal:

    =======================================================
    Plot Interaction Help
    =======================================================
    KEY             | ACTION
    ---             | ------
    n               | Next page
    b               | Previous page
    w               | Add reference time
    p               | Remove last reference
    c               | Clear all picks
    e               | Manual pick (interactive input)
    1–6             | Quick picks (P/S + polarity)
    d / Del         | Delete pick under cursor
    v               | Open command prompt
    u               | Set start time at cursor
    i               | Set end time at cursor
    Right-click drag | Select time window
    Double left-click | Add pick
    ESC / Enter     | Exit current mode
    =======================================================


Sometimes, we do not want to plot the seismogram itself. It turn out to process and write the output files in our computer. This can be done adding option `-a` to the command.

```bash
surfquake quick -w "./tutorial1/data/*" -c "./configs/typical_config.yaml" -a
```

On the other hand if you are interested in automatically save your plots, edit the following fields in your file `configs/plotting_config.yaml` and run another time the command with option `-a`:

    plot_output_path: /my_output_plot_folder
    show_plot: True

---

## Using Bash script

Now, how can I concatenate commands of surfquake in your terminal? The answer is creating a [bash script file](https://projectisp.github.io/surfquaketutorial.github.io/bash_script/){: target="_blank" rel="noopener noreferrer" }. 
Please, create a file, let's name it `run_commnads.sh`. Write the following commands in your file.

```bash
#!/bin/bash

# Activate your conda or virtual environment
source activate isp  # Replace with your environment manager if needed (e.g., 'conda activate surfquake')
surfquake explore -w './tutorial1/data/*'
surfquake quick -w "./tutorial1/data/*" -c "./configs/typical_config.yaml" --plot_config "./configs/plotting_config.yaml"

```

Ok, lets run the bash script. Open a terminal and go to the folder where you have the working directory and the `run_commnads.sh` file. Now write in the terminal the following commands

```bash
chmod u+x run_commnads.sh
sh run_commnads.sh
```

---

## Interactive Processing

- Open an interactive propt pressing `v`. See the terminal, the directions for on-air seismic interaction are there.
- In the new prompt `help` to open the interative signal processing menu. If you want details and examples of one command, fo example: `help filter`



```bash
╔══════════════════════════════════════════════════════════════╗
║              SurfQuake Interactive Plot Console              ║
╚══════════════════════════════════════════════════════════════╝

▸ Navigation
  ──────────────────────────────────────────────────────────
  n                      Next trace page
  b                      Previous trace page
  p                      Return to picking mode
  back2future            Restore original waveforms (undo all processing)
  exit                   Close interactive session
  history [N]            Show the last N commands entered at the prompt (default 10)

▸ Signal Processing
  ──────────────────────────────────────────────────────────
  algebra                Apply a Math expression on traces
  rmean                  Remove mean on traces
  taper                  Apply taper window on traces
  flip                   Flip the polarity of a seismic trace
  reverse                Reverse the data points of seismic traces
  filter                 Apply signal filtering
  cut                    Trim traces using picks or UTC times
  concat                 Merge trace segments, no parameters needed
  rotate                 Rotate seismic components
  shift                  Align traces by phase arrivals
  write                  Export traces to HDF5, write --folder_path <path> [--all]

▸ Spectral Analysis
  ──────────────────────────────────────────────────────────
  spectrum / sp          Plot amplitude spectrum
  spectrogram / spec     Plot multitaper spectrogram
  cwt                    Continuous wavelet transform
  save_tf                Save Time-Frequency Anlysis

▸ Array Processing
  ──────────────────────────────────────────────────────────
  beam                   FK beamforming analysis
  smap                   Slowness map estimation
  stack                  Trace stacking
  xcorr                  Cross-correlate traces
  conv                   Convolve traces

▸ Polarization & Picking
  ──────────────────────────────────────────────────────────
  pm                     Particle motion analysis
  load_picks             Import picks from file (NLL file format, load_picks --file <file_path>
  plot_type              Change plotting mode (standard, record, overlay)

▸ Seismogram Information
  ──────────────────────────────────────────────────────────
  info                   Show trace headers

▸ Run python script
  ──────────────────────────────────────────────────────────
  runpy                  Run your own script

```


