# Trace Header

SurfQuake processing and visualization tools make extensive use of the metadata embedded in ObsPy Trace objects (MiniSEED format). These headers not only contain acquisition-related data (station, channel, sampling rate) but are also extended during analysis to store **arrival picks**, **reference markers**, and more.

SurfQuake processing and visualization tools make extensive use of the metadata embedded in ObsPy Trace objects, which are often read from MiniSEED (MSEED) files. While MSEED is a widely accepted format for storing seismic time series, it is inherently read-only in terms of header modification—meaning users cannot persistently embed new metadata directly into the original file.

To overcome this limitation, SurfQuake dynamically extends the Trace.stats header in memory during processing and interaction. This enriched metadata includes not only core acquisition parameters like network, station, channel, and sampling rate, but also a wide array of analysis results such as:

Arrival picks from manual or automatic phase picking

Reference markers used for alignment or annotation

Geodetic attributes like source-receiver distance, backazimuth, and theoretical arrival times

Since these enhancements cannot be stored back into standard MSEED files, SurfQuake writes processed trace data to HDF5 (.h5) format. This format supports flexible, hierarchical storage and allows full preservation of the enriched metadata alongside waveform data—ensuring reproducibility, traceability, and compatibility with downstream processing.

By adopting HDF5 for intermediate and final outputs, SurfQuake enables a more transparent and integrated seismic analysis workflow, without losing critical contextual information during each stage of the pipeline.

This guide outlines the custom fields and their use cases for SurfQuake users.

---

## Core Header Fields (ObsPy Standard)

These fields are part of the default `trace.stats` dictionary and are typically populated during reading:

| Field            | Description                                |
|------------------|--------------------------------------------|
| `network`        | Network code (e.g., `IU`, `CI`)            |
| `station`        | Station code (e.g., `ANMO`)                |
| `location`       | Location code (e.g., `00`, `10`)           |
| `channel`        | Channel name (e.g., `BHZ`, `HHN`)          |
| `starttime`      | Trace start time (`UTCDateTime`)           |
| `endtime`        | Trace end time (`UTCDateTime`)             |
| `sampling_rate`  | Sampling rate in Hz                        |
| `npts`           | Number of points                           |

---

## Header Info from Command Line

surfquake facilitates a tool to explore the information of your seismogram files.
Supports standard and SurfQuake-extended headers such as picks, references, and geodetic attributes

```bash
surfquake -w [path to waveform files (e.g. './data/*Z or './mseed1,./mseed2] -c [Number of traces (columns) to show per file, default 1]
```

Example
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


## Trace header modification

If you need to modify the header of your traces, I recommend that you use *quick* or *process_daily* command and use the **rename** process. This process simply maps the original traces header information into the new one. Here, you will find an example of config file with the process of mapping network, station, location and channel codes.


```bash
Analysis:
  process_1:
      name: 'rename'
      
      networks:
        IM: IH      # Replace network XX → YY
        IK: IN
      
      stations:
        IL02: XX02
        ARNO: MELI
      
      channels:
        SHZ: BHZ     # Entire channel replacement

      components:
        Z: E         # Only the last component letter (Z → E, N → Z, etc.)
```


## SurfQuake-Extended Header Fields

These fields are dynamically added during processing or interactive analysis:

### `trace.stats.picks` (list of dicts)

Stores manually or automatically picked phase arrivals.

Each entry contains:
- `time` (float): UTC timestamp of the pick
- `phase` (str): Picked phase (e.g., `P`, `S`, `Lg`)
- `amplitude` (float): Measured amplitude at pick time
- `polarity` (str): `U` (up), `D` (down), or `?`

```json
{
  "time": 1727123830.321,
  "phase": "P",
  "amplitude": 3.21e-05,
  "polarity": "U"
}
```

### `trace.stats.references` (list of float)

Stores timestamps marking user-defined reference lines during plotting (e.g., using keypress `w`). These are used for alignment, cross-checking picks, or beamforming time tags.

### `trace.stats.geodetic`

Stored during preprocessing via `set_header_func`:

| Field            | Description                             |
|------------------|-----------------------------------------|
| `distance`       | Source-receiver distance (km)           |
| `baz`            | Backazimuth from station to source (°)  |
| `az`             | Azimuth from source to station (°)      |
| `incidence`      | Theoretical incidence angle (°)         |
| `arrivals`       | Dict of theoretical arrival times per phase name (UTC timestamp) |

### Example:

```python
trace.stats.geodetic = {
    "distance": 254.7,
    "baz": 123.4,
    "az": 303.4,
    "incidence": 31.5,
    "arrivals": {
        "P": 1727123829.20,
        "S": 1727123844.03
    }
}
```

---

## Trace Header Management in SurfQuake

### During Processing

- Headers are updated by processing steps such as:
  - `shift`: aligns data using theoretical or picked times
  - `rotate`: may use `geodetic` data to perform LQT rotations
  - `beamforming`: may store attributes related to time or azimuth reference

### During Plotting

- Interactive actions modify headers:
  - **Press `e`**: Add pick to `trace.stats.picks`
  - **Press `w`**: Add reference marker to `trace.stats.references`
  - **Press `d` or `p`**: Remove the last pick or reference
  - **Press `m`**: Remove all reference lines

---

## Best Practices

- Pick and reference data are stored in-memory and optionally exported to CSV.
- Ensure that when traces are saved or passed between steps, `trace.stats` is retained.
- You can inspect or modify these attributes using standard Python syntax:
  ```python
  trace.stats.picks.append({...})
  trace.stats.references = []
  ```

---


### Why This Matters

- You can **add custom metadata** to `trace.stats` for later use (e.g., annotations, flags, temporary calculations).
- You can **modify waveform data** directly, apply filters, or normalize traces.
- You can **access event and station metadata** for context-aware processing.
- Changes made in the user script are **preserved** if the output is saved to HDF5 or passed through the pipeline.

This flexibility makes SurfQuake an ideal platform for advanced, reproducible seismic workflows tailored to your research or operational needs.



## Notes

These headers extend the utility of MiniSEED traces in SurfQuake workflows by embedding processing history and interactive annotations, making the data more portable and self-descriptive across analysis steps.

