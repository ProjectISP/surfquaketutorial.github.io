# Custom User Scripts in SurfQuake

The config files cover the standard processing methods. But research is rarely standard: you may want to test a new algorithm, apply a correction only you know about, reject bad traces with your own criteria, or export a table of measurements. **User scripts** let you plug your own Python code into any SurfQuake workflow, with full access to the waveforms, their headers, the station metadata and the event.

You write a small Python file with one function. SurfQuake does the rest: reading the files, cutting around events, processing, plotting and writing the output. Your function receives the stream of traces in the middle of the workflow and returns it, possibly modified.

There are two ways to use a script:

| | Pipeline post-script | Interactive script |
|---|---|---|
| Where | `quick`, `processing`, `processing_daily` | the interactive command prompt of the plot |
| How | `--post_script myscript.py` | `runpy myscript.py` |
| Function | `run(stream, **kwargs)` | `process(stream)` |
| Runs | automatically, for every stream (each event, each day...) | once, when you type the command |
| Receives | stream + `inventory` (+ `event` with `processing`) | stream of the plotted traces |

---

## Purpose

Some examples of what you can do with a script:

- **Your own algorithms**: a new picker, a polarisation analysis, a machine learning model, a custom filter or correction.
- **Quality control**: remove dead, flat, clipped or noisy traces with your own criteria.
- **Read and write the header**: use picks, theoretical arrivals and event information, and store your own results (`tr.stats.snr = ...`). See [Trace Header](trace_header_documentation.md).
- **Add picks** automatically, which are then shown on the plot and saved in the HDF5 output.
- **Export** measurements: amplitudes, SNR, peak values, spectral parameters... to a CSV table.

---

## Where the script runs

For each stream, `quick`, `processing` and `processing_daily` follow these steps:

1. Read the files (and, with `processing`, cut them around each event and fill the [header](trace_header_documentation.md) with the event information).
2. Apply the trace methods and the stream methods of your config file.
3. **Your script**, if `--post_script_stage before` (default).
4. Plot (interactive mode).
5. **Your script**, if `--post_script_stage after`.
6. Write the output files (`-o`, with `-a`).

So your script always receives the **processed** traces, and what it returns is what is plotted and written.

- `before`: the script runs before plotting. Good for data transformations, quality control, automatic picks or header editing: you will see the result on the plot.
- `after`: the script runs after plotting. Good to act on what you did on the plot: the manual picks and references are already in the header.

---

## Pipeline post-script: `run(stream, **kwargs)`

Each script must define a function `run(stream, **kwargs)`:

- `stream`: an ObsPy `Stream` with the current group of traces (one event with `processing`, one time span with `processing_daily`, all the files with `quick`).
- `kwargs["inventory"]`: the ObsPy `Inventory` given with `-i`, or `None`.
- `kwargs["event"]`: only with `processing`, a dict with the event of the current stream: `origin_time` (`UTCDateTime`), and `latitude`, `longitude`, `depth` (km) and `magnitude` when they are in the event file.

The function must **return the stream**.

### Template

You can generate this template in your working folder with `surfquake make_config -c script_template -o ./scripts`.

```python
import numpy as np

def run(stream, **kwargs):
    '''
    User-defined post-processing hook for each stream.

    :param stream: obspy Stream
    :param inventory: obspy Inventory (passed via kwargs)
    :param event: dict with origin_time, latitude, longitude, depth, magnitude
                  (passed via kwargs, only with surfquake processing)
    '''
    inventory = kwargs.pop('inventory', None)
    event = kwargs.pop('event', None)

    if event is not None:
        print(f"Post-processing {len(stream)} traces from event at {event['origin_time']}")
    else:
        print(f"Post-processing {len(stream)} traces (no event info provided)")

    # Example: calculate global mean amplitude
    all_data = np.hstack([tr.data for tr in stream])
    print("Mean amplitude:", np.mean(all_data))

    # Example: apply a simple differencing operator to each trace
    for tr in stream:
        tr.data = np.diff(tr.data)

    return stream
```

### Usage in CLI

```bash
surfquake quick -w "./data/*" -c config.yaml -i inventory.xml \
    --post_script myscript.py --post_script_stage before

surfquake processing -p project.pkl -i inventory.xml -e events.txt -c config.yaml \
    -o ./output -a --post_script myscript.py

surfquake processing_daily -p project.pkl -c config.yaml \
    --post_script myscript.py --post_script_stage after
```

| Argument | Description |
|----------|-------------|
| `--post_script` | Path to your Python script. |
| `--post_script_stage` | `before` (default) or `after` plotting. |

---

## Examples

### Quality control: remove dead and flat traces

```python
# qc_traces.py
import numpy as np
from obspy import Stream

def run(stream, **kwargs):
    kept = Stream()
    for tr in stream:
        data = tr.data
        flat = np.ptp(data) == 0                      # constant signal
        zeros = np.mean(data == 0) > 0.2              # more than 20 % of zeros
        if flat or zeros:
            print(f"[qc] {tr.id} removed")
            continue
        kept.append(tr)
    return kept
```

### A custom automatic picker

This script runs a recursive STA/LTA on each trace and writes the onsets into the header as picks. With `--post_script_stage before`, the picks are drawn on the plot, where you can review, delete or add picks before saving.

```python
# stalta_picker.py
from obspy.signal.trigger import recursive_sta_lta, trigger_onset

STA, LTA = 1.0, 20.0        # seconds
ON, OFF = 3.5, 1.0          # trigger thresholds

def run(stream, **kwargs):
    for tr in stream:
        fs = tr.stats.sampling_rate
        cf = recursive_sta_lta(tr.data, int(STA * fs), int(LTA * fs))
        onsets = trigger_onset(cf, ON, OFF)

        if "picks" not in tr.stats:
            tr.stats.picks = []
        for on, _ in onsets:
            t = tr.stats.starttime + on / fs
            tr.stats.picks.append({"time": t.timestamp, "phase": "P",
                                   "amplitude": float(tr.data[on]), "polarity": "?"})
        print(f"[picker] {tr.id}: {len(onsets)} pick(s)")
    return stream
```

### Signal-to-noise ratio around the theoretical P (with `processing`)

With `surfquake processing` and an event file, every trace knows its theoretical arrivals. This script measures the SNR around P, stores it in the header and removes the traces below a threshold.

```python
# snr_filter.py
import numpy as np
from obspy import Stream, UTCDateTime

NOISE, SIGNAL = 5.0, 5.0     # seconds before / after P
MIN_SNR = 3.0

def rms(x):
    return np.sqrt(np.mean(x ** 2)) if len(x) else np.nan

def run(stream, **kwargs):
    kept = Stream()
    for tr in stream:
        arrivals = tr.stats.get("geodetic", {}).get("arrivals", [])
        p = next((UTCDateTime(a["time"]) for a in arrivals if a["phase"] == "P"), None)
        if p is None:
            kept.append(tr)              # no theoretical P: keep the trace untouched
            continue

        noise = tr.slice(p - NOISE, p).data
        signal = tr.slice(p, p + SIGNAL).data
        snr = rms(signal) / rms(noise)
        tr.stats.snr = float(snr)        # your own header field, saved in HDF5

        if snr >= MIN_SNR:
            kept.append(tr)
        else:
            print(f"[snr] {tr.id}: SNR = {snr:.1f}, removed")
    return kept
```

### Export a table of measurements

A script is imported **once**, so variables at the top of the file keep their value from one stream to the next. This is useful to build a table over all your events. This example writes the peak amplitude of each trace, for each event; with `remove_response` (`units: 'VEL'`) in the config, it is the peak ground velocity.

```python
# peak_table.py
import csv
import os

OUTPUT = "./peak_values.csv"

def run(stream, **kwargs):
    event = kwargs.get("event") or {}
    inventory = kwargs.get("inventory")
    new_file = not os.path.exists(OUTPUT)

    with open(OUTPUT, "a", newline="") as f:
        writer = csv.writer(f)
        if new_file:
            writer.writerow(["origin_time", "magnitude", "trace_id", "lat", "lon",
                             "distance_km", "peak"])
        for tr in stream:
            lat = lon = None
            if inventory is not None:
                try:
                    coords = inventory.get_coordinates(tr.id, tr.stats.starttime)
                    lat, lon = coords["latitude"], coords["longitude"]
                except Exception:
                    pass
            distance = tr.stats.get("geodetic", {}).get("geodetic", [None])[0]
            writer.writerow([event.get("origin_time"), event.get("magnitude"), tr.id,
                             lat, lon, distance, float(abs(tr.data).max())])
    return stream
```

---

## Interactive scripts: `runpy`

Scripts can also be run from the interactive command prompt of the plot (press `v`). This is useful to try an idea on the traces you are looking at, without leaving the plot. Here the script must define a function `process(stream)`:

- `stream`: an ObsPy `Stream` with the traces currently loaded in the plot, including your picks and references.
- What you return **replaces** the plotted traces, and the plot is redrawn. If you return `None`, the traces are left unchanged.

```bash
help runpy
runpy ./scripts/drop_flat_traces.py
```

Three examples are available with `surfquake make_config`:

| Template | What it does |
|----------|--------------|
| `script_air_1` | Removes the mean of each trace and prints a summary. |
| `script_air_2` | Drops flat or empty traces. |
| `script_air_3` | Reads the reference times and the theoretical P arrival from the header, computes an SNR around P and tags the low-SNR traces in `tr.stats`. |

```bash
surfquake make_config -c script_air_3 -o ./scripts
```

```python
# drop_flat_traces.py
from obspy import Stream

def process(stream):
    threshold = 1e-6
    kept = [tr for tr in stream if tr.data.max() - tr.data.min() > threshold]
    removed = len(stream) - len(kept)
    if removed:
        print(f"[post_script] Dropped {removed} flat/empty trace(s).")
    return Stream(traces=kept)
```

!!! tip
    You can write a single file valid for both uses, with a `process(stream)` that simply calls `run(stream)`.

---

## Combining scripts and config files

Config files and scripts work together: let the config do the standard processing, and your script the part that is specific to your research. For example, filter and normalise the traces with a config, then detect and annotate phase arrivals with your own picker:

```yaml
# config.yaml
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
    fmax: 10.0
    corners: 4
    zerophase: True
```

```bash
surfquake quick -w "./data/*" -c config.yaml --post_script stalta_picker.py --post_script_stage before
```

---

## Best Practices

- **Always return the stream** from `run()`. If the script returns nothing, there is nothing to plot or to write.
- **Errors do not stop the workflow**: if your script raises an exception, a warning is printed and the stream continues unchanged. Watch the terminal for `[WARNING] Post-script ... failed`.
- **Test your script on a few files first**, with `quick` in interactive mode, before running it on a big dataset with `-a`.
- **Read optional header fields with a default** (`getattr(tr.stats, "picks", [])`, `tr.stats.get("geodetic", {})`): traces without picks or events do not have them.
- **Store your results in the header** (`tr.stats.my_value = ...`) to use them in later steps; they are kept in the HDF5 output.
- **Print** useful information: it is your log.
- **Performance**: when a post-script is given, the stations are processed one after the other instead of in parallel, so large runs take longer.

---

This feature makes SurfQuake highly adaptable for both research and operational seismic processing pipelines: everything SurfQuake does is the frame, and your science goes inside.
