# <span style="color:darkcyan;">**Create Metadata from CLI**</span>

The `csv2xml` tool allows users to generate **StationXML metadata**
required by SurfQuake modules.

It supports two common scenarios:

1.  **Only station coordinate file available** → A valid StationXML is
    generated with default channel information.
2.  **Station file + RESP/Inventory files available** → Full response
    information is incorporated automatically.

------------------------------------------------------------------------

## Run Create Metadatafrom CLI


``` bash
# Ask for interactive help
surfquake csv2xml -h
```


```bash
# Command description
>> surfquake csv2xml -c [csv_file_path] -r [resp_files_path] -o [output_path] -n [stations_xml_name]

# Example
>> surfquake csv2xml -c ./stations_file.txt -r ./resp_files_folder -o ./output_path -n metadata.xml
```

*Arguments*

-   `-c [Required], --csv_file_path` → Path to station table 
-   `-o [Required], --output_path` → Directory for output StationXML
-   `-n [Required], --stations_xml_name` → Output filename

*Optional Argument*

-   `-r [Optional], --resp_files_path` → Directory containing RESP/Inventory files



------------------------------------------------------------------------

## <span style="color:darkcyan;">**Create Metadata from Library**</span>

### Class
[`Convert`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/create_station_xml.py)
```python
class Convert:
    def __init__(self, file_path, sep='\s+', resp_files=None):
        self.file_path = file_path
        self.respfiles = resp_files
        self.sep = sep
        self.all_resps = []
        if self.respfiles is not None:
             self.check_resps()
```         
#### Method
[`create_stations_xml`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/create_station_xml.py)
```python
def create_stations_xml()
```

## Example using library

```python linenums="1"
from surfquakecore.utils.create_station_xml import Convert
stations_file = "/Users/roberto/Documents/python_notes/my_utils/test_data/coords.txt"
path_dest = "/Users/roberto/Documents/python_notes/my_utils/test_data"
resp_files = "/Users/roberto/Documents/python_notes/my_utils/resp_files"
name = "metadata.xml"
sc = Convert(stations_file, resp_files=resp_files)
data_map = sc.create_stations_xml()
inventory = sc.get_data_inventory(data_map)
sc.write_xml(path_dest, name, inventory)
```

## Station File

The station table must contain at least the following columns:

    Net Station Lat Lon elevation start_date starttime end_date endtime

Example (space-separated or CSV format supported):

    WM ARNO 37.0988 -6.7322 117.0 2007-01-01 00:00:00 2050-12-31 23:59:59
    WM AVE 33.2981 -7.4133 230.0 2007-01-01 00:00:00 2050-12-31 23:59:59
    WM CART 37.5868 -1.0012 65.0 2007-01-01 00:00:00 2050-12-31 23:59:59

-   Columns are case-insensitive.
-   The tool automatically detects whitespace or comma-separated files.
-   Date format must follow:

```{=html}
    YYYY-MM-DD HH:MM:SS
```


Start/end times can be wide ranges (e.g., 2007 → 2050).

If no channel information is provided, a **default channel** is
created: - Channel code: `HHZ` - Sample rate: `100.0 Hz`

------------------------------------------------------------------------

- Optional Columns (Advanced Metadata)

The tool automatically uses additional columns if present.

Complete Station File

    Net Station site_name Lat Lon elevation start_date starttime end_date endtime channel location_code sample_rate azimuth dip depth clock_drift
    WM ARNO ARNO_OBS_01 37.0988 -6.7322 117.0 2007-01-01 00:00:00 2050-12-31 23:59:59 HHZ 00 100.0 0.0 -90.0 0.0 1.2e-7
    WM ARNO ARNO_OBS_01 37.0988 -6.7322 117.0 2007-01-01 00:00:00 2050-12-31 23:59:59 HHN 00 100.0 0.0 0.0 0.0 1.2e-7
    WM ARNO ARNO_OBS_01 37.0988 -6.7322 117.0 2007-01-01 00:00:00 2050-12-31 23:59:59 HHE 00 100.0 90.0 0.0 0.0 1.2e-7
    WM AVE AVE_LAND_01 33.2981 -7.4133 230.0 2007-01-01 00:00:00 2050-12-31 23:59:59 EHZ  --  200.0 0.0 -90.0 0.0
    WM AVE AVE_LAND_01 33.2981 -7.4133 230.0 2007-01-01 00:00:00 2050-12-31 23:59:59 EHN  --  200.0 0.0 0.0 0.0
    WM AVE AVE_LAND_01 33.2981 -7.4133 230.0 2007-01-01 00:00:00 2050-12-31 23:59:59 EHE  --  200.0 90.0 0.0 0.0
    WM CART CART_OBS_02 37.5868 -1.0012 65.0 2007-01-01 00:00:00 2050-12-31 23:59:59 BHZ 10 40.0 0.0 -90.0 1.5 8.0e-8
    WM CART CART_OBS_02 37.5868 -1.0012 65.0 2007-01-01 00:00:00 2050-12-31 23:59:59 BHN 10 40.0 0.0 0.0 1.5 8.0e-8
    WM CART CART_OBS_02 37.5868 -1.0012 65.0 2007-01-01 00:00:00 2050-12-31 23:59:59 BHE 10 40.0 90.0 0.0 1.5 8.0e-8
     

  -----------------------------------------------------------------------
- Station-Level Optional Columns

  ---------------------------------------- ------------------------------
  `site_name`                              Human-readable station name

  `clock_drift`                            Clock drift rate
                                           (seconds/second). Applied to
                                           all channels unless overridden

  -----------------------------------------------------------------------
- Channel-Level Optional Columns

  ---------------------------------------- ------------------------------
  `channel`                                Channel code (e.g., HHZ, HHN,
                                           HHE)

  `location_code`                          SEED location code

  `sample_rate`                            Sampling rate (Hz)

  `dip`                                    Channel dip (degrees)

  `azimuth`                                Channel azimuth (degrees)

  `depth`                                  Sensor depth (meters)

  `clock_drift`                            Channel-specific drift
                                           (overrides station-level
                                           drift)

*If multiple rows exist for the same station with different channels, all
channels will be included in the StationXML.*

------------------------------------------------------------------------

## Using RESP / Inventory Files

If a RESP (or StationXML inventory) directory is provided:

-   The tool scans recursively.
-   All channels found in RESP files are indexed by:
    `(Network, Station, Location, Channel)`
-   If the CSV does not specify channels, RESP channels are used
    automatically.
-   If the CSV specifies channels, matching RESP responses are attached
    when available.

> RESP files must correspond to the correct Network and Station
> codes.

------------------------------------------------------------------------


## Summary

The updated `csv2xml` tool now:

-   Accepts whitespace or comma-separated files
-   Validates numeric and date fields
-   Supports multi-channel stations
-   Integrates RESP metadata robustly
-   Supports OBS clock drift metadata
-   Applies intelligent defaults when optional columns are missing

This makes metadata creation flexible, robust, and suitable for both
simple deployments and advanced OBS workflows.

