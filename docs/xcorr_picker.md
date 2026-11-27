# surfquake ccpick — cross-correlation picker

Nov 23, 2026 · @Roberto Cabieces

## Overview

`surfquake ccpick` finds P and S arrivals by template matching: it slides waveforms of a known event (the *master*) along the project data and picks wherever the data look like it. Its pick files are identical to those of `surfquake pick` (PhaseNet), so association and location work unchanged.

It has two modes:

| Mode | What it matches | Detection statistic | Best for |
| --- | --- | --- | --- |
| `simple` (default) | One template channel against one data channel | Raw correlation coefficient (default 0.7) | A single station, quick checks |
| `network` | All channels of one master event at once | Scaled network correlation coefficient (default 6), as in Gibbons & Ringdal (2006) | Networks and arrays; detecting smaller events with fewer false alarms |

ccpick complements PhaseNet rather than replacing it: it only finds events similar to a master, at about the same place and with a similar mechanism. A typical workflow is to build the catalogue with PhaseNet, choose the best events as masters, then run ccpick to fill in the small events.

ccpick does not need TensorFlow.

## Quick start

The smallest run needs a project file, a folder with one template and an output folder.

1. Create the project as usual (`surfquake project ...`).
2. Put a template in a folder. The easiest way: open the data in the plotting tool, pick the P (or S) on the master event, then save the trace with `write --folder_path ./templates`. The pick is stored in the H5 file's header.
3. Run the picker:

```bash
# simple mode: one template channel per data channel
surfquake ccpick -f ./project.pkl -t ./templates -o ./picks_cc

# network mode: all channels of the master event together
surfquake ccpick -f ./project.pkl -t ./templates -o ./picks_cc --mode network
```

The first line of the run tells you where the templates came from and how many were found, for example `[ccpick] 4 template trace(s) from trace headers (stats.picks) -> 1 template(s)`. The run ends with a summary of stations and P/S picks.

## Preparing templates

Templates are waveform files in one folder, in any format ObsPy can read (MSEED, SAC, H5, ...). For each template ccpick needs its **phase** (P or S) and its **onset time**. It takes them from one of two sources, in this order.

### Option 1: a `templates_info.txt` file

If the folder contains `templates_info.txt`, it is the only source used. One line per template file, whitespace or comma separated, `#` for comments:

```
# file                        phase  pick_time (UTC)           event
WM.EMAL..HHZ.20240726.mseed   P      2024-07-26T06:42:31.520   EV1
CA.ARBS..HHZ.20240726.mseed   P      2024-07-26T06:42:33.050   EV1
WM.EMAL..HHN.20240726.mseed   S      2024-07-26T06:42:34.100   EV1
WM.EMAL..HHZ.20240901.mseed   P      2024-09-01T12:00:05.000
```

- **file**: path relative to the templates folder. One trace per file (only the first trace is read), already cut around the phase.
- **phase**: P or S.
- **pick\_time**: absolute UTC onset time. It must lie inside the template window.
- **event** (optional, network mode): lines with the same event and phase form one multichannel template. Lines without event are single-channel templates.

### Option 2: picks in the trace header (no text file)

Without `templates_info.txt`, ccpick reads every file in the folder and its subfolders and uses the SurfQuake pick header, `tr.stats.picks`. That is where the plotting tool keeps manual picks, and its `write` command saves them to H5.

- Every trace with a P and/or S pick gives one template per phase. Phase names such as Pg, Pn or Sg count as P or S; other phases (for example `?`) are ignored.
- If a trace has two picks of the same phase, only the first is used and a warning is printed.
- The template is cut to 1 s before and 4 s after the pick (`--pre_pick`, `--post_pick`), so you can save full-length traces without cutting them yourself.
- In network mode, all templates in the same folder or subfolder belong to one master event. Keep each master event in its own subfolder.
- Files that are not waveforms are skipped.

### Rules for both options

- The template's network, station and channel codes must match the data it will be compared with. The location code is ignored.
- Templates are read with automatic format detection, whatever `read.format` says in the global config. H5 templates therefore work with MiniSEED data.
- Templates are resampled to the data's sampling rate if needed.
- In network mode, each template must keep its **true start time**: the time differences between channels are part of the template.
- A good template is short (typically 2–5 s at local distances) and has a high signal-to-noise ratio. Cut P windows on vertical channels and S windows on horizontals.

If neither source gives a template, ccpick stops with an error that explains both options.

## Simple mode

In simple mode every template is one channel, and a pick is made wherever its correlation with the matching data channel reaches the threshold (default 0.7).

For each project file and each trace:

1. Gaps are filled with zeros and the trace is trimmed to the UTC day that holds most of its samples (the same rule as PhaseNet).
2. Data and templates get the same processing: demean, linear detrend, short taper, zero-phase 4-pole bandpass `--fmin` to `--fmax` (default 2–15 Hz).
3. ObsPy's `correlation_detector` slides every template of that channel along the trace.
4. Each detection becomes a pick: the time is where the template's onset falls on the data, the weight is the correlation value (0 to 1).

When several templates detect the same arrival, only the best one is kept. Two picks of the same channel and phase must be at least `--distance` seconds apart (default 2 s).

The correlation coefficient's meaning depends on the template: noise alone easily reaches 0.5 with a short, narrow-band template. If you get many false picks, raise `--threshold` or use longer templates.

## Network mode (Gibbons & Ringdal 2006)

Network mode correlates all channels of one master event at once and stacks the correlation traces, not the waveforms. In the paper this detects events about one magnitude unit smaller than an STA/LTA on the array beam. It works because, for an event at the master's location, every channel matches its template at the same moment, even when the waveforms differ from station to station.

&#91;embedded content: network mode · from correlation to picks\]

Every detection must clear the scaled threshold and the ratio test, and the slowness test when an inventory is given; only then is each channel's own maximum turned into a pick.

### 1. Data are processed one day at a time

Files are grouped by UTC day. All template channels of that day are read together, gaps are filled with zeros, and every channel is padded to the full day, so one short station cannot shorten the day for the others. A template is only run if at least `--min_channels` of its channels are present (default 3, or fewer if the template has fewer).

### 2. Network correlation coefficient

Each channel is correlated with its own template window, giving Cⱼ(t). Because the windows keep their true start times, all Cⱼ peak together for a repeat of the master, and their average is the network coefficient:

```latex
C_A(t) = \frac{1}{M} \sum_{j=1}^{M} C_j(t)
```

### 3. Scaled coefficient: the detection statistic

A raw value of C\_A means different things for different templates and networks, so each value is judged against its local background. The RMS of C\_A is measured in two windows on either side of t, leaving a gap around t so the template's own side-lobes do not count:

```latex
\text{scaled}(t) = \frac{C_A(t)}{\mathrm{RMS}\left[\, C_A \text{ over } [t-b,\, t-a] \cup [t+a,\, t+b] \,\right]}
```

Defaults are a = 1.0 s and b = 2.5 s (`--scc_a`, `--scc_b`), and a detection needs a scaled value of at least 6 (`--threshold`). The paper used 6.0 in its tests and 5.5 at the Barentsburg mine. The value works like a signal-to-noise ratio of the correlation trace.

### 4. From one detection to a pick per station

A detection gives one time for the whole network. For each channel, ccpick looks for that channel's own correlation maximum Cⱼᴸ within ±`--lag_win` (default 0.5 s) of the detection, and its small time shift (lag):

- pick time = master pick on that channel + (detection time − template start) + lag
- weight = Cⱼᴸ, between 0 and 1 like a PhaseNet probability
- a pick is written only if Cⱼᴸ ≥ `--min_channel_cc` (default 0.3), so stations that do not really see the event get no pick

### 5. False-alarm tests

**Ratio test (always on).** If the channels truly agree, stacking at zero delay loses almost nothing and the ratio below is close to 1. If their peaks are scattered in time, the stack is much smaller than its parts and the detection is rejected:

```latex
\frac{C_A(t_d)}{\operatorname{mean}_j\, C_j^{L}} \ge 0.58
```

The threshold is `--min_ratio`. This test needs no station coordinates, so it works on any network.

**Slowness test (optional, small arrays).** With `--inventory`, ccpick fits a plane wave to the channel lags. A true repeat has the same slowness as the master, so the fitted slowness is close to zero; a detection is rejected if it exceeds `--max_slowness` (default 0.04 s/km, the paper's value). It needs at least 3 stations with coordinates. This is a simpler version of the paper's f–k analysis of the correlation traces.

Small-aperture arrays are the most prone to false alarms, because an unrelated wavefront can match on all sensors within a short time. On sparse networks a chance match at every station is unlikely, and the ratio test is usually enough.

### 6. Optional magnitude

With `--master_mag`, each detection gets a magnitude relative to the master, from the amplitude ratio between data and template (ObsPy's `template_magnitudes`). It is written to `ccpick_detections.csv`, not to the pick files.

## Command-line reference

```bash
surfquake ccpick -f PROJECT -t TEMPLATES -o OUTPUT [--mode simple|network] [-th THRESHOLD] [options]
```

| Option | Default | Mode | What it does |
| --- | --- | --- | --- |
| `-f`, `--project_file` | required | both | SurfQuake project file |
| `-t`, `--template_dir` | required | both | Folder with the templates (`templates_info.txt` or header picks) |
| `-o`, `--output_dir` | required | both | Folder for the pick files |
| `--mode` | `simple` | both | `simple` or `network` |
| `-th`, `--threshold` | 0.7 / 6.0 | both | Raw correlation in simple mode; scaled coefficient in network mode |
| `--fmin`, `--fmax` | 2 / 15 Hz | both | Bandpass applied to data and templates |
| `--distance` | 2.0 s | both | Minimum time between two detections, or two picks of the same channel and phase |
| `--pre_pick`, `--post_pick` | 1.0 / 4.0 s | both | Header-pick templates only: seconds kept before and after the pick |
| `--save_cc` | none | both | Folder where the correlation traces are saved for review (H5) |
| `--save_cc_mode` | `detections` | both | `detections`: a window around each detection; `full`: whole traces |
| `--save_cc_window` | 30 s | both | Half length of the window saved around each detection |
| `-n`, `--net` / `-s`, `--station` / `-ch`, `--channel` | all | both | Project filters, same as `surfquake pick` |
| `--min_date`, `--max_date` | all | both | Time filter, format `YYYY-MM-DD HH:MM:SS.sss` |
| `-v`, `--verbose` | off | both | Print progress per trace or template |
| `--scc_a`, `--scc_b` | 1.0 / 2.5 s | network | Gap and outer limit of the background windows |
| `--min_ratio` | 0.58 | network | Ratio test threshold |
| `--min_channel_cc` | 0.3 | network | Minimum channel correlation to write a pick |
| `--lag_win` | 0.5 s | network | Search window for each channel's own maximum |
| `--min_channels` | 3 | network | Minimum template channels present in the data that day |
| `--inventory` | none | network | StationXML file; turns on the slowness test |
| `--max_slowness` | 0.04 s/km | network | Slowness test threshold |
| `--master_mag` | none | network | Master magnitude; gives a magnitude per detection |

Examples:

```bash
# one station, stricter threshold, only July 2024
surfquake ccpick -f project.pkl -t ./templates -o ./picks -s EMAL -th 0.8 \
    --min_date "2024-07-01 00:00:00" --max_date "2024-07-31 23:59:59"

# small array with slowness test and magnitudes
surfquake ccpick -f project.pkl -t ./templates -o ./picks --mode network \
    --fmin 3 --fmax 12 --inventory stations.xml --master_mag 1.8
```

## Outputs

Both modes write the same files as `surfquake pick`, with the same columns, so association (REAL) and location (NonLinLoc) read them unchanged.

| File | Contents |
| --- | --- |
| `original_picks.csv` | One row per pick: date, NET.STA.PHASE, year, month, day, net, station, flag, seconds since midnight, date\_time, weight, amplitude, phase, channel |
| `nll_picks.txt` | Picks in NonLinLoc format; the weight is converted to a time error |
| `YYYYMMDD/NET.STA.P.txt`, `.S.txt` | One folder per day in REAL format |
| `ccpick_detections.csv` | Network mode only: one row per detected event |

How ccpick fills the columns that PhaseNet fills from its model:

- **weight**: the correlation value (simple mode) or the channel's own correlation maximum Cⱼᴸ (network mode), between 0 and 1. Correlation values are usually lower than PhaseNet probabilities, so NonLinLoc errors come out larger.
- **amplitude**: the largest absolute value in the 10 s after a P pick or 5 s after an S pick, on the data high-passed at 0.5 Hz, scaled like PhaseNet's.
- **date** and **tt**: the pick's own UTC day and seconds since midnight of that day.

`ccpick_detections.csv` has these columns:

| Column | Meaning |
| --- | --- |
| `time` | Detection time (template start aligned on the data) |
| `phase`, `template` | Phase and template name (event or folder) |
| `n_channels`, `n_picks` | Channels correlated, and picks written |
| `cc_mean` | Network coefficient C\_A at the detection |
| `scaled_cc` | Scaled coefficient, the value compared with the threshold |
| `ratio` | Ratio test value |
| `slowness` | Fitted slowness in s/km, if the slowness test ran |
| `magnitude` | Relative magnitude, if `--master_mag` was given |
| `status` | `accepted`, `rejected_ratio` or `rejected_slowness`; rejected detections have no picks |

This file is the place to tune the threshold: compare the `scaled_cc` of events you know are real with that of doubtful detections.

## Reviewing the correlation traces

With `--save_cc FOLDER`, ccpick also writes the correlation traces it used, so anyone can check why a detection was made or rejected. Files are SurfQuake H5, one folder per day, and open directly in the plotting tool. Each detection is marked in the trace header (`stats.picks`), so the plotting tool shows the marks.

```bash
surfquake ccpick -f project.pkl -t ./templates -o ./picks --mode network --save_cc ./cc_review
```

| Saving mode | What is written | Size |
| --- | --- | --- |
| `detections` (default) | One file per detection, cut to ±`--save_cc_window` seconds (default 30 s) | Small: about 40 kB per detection for 6 traces at 20 Hz |
| `full` | The whole correlation traces, one file per template and day, all detections marked | Large: about 35 MB per trace and day at 100 Hz |

What each file contains:

| Trace id | Mode | Content | Where the mark is |
| --- | --- | --- | --- |
| Data id, for example `WM.EMAL..HHZ` | both | Correlation of that channel with its template | Simple mode: the pick. Network mode: the channel's own maximum, which gives its pick |
| `CC.<template>..AVG` | network | Network coefficient C\_A | Detection time |
| `CC.<template>..SCC` | network | Scaled coefficient, the value compared with the threshold | Detection time |

- File names: `<save_cc>/YYYYMMDD/<template>_<phase>_<detection time>.h5`, or `..._day.h5` in full mode.
- In network mode, detections rejected by the ratio or slowness test are saved too. The reason is in the file name (`..._rejected_ratio.h5`) and in the mark label (`P-rejected_ratio`), so you can see what the tests removed.
- Each trace carries a `stats.ccpick` header with the template, phase, status, thresholds and the detection's values (C\_A, scaled value, ratio, slowness).
- Saving needs `obspyh5`, which SurfQuake already uses for H5 files.

## Tuning, troubleshooting and limitations

### Calibration in five steps

1. Choose a master with a high signal-to-noise ratio on every channel.
2. Choose the band where the master looks clean and similar between nearby events, and set `--fmin`/`--fmax`.
3. Run one day with the defaults. In network mode, look at `scaled_cc` for known events in `ccpick_detections.csv`.
4. Set `--threshold` just below the weakest known repeat, and check that the ratio test removes what slips in.
5. On a small-aperture array, add `--inventory` for the slowness test.

### Common problems

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| `no templates found in ...` | No `templates_info.txt` and no P/S picks in the trace headers | Add the text file, or pick and save the templates with the plotting tool's `write` |
| No picks at all | Template codes do not match the data | Check NET.STA.CHA of templates and data; the location code is ignored, the channel code is not |
| No picks at all (network mode) | Fewer channels than `--min_channels` on that day | Lower `--min_channels` or add channels |
| `pick ... is outside the template window` | Wrong pick time in `templates_info.txt`, or window cut too short | Check the UTC time and the file's start and end |
| `has two traces for channel ...` | Two master events in the same network-mode group | Use one subfolder (or one event name) per master |
| Many false picks | Threshold too low for the template length and band | Raise `--threshold`; use longer or broader-band templates; in network mode add channels |
| Known small events missed | Threshold too high, or band not adapted | Lower `--threshold` step by step; adjust `--fmin`/`--fmax` |
| High memory use (network mode) | Whole days of all template channels are loaded at once | About 8 bytes per sample per channel; for example 30 channels at 100 Hz need about 2 GB. Run fewer stations at a time |

### What ccpick cannot do

- It only finds events similar to a master: same place, to within a fraction of the dominant wavelength, and a similar mechanism. Use several masters to cover a larger zone.
- Network-mode picks assume the new event is at the master's location; each channel's lag only refines its pick locally.
- Thresholds, windows and bands must be calibrated for each site and network.

## Python API and code layout

The same picker can be used from Python. `CCPicker.pick()` returns the pick table, and the writers in `pick_output` save it in the PhaseNet formats.

```python
from surfquakecore.project.surf_project import SurfProject
from surfquakecore.cc_picker import CCPicker
from surfquakecore.phasenet import pick_output

sp = SurfProject.load_project("project.pkl")
picker = CCPicker(sp.project, "./templates", mode="network", threshold=6.0,
                  fmin=2, fmax=15, output="./picks")
picks = picker.pick()                      # pandas DataFrame, PhaseNet columns

pick_output.save_original_picks(picks, "./picks")
pick_output.write_nlloc_format(picks, "./picks")
pick_output.convert2real(picks, "./picks")
picker.write_detections("./picks")         # network mode: ccpick_detections.csv
print(picker.detections.head())
```

The keyword arguments of `CCPicker` are the command-line options without dashes (`min_ratio`, `lag_win`, `pre_pick`, ...). `inventory` takes an ObsPy `Inventory`, not a file name.

| File | Role |
| --- | --- |
| `surfquakecore/cc_picker/cc_picker.py` | `CCPicker`: template loading (text file or header picks), simple and network modes, false-alarm tests |
| `surfquakecore/phasenet/pick_output.py` | Pick table format and writers shared with PhaseNet, without TensorFlow |
| `surfquakecore/cli/main.py` | `_ccpick()` command and `_filter_project()` project filters |

Reference: S. J. Gibbons and F. Ringdal (2006). The detection of low magnitude seismic events using array-based waveform correlation. *Geophysical Journal International*, 165, 149–166. [doi:10.1111/j.1365-246X.2006.02865.x](https://doi.org/10.1111/j.1365-246X.2006.02865.x)
