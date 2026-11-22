# Trace Header

Every seismic trace carries two things: the **samples** (`tr.data`, a NumPy array) and the **header** (`tr.stats`), the metadata that gives the samples a meaning: which station and channel recorded them, when the first sample was taken, how many samples per second...

SurfQuake reads your files with ObsPy, so every trace is an ObsPy `Trace` with a `tr.stats` header. During processing and interaction, SurfQuake **extends** this header with the results of your analysis:

- **Picks**: phase arrivals picked manually or automatically.
- **References**: reference times marked on the plot, used to cut or align traces.
- **Geodetic information**: event location, source–receiver distance, azimuths and theoretical arrival times.

The header is the memory of the workflow: what you add in one step can be used by the next one. A pick made on the plot is used by `cut` (`method: phase`) or `shift` (`phase`); the theoretical arrivals computed by `surfquake processing` are used by `shift` (`phase_theo`); the back-azimuth is used by `rotate` (`GAC`); and your own [scripts](scripts.md) can read and write any field.

---

## Why HDF5?

MiniSEED, the standard format of seismic archives, has a **fixed header**: it stores the station codes, the start time and the sampling rate, but there is no place for picks, references or event information. If a processed trace were written back to MiniSEED, all this information would be lost.

For this reason SurfQuake writes processed traces in **HDF5** (`.h5`) format. HDF5 is a hierarchical format that stores the samples together with the **whole** `tr.stats` dictionary, including the SurfQuake fields. When you read the file again (with surfquake or with ObsPy), the trace comes back exactly as you left it: data, picks, references and geodetic information.

```bash
surfquake quick -w "./data/*" -c ./config.yaml -o ./processed -a
```

The output files are named `<trace_id>.D.<year>.<julday>.h5`, e.g. `ES.EALB..HHZ.D.2018.233.h5`. They can be read like any other waveform file by every surfquake command, and in Python:

```python
from obspy import read

tr = read("./processed/ES.EALB..HHZ.D.2018.233.h5")[0]
print(tr.stats.picks)       # the picks are still there
print(tr.stats.geodetic)    # and the event information
```

ℹ️ Reading HDF5 in your own Python code needs the `obspyh5` plugin, which is installed with surfquake.

---

## Core Header Fields (ObsPy Standard)

These fields are part of the default `trace.stats` dictionary and are filled when the file is read:

| Field            | Description                                |
|------------------|--------------------------------------------|
| `network`        | Network code (e.g., `IU`, `ES`)            |
| `station`        | Station code (e.g., `ANMO`)                |
| `location`       | Location code (e.g., `00`, `10`, or empty) |
| `channel`        | Channel code (e.g., `BHZ`, `HHN`)          |
| `starttime`      | Time of the first sample (`UTCDateTime`)   |
| `endtime`        | Time of the last sample (`UTCDateTime`)    |
| `sampling_rate`  | Samples per second, in Hz                  |
| `delta`          | Time between samples, in s (1 / sampling_rate) |
| `npts`           | Number of samples                          |

The **trace id** is `network.station.location.channel`, e.g. `ES.EALB..HHZ` (the location is empty here). The channel code follows the SEED convention: the first letter is the band (sampling rate and period: `H` = high broadband, `B` = broadband, `S` = short period), the second the instrument (`H` = high-gain seismometer, `N` = accelerometer), and the third the **component** (`Z`, `N`, `E`, or `1`, `2` for sensors not oriented to North and East).

---

## Header Info from Command Line

The `info` command prints the header of your files on screen, including the SurfQuake fields (picks, references and, when available, the event and geodetic information). It is the quickest way to check your data before processing: are all sampling rates the same? Do the start times match? Do the processed files keep their picks?

```bash
surfquake info -h
surfquake info -w [path to waveform files, e.g. './data/*Z' or './mseed1,./mseed2'] -c [number of traces (columns) shown side by side, default 1]
```

The files can be given with a wildcard (`'./data/*Z'`) or as a list separated by commas.

Example:

```bash
> surfquake info -w './ES.ELOB..HHZ.D.2018.233,./ES.EMAZ..HHZ.D.2018.233' -c 2

File                 | ES.ELOB..HHZ.D.2018.233   | ES.EMAZ..HHZ.D.2018.233
Network              | ES                        | ES
Station              | ELOB                      | EMAZ
Location             |                           |
Channel              | HHZ                       | HHZ
Start Time           | 2018-08-21T00:12:58.4300  | 2018-08-21T00:12:56.3977
End Time             | 2018-08-21T00:47:06.6900  | 2018-08-21T00:47:03.5077
Sampling Rate (Hz)   | 100.0                     | 100.0
Delta (s)            | 0.01                      | 0.01
Npts                 | 204827                    | 204712
Picks                | None                      | None
References           | None                      | None
```

When the traces carry event information (files processed with `surfquake processing` and an event file), `info` also shows the rows `Origin Time`, `Event Lat`, `Event Long`, `Distance (km)`, `Az (°)`, `Baz (°)`, `Incidence (°)` and `Arrivals`.

---

## SurfQuake-Extended Header Fields

These fields are added during processing or interactive analysis. A trace that has never been picked has no `picks` field: always access them safely, e.g. `getattr(tr.stats, "picks", [])`.

### `trace.stats.picks` (list of dicts)

Phase arrivals picked manually on the plot, or added by your scripts. Each entry contains:

| Key | Type | Description |
|-----|------|-------------|
| `time` | float | Time of the pick, as a UTC timestamp (seconds since 1970-01-01). Convert it with `UTCDateTime(p["time"])`. |
| `phase` | str | Phase name (e.g., `P`, `S`, `Pg`, `Lg`). |
| `amplitude` | float | Amplitude of the trace at the pick time. |
| `polarity` | str | First-motion polarity: `U` (up), `D` (down) or `?` (unknown). |

```json
{
  "time": 1727123830.321,
  "phase": "P",
  "amplitude": 3.21e-05,
  "polarity": "U"
}
```

The picks are also written to the CSV file set by `pick_output_file` in the plotting configuration (default `./picks.csv`).

### `trace.stats.references` (list of floats)

Reference times (UTC timestamps) marked on the plot with the key `w`. A reference is added to **all** the traces of the plot at the same time, so it is a common time mark: for example the origin time of an event, or the start of a window you want to cut (`cut`, `method: reference`, uses the last reference).

### `trace.stats.geodetic` (dict)

Event and source–receiver information. It is filled by `surfquake processing` when you give an event file (`-e`) and an inventory (`-i`), for each event and station:

| Key | Content |
|-----|---------|
| `otime` | Origin time of the event. |
| `event` | `[latitude, longitude, depth]` of the event (depth in km). |
| `geodetic` | `[distance_km, azimuth, back_azimuth, incidence]`: source–receiver distance (km); azimuth from the source to the station (°); back-azimuth from the station to the source (°); incidence angle of the first arrival (°). |
| `arrivals` | List of theoretical arrivals (TauP), one dict per phase: `phase`, `time` (UTC timestamp), `ray_param` (s/°) and `incident_angle` (°). |

```python
trace.stats.geodetic = {
    "otime": UTCDateTime("2018-08-21T00:27:00.5"),
    "event": [42.51, 1.43, 10.0],
    "geodetic": [254.7, 303.4, 123.4, 31.5],
    "arrivals": [
        {"phase": "P", "time": 1534811249.2, "ray_param": 13.1, "incident_angle": 31.5},
        {"phase": "S", "time": 1534811264.0, "ray_param": 23.6, "incident_angle": 31.9},
    ],
}
```

With `quick` and `processing_daily` there is no event, so the geodetic values are set to `-1` and the event to `None`.

SurfQuake also copies the back-azimuth and the incidence angle to the ObsPy fields `trace.stats.back_azimuth` and `trace.stats.inclination`, which are the ones used by `rotate` with `method: 'GAC'`.

---

## Who writes and who reads the header

| Action | Writes | Read by |
|--------|--------|---------|
| Double left-click on a trace, keys `1`–`6`, key `e` | `picks` | `cut` / `cut_stream` (`phase`), `shift` (`phase`), scripts |
| Key `w` | `references` (all traces) | `cut` / `cut_stream` (`reference`), scripts |
| `surfquake processing -e events.txt -i inventory.xml` | `geodetic`, `back_azimuth`, `inclination` | `shift` (`phase_theo`), `rotate` (`GAC`), `info`, scripts |
| Your post-scripts | anything you want | the following steps, the plot, the HDF5 output |

Interactive keys that edit the header (see [Plotting](plotting_interaction_documentation.md)):

| Key | Action |
|-----|--------|
| Double left-click | Add a pick (you are asked for the phase and polarity) |
| `1` / `2` / `3` | Quick P pick with polarity `?` / `U` / `D` |
| `4` / `5` / `6` | Quick S pick with polarity `?` / `U` / `D` |
| `e` | Manual pick (interactive input) |
| `d` / `Del` | Delete the pick under the cursor |
| `c` | Clear all picks |
| `w` | Add a reference time (all traces) |
| `p` | Remove the last reference time |

---

## Trace header modification

To change the codes of your traces (network, station, location, channel), use the **rename** method with `quick` or `processing_daily`. It maps the original codes to the new ones, for example to harmonise data from different providers or to change `1/2` components into `N/E`:

```yaml
Analysis:
  process_1:
    name: 'rename'
    networks:
      'IM': 'IH'
    stations:
      'IL02': 'XX02'
    channels:
      'SHZ': 'BHZ'      # whole channel replacement
    components:
      '1': 'N'          # only the last letter of the channel
      '2': 'E'
```

⚠️ Always quote the codes: unquoted `00` becomes `0`, and `NO` (Norway) becomes `False`. See [Processing Traces](processing_config.md) for all the options.

---

## Working with the header in Python

In your [scripts](scripts.md), or in any Python session, the header is a dictionary-like object:

```python
from obspy import UTCDateTime

for tr in stream:
    # read: always with a default, the field may not exist
    picks = getattr(tr.stats, "picks", [])
    p_picks = [p for p in picks if p["phase"] == "P"]

    # write a pick
    if "picks" not in tr.stats:
        tr.stats.picks = []
    tr.stats.picks.append({"time": UTCDateTime("2018-08-21T00:27:15").timestamp,
                           "phase": "P", "amplitude": 0.0, "polarity": "?"})

    # theoretical P arrival (only after surfquake processing with an event file)
    arrivals = tr.stats.get("geodetic", {}).get("arrivals", [])
    p_theo = next((UTCDateTime(a["time"]) for a in arrivals if a["phase"] == "P"), None)

    # your own fields
    tr.stats.quality = "good"
```

!!! tip "Best practices"
    - Store times as UTC timestamps (floats), like SurfQuake does: they are simple, sortable and safe to write in HDF5.
    - Read optional fields with a default (`getattr`, `.get`), because traces without picks or events do not have them.
    - Keep the `tr.stats` when you create new traces in a script (`tr.copy()`), otherwise the picks and geodetic information are lost.
    - Write your results in HDF5 (`-o` with `-a`) to keep the header between runs.

---

## Notes

The extended header turns each trace into a self-descriptive object: the waveform travels together with its acquisition information, its picks, its event and the results of your own analysis, from one processing step to the next and from one run to the next.
