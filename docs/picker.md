
# <span style="color:darkcyan;">**Phase Picker**</span>

The phase picker of surfQuake uses the deep neural network **PhaseNet** (Zhu and Beroza, 2019) to estimate the arrival times of the **P** and **S** waves of local and regional earthquakes in continuous data. The picks are saved in three formats, ready to be used by the associator (`surfquake associate`), the location (`surfquake locate`), the first-motion polarities (`surfquake polarity`) or the interactive plot.

For a complete example with real data, see the [Phase Picking Tutorial](tutorials/picking_tutorial.md).

## <span style="color:darkcyan;">**How it works**</span>

PhaseNet was trained with a large set of P and S arrivals picked by analysts. For every sample of a seismogram it returns three probabilities that add up to 1: **P**, **S** and **noise**. A **pick** is a peak of the P (or S) probability curve higher than a threshold, and its height is the **probability** of the pick, between 0 and 1.

surfQuake runs PhaseNet on every file of a [project](project.md):

1. **Reading and preprocessing.** Every file is read, detrended (spline), merged (gaps filled with zeros), **resampled to 100 Hz** if it has another sampling rate, and **high-pass filtered at 0.5 Hz**. The trace is trimmed to the UTC day that contains most of its samples.
2. **Prediction.** The network computes the P, S and noise probabilities along the whole trace. Files are processed in batches (3 files at a time by default), on the GPU if available.
3. **Peak detection.** The peaks of the P and S curves above the thresholds become picks. Two picks of the same phase must be at least 50 samples apart (0.5 s at 100 Hz).
4. **Amplitude.** For every pick, the maximum absolute amplitude of the trace in a window after it (10 s for P, 5 s for S, or until the next pick) is measured.
5. **Output.** The picks are written in three formats (see [Output files](#output-files)).

!!! note "Things to know before picking"
    - **Each file is picked alone**, one component at a time. The same arrival is usually picked on Z, N and E, giving up to three picks per arrival. The associator handles these duplicates.
    - **PhaseNet works at 100 Hz**, the sampling rate used to train it. Files with another sampling rate are resampled automatically; the pick times are not affected.
    - **All the files should cover the same time span.** The input length of the network is taken from the first file, so longer files are cut. Daily files are the ideal input.
    - Components named **1** and **2** are treated as **E** and **N**.

## <span style="color:darkcyan;">**Phase picker from CLI**</span>

```
Key Arguments:
        -f, --project_file        [REQUIRED]    Path to your seismic project file
        -o, --output_dir          [REQUIRED]    Directory to save pick results (created if it does not exist)
        -pt, --p_thresh           [OPTIONAL]    Threshold for P-wave probability (0–1) (default: 0.3)
        -st, --s_thresh           [OPTIONAL]    Threshold for S-wave probability (0–1) (default: 0.3)
        -n, --net                 [OPTIONAL]    Network code filter (default: *)
        -s, --station             [OPTIONAL]    Station code filter (default: *, e.g. "SFS|ARNO")
        -ch, --channel            [OPTIONAL]    Channel filter (default: *, e.g. "BH?")
        --min_date                [OPTIONAL]    Filter Start date (format: YYYY-MM-DD HH:MM:SS), DEFAULT min date of the project
        --max_date                [OPTIONAL]    Filter End date   (format: YYYY-MM-DD HH:MM:SS), DEFAULT max date of the project
        --device                  [OPTIONAL]    auto | cpu | gpu (default: auto, the GPU if available)
        -v, --verbose             [OPTIONAL]    Print the number of P and S picks of every file
```

The filters `-n`, `-s` and `-ch` accept a pattern with `*` and `?` (`"HH?"`, `"E*"`) or a list of names separated by `|` (`"ARNO|UCM|EMAL"`). Write them between quotes. `--min_date` and `--max_date` select the **files** of the project that cover the interval; they do not cut the files.

### Interactive help

```bash
>> surfquake pick -h
```

### Run Phase Picker from CLI

```bash
>> surfquake pick -f ./project/surfquake_project.pkl -o ./picks -pt 0.3 -st 0.3 --verbose
```

Only the vertical components of some stations, on the CPU:

```bash
>> surfquake pick -f ./project/surfquake_project.pkl -o ./picks -s "ARNO|UCM" -ch "HHZ" --device cpu
```

At the end, a summary is printed with the number of files, stations with picks, P and S picks, and the total time.

### Choosing the thresholds

A **low** threshold finds the arrivals of small events but also picks noise; a **high** threshold keeps only clear arrivals but misses small events and many S waves, whose probabilities are usually lower than those of the P waves. The usual strategy is to keep the default (0.3) and let the association and the location reject the false picks, which do not fit with the other stations.

## <span style="color:darkcyan;">**Output files**</span>

```
--output_dir
    --original_picks.csv
    --nll_picks.txt
    --20220131
        --CA.ARBS.P.txt
        --CA.ARBS.S.txt
        ......
```

### original_picks.csv

The complete table, one line per pick:

    date,fname,year,month,day,net,station,flag,tt,date_time,weight,amplitude,phase,channel
    20220131,CA.ARBS.P,2022,1,31,CA,ARBS,1,39383.88,2022-01-31T10:56:23.880000,0.5383206605911255,8557892.700195312,P,HHZ
    20220131,CA.ARBS.S,2022,1,31,CA,ARBS,1,85480.59,2022-01-31T23:44:40.590000,0.30124416947364807,8481788.269042969,S,HHZ

| Column | Meaning |
|--------|---------|
| `date` | Day of the file (YYYYMMDD) |
| `fname` | `NET.STATION.PHASE` |
| `year`, `month`, `day` | Day of the file |
| `net`, `station`, `channel` | Where the pick was made |
| `flag` | Always 1 |
| `tt` | Arrival time in seconds from 00:00:00 of the day |
| `date_time` | Arrival time (UTC) |
| `weight` | Probability of the pick given by PhaseNet (0–1) |
| `amplitude` | Maximum amplitude after the pick, in counts multiplied by a fixed factor (2080 × 25). It is not a calibrated amplitude: use it only to compare picks of the same channel |
| `phase` | P or S |

### Daily folders (REAL format)

One folder per day (`YYYYMMDD`) and one file per station and phase (`NET.STATION.P.txt`, `NET.STATION.S.txt`), with three columns: `tt`, probability and amplitude, sorted by time. This is the input of the associator REAL (`surfquake associate -p output_dir`).

    39383.88 0.538321 8.557893e+06

### nll_picks.txt (NonLinLoc format)

All the picks in the NonLinLoc phase format:

    Station_name	Instrument	Component	P_phase_onset	P_phase_descriptor	First_Motion	Date	Hour_min	Seconds	Err	ErrMag	Coda_duration	Amplitude	Period
    ARBS   ?    HHZ  ? P      ? 20220131 1056 23.8800 GAU 5.00e-01 -1.00e+00 8.56e+06 -1.00e+00

The uncertainty `ErrMag` (in seconds) is obtained from the probability of the pick:

| Probability | ErrMag (s) |
|-------------|------------|
| > 0.95 | 0.02 |
| 0.90 – 0.95 | 0.04 |
| 0.80 – 0.90 | 0.07 |
| 0.70 – 0.80 | 0.10 |
| 0.60 – 0.70 | 0.15 |
| ≤ 0.60 | 0.50 |

This file can be loaded in the interactive plot with `load_picks --file nll_picks.txt` (see [Plotting Interaction](plotting_interaction_documentation.md)), and is the input of `surfquake polarity`.

## <span style="color:darkcyan;">**Phase Picker from Library**</span>

### Classes

[`PhasenetISP`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
class PhasenetISP:
    def __init__(files, batch_size=3, highpass_filter=0.5, min_p_prob=0.3, min_s_prob=0.3, min_peak_distance=50,
                 amplitude=False, output=None, device="auto", verbose=False):

    """
    Main class to initialize the picker

    :param files: the project dictionary (SurfProject.project), with the paths of the seismograms and their stats
    :type dict: required (see Project section)

    :param batch_size: number of files processed at once (larger batch size uses more memory)
    :type int:

    :param highpass_filter: corner frequency (Hz) of the high-pass filter applied to the raw seismogram.
                            Set to 0 to not apply any pre-filter
    :type float:

    :param min_p_prob: probability threshold for P picks
    :type float:

    :param min_s_prob: probability threshold for S picks
    :type float:

    :param min_peak_distance: minimum distance in samples between two picks of the same phase
    :type int:

    :param amplitude: if True, the amplitude after every pick is measured
    :type bool:

    :param output: output directory, created if it does not exist (optional)
    :type str:

    :param device: "auto" (GPU if available, otherwise CPU), "cpu" or "gpu"
    :type str:

    :param verbose: print the number of P and S picks of every file
    :type bool:

    :returns:
    :rtype: :class:`surfquakecore.phasenet.phasenet_handler.PhasenetISP`
    """
```

[`PhasenetUtils`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py): static methods to convert and write the picks.

### Methods

[`PhasenetISP.phasenet`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
# instance method
def phasenet(self):
    """
    Runs PhaseNet on all the files of the project.

    :return: a pandas DataFrame with the raw predictions (one row per file)
    """
```

[`PhasenetUtils.split_picks`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
@staticmethod
def split_picks(picks):
    """
    Converts the output of phasenet() into a table with one row per pick
    (the columns of original_picks.csv).

    :param picks: output of PhasenetISP.phasenet()
    :type picks: pandas DataFrame
    :return: pandas DataFrame
    """
```

[`PhasenetUtils.convert2real`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
@staticmethod
def convert2real(picks, pick_dir: str, clean_output_folder=False):
    """
    Writes the picks in daily folders, in the input format of the associator REAL.

    :param picks: output of split_picks
    :param pick_dir: output directory
    :param clean_output_folder: if True, the content of pick_dir is deleted first
    """
```

[`PhasenetUtils.save_original_picks`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
@staticmethod
def save_original_picks(original_picks, original_p_dir):
    """
    Writes original_picks.csv.

    :param original_picks: output of split_picks
    :param original_p_dir: output directory
    """
```

[`PhasenetUtils.write_nlloc_format`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/phasenet/phasenet_handler.py)

```python
@staticmethod
def write_nlloc_format(dataframe, output, starttime=None, endtime=None):
    """
    Writes nll_picks.txt in NonLinLoc phase format.

    :param dataframe: output of split_picks
    :param output: output directory
    :param starttime: only picks after this time (optional, "%Y-%m-%d %H:%M:%S" or UTCDateTime)
    :param endtime: only picks before this time (optional)
    """
```

### Example using library

```python linenums="1"
from multiprocessing import freeze_support
from surfquakecore.phasenet.phasenet_handler import PhasenetUtils
from surfquakecore.phasenet.phasenet_handler import PhasenetISP
from surfquakecore.project.surf_project import SurfProject

### Set Paths to project file and output folder ###
path_to_project = "./project/surfquake_project.pkl"
output_picks = "./picks"

if __name__ == '__main__':
    freeze_support()

    # Load project
    sp_loaded = SurfProject.load_project(path_to_project_file=path_to_project)

    # Instantiate the class PhasenetISP
    phISP = PhasenetISP(sp_loaded.project, amplitude=True, min_p_prob=0.30, min_s_prob=0.30, output=output_picks)

    # Running Stage
    picks = phISP.phasenet()

    # From PhaseNet output to the three output formats
    picks_results = PhasenetUtils.split_picks(picks)
    PhasenetUtils.convert2real(picks_results, output_picks)
    PhasenetUtils.save_original_picks(picks_results, output_picks)
    PhasenetUtils.write_nlloc_format(picks_results, output_picks)
```

`if __name__ == '__main__':` and `freeze_support()` are needed because the files are read in parallel.

## <span style="color:darkcyan;">**References**</span>

Zhu, Weiqiang, and Gregory C. Beroza. "PhaseNet: a deep-neural-network-based seismic arrival-time picking method." Geophysical Journal International 216.1 (2019): 261-273.
