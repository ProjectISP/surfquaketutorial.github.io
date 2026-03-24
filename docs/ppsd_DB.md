# <span style="color:darkcyan;">**Probability Power Spectral Density Functions (PPSD)**</span>
The Probability Power Spectral Density Functions ([Peterson, 1993; McNamara & Buland, 2004](https://pubs.usgs.gov/publication/70026920)) are used to characterize the background seismic noise.


## <span style="color:darkcyan;">**PPSD DB CLI**</span>

This command creates a file-based database that stores Probability Power Spectral Density Functions (PPSD) in a pickle file. The file is basically a dictionary with keys net, station channel and the ObsPy object with the PPSD.

```     
        Key Arguments:
            -p,  --project_file        [REQUIRED] Path to project file (previously generated using surfquake project)
            -i,  --inventory_file      [REQUIRED] Path to stations metadata (XML or RESP)
            -s,  --save_file           [REQUIRED] Path to save the data base
            -le, --length              [OPTIONAL] Length of data segments passed to psd in seconds (default 3600)
            -ov, --overlap             [OPTIONAL] Overlap in percentage of segments passed to psd (default 50)
            -sm, --smoothing           [OPTIONAL] PSDs are averaged over a octave at each central freq (default 1)
            -n,  --net                 [OPTIONAL] project net filter (Default: *, Example WM)
            -s,  --station             [OPTIONAL] project station filter (Default: *, Example ARNO)
            -ch, --channel             [OPTIONAL] project channel filter (Default: *, Example BH?)

```

### Usage

```bash
>> surfquake ppsdDB -p [project_file] -i [inventory_file] -s [save_file] -le [Time window length] -sm [Smoothing] -ov [Overlap] -pr [Period] -nt [Net Selection]  -st [Station Selection] -ch [Channel Selection]
``` 
### Interactive help

```bash
>> surfquake ppsdDB -h
``` 

### Examples CLI

```bash
>> surfquake ppsdDB -p "./ppsd_test" -i "./meta/metadata.xml" -s "./output/test.pkl"
```

### Class
[`PPSDSurf`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/PPSD/PPSD.py)

```python
class PPSDSurf:
    def __init__(self, files_path: Union[str, SurfProject], metadata: Union[str, Inventory], length=3600, overlap=50, smoothing=1.0, period=0.125):
    
    """

    PPSDs utils for Surfquake.

    Parameters
    ----------
    files_path : str | SurfProject | dict
        Can be:
        - a filesystem path containing waveform files,
        - a SurfProject instance,
        - a SurfProject.project-like dictionary.
    metadata : str | Inventory
        Dataless / StationXML / ObsPy inventory metadata.
    length, overlap, smoothing, period
        PPSD configuration values.

    """

```


