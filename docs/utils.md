
# <span style="color:darkcyan;">**Utilities**</span>

In this section we show some useful utilities to make easier how to create a metadata file and manage a catalog.

## <span style="color:darkcyan;">**Create Metadata**</span>

### <span style="color:darkcyan;">**Create Metadata from CLI**</span>

In many ocasions users only have a stations coordinates files. Other times they have the stations coordinates files and resp files (IMPORTANT: one per stations). In this context, we can use the csv2xml tool to create the metadata that is needed in sufquake for running many modules.

First, the stations file must be in this simple way.

    Net Station Lat Lon elevation start_date starttime end_date endtime
    WM ARNO 37.0988 -6.7322 117.0 2007-01-01  00:00:00 2050-12-31  23:59:59
    WM AVE 33.2981 -7.4133 230.0 2007-01-01  00:00:00 2050-12-31  23:59:59
    WM CART 37.5868 -1.0012 65.0 2007-01-01  00:00:00 2050-12-31  23:59:59
    WM CEU 35.8987 -5.3731 320.0 2007-01-01  00:00:00 2050-12-31  23:59:59
    WM CHAS 35.1837 -2.4304 110.0 2007-01-01  00:00:00 2050-12-31  23:59:59

Just separated by a simple space. Do not worry to much about starttime and end time. it can be something crazy like in the example. Optionally, the user can incorporate the root path where resp files are saved to be incorporated to the metadata.
#### Usage
```bash
>> surfquake csv2xml -c [csv_file_path] -r [resp_files_path] -o [output_path] -n [stations_xml_name]
```
#### Interactive help

```bash
>> surfquake csv2xml -h
``` 

#### Run Create Metadatafrom CLI
```bash
>> surfquake csv2xml -c ./stations_file.txt -r ./resp_files_folder -o ./output_path -n metadata.xml
```

### <span style="color:darkcyan;">**Create Metadata from Library**</span>

#### Class
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

### Example using library

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

## <span style="color:darkcyan;">**Manage catalog**</span>

With this tool, the user can create an Obspy [<span style="color:#5DADE2;">catalog</span>](https://docs.obspy.org/packages/autogen/obspy.core.event.Catalog.html#obspy.core.event.Catalog) merging the information from the location, source spectrum and moment tensor inversion output. Finally, the user can filer the catalog by time spam or geographically. We have added a method to write in human language the catalog. See an example:

    Event 46: Date 01/02/2022 02:03:00.598999 rms 0.47 s Lat 42.5250 Lon 1.4252 Depth -2.0 km +- 1.3 min_dist 0.089 max_dist 0.621 smin 0.6 km smax 0.7 km ell_azimuth 161.3 gap 65.3 conf_lev 90.0 %
    Magnitudes: Mw 3.58 +- 0.11
    Moment Tensor Solution:
    Mw 3.77 Mo 5.07e+14 Nm DC 44.64 % CLVD 22.70 % iso 32.65 % variance_red 34.54
    Nodal Plane: Strike 319.9 Dip 40.0 Rake -74.7
    Moment Tensor: mrr -1.05e+14 mtt 2.64e+14 mpp 3.38e+14 mrp 1.09e+13 mrt -7.33e+13 mrp -7.33e+13
    station phase polarity date time time_residual distance_degrees distance_km azimuth takeoff_angle
    PAND P ? 01/02/2022 02:03:01.900000 -0.32 0.1 9.9 90.8 89.4
    PAND S ? 01/02/2022 02:03:03.330000 0.00 0.1 9.9 90.8 89.4
    ARBS P ? 01/02/2022 02:03:02.340000 -0.48 0.1 13.6 137.9 90.8
    ARBS S ? 01/02/2022 02:03:04.290000 -0.04 0.1 13.6 137.9 90.8
    CEST P ? 01/02/2022 02:03:03.200000 -0.08 0.1 16.3 300.1 87.5
    CEST S ? 01/02/2022 02:03:05.210000 0.10 0.1 16.3 300.1 87.5
    CSOR P ? 01/02/2022 02:03:05.170000 -0.19 0.3 29.1 234.8 87.9
    CSOR S ? 01/02/2022 02:03:09.050000 0.44 0.3 29.1 234.8 87.9
    SALF S ? 01/02/2022 02:03:09.580000 -0.39 0.3 32.7 322.8 87.4
    CORG P ? 01/02/2022 02:03:05.770000 -0.40 0.3 34.0 194.3 87.1
    CORG S ? 01/02/2022 02:03:10.930000 0.97 0.3 34.0 194.3 87.1
    GENF P ? 01/02/2022 02:03:06.150000 -0.21 0.3 35.1 19.0 87.6
    GENF S ? 01/02/2022 02:03:10.530000 0.26 0.3 35.1 19.0 87.6
    VALC P ? 01/02/2022 02:03:08.590000 -0.70 0.5 53.0 106.7 89.4
    CARF P ? 01/02/2022 02:03:09.900000 -0.38 0.5 59.1 69.0 88.7
    CARF S ? 01/02/2022 02:03:17.300000 0.43 0.5 59.1 69.0 88.7
    FNEB P ? 01/02/2022 02:03:11.830000 -0.09 0.6 69.1 52.5 88.3
    FNEB S ? 01/02/2022 02:03:20.730000 1.11 0.6 69.1 52.5 88.3

### <span style="color:darkcyan;">**Manage Catalog from CLI**</span>
#### Usage
```bash
>> surfquake buildcatalog -e [path_event_files_folder] -s [path_source_summary_file] -m [path_mti_summary_file] -t [type_of_catalog] -o [path_to_ouput_folder]
```
#### Interactive help

```bash
>> surfquake buildcatalog -h
``` 

#### Run Create Metadatafrom CLI
```bash
>> surfquake buildcatalog -e ./locations -s source_summary.txt -m mti_source_summary.txt -t "QUAKEXML" -o ./catalog 
```

### <span style="color:darkcyan;">**Manage Catalog from Library**</span>

#### Classes
[`BuildCatalog`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
```python
class BuildCatalog:
    def __init__(self, loc_folder, output_path, format="QUAKEML", source_summary_file=None, mti_summary_file=None):

        """
        BuildCatalog class helps to join information from all surfquake outputs and create a catalog

        Attributes:
        - loc_folder (str): Path to the folder where the user have the locations files *hyp
        - output_path (str): Output folder path where catalog object and file will be saved
        - format (str): https://docs.obspy.org/packages/autogen/obspy.core.event.Catalog.write.html
        - source_summary_file (str): Path to the output file from source module
        - mti_summary_file (str): Path to the output file from mti module

        Methods:
        - __init__(root_path): Initialize a new instance of BuildCatalog.
        - __merge_info(catalog: Catalog): Merges the information from loc, source and mti
        - build_catalog(): Starts the process to create the catalog from loc files, then calls __merge_info
        """
```

[`WriteCatalog`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
```python
class WriteCatalog:
    def __init__(self, path_catalog):

        """
        WriteCatalog class helps to filter the catalog obj write in the most Readable

        Attributes:
        - path_catalog (str): Path to the pickle file saved when run build_catalog_loc from BuildCatalog class

        Methods:
        - __init__(root_path): Initialize a new instance of BuildCatalog.
        - filter_time_catalog(verbose=True, **kwargs): filter the catalog in time span
        - filter_geographic_catalog(verbose=True, **kwargs): filter the catalog in geographic contrain and magnitude
        - write_catalog(catalog: Catalog, format, output_path)
        - write_catalog_surf(catalog: Union[Catalog, None], output_path)
        """

        self.path_catalog = path_catalog
        self.catalog = []
        self.__test_catalog()
```

#### Methods
[`build_catalog_loc`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
Instance method from BuildCatalog Class
```python
def build_catalog_loc(self):
```

[`filter_time_catalog`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
Instance method from WriteCatalog Class
```python
    def filter_time_catalog(self, verbose=True, **kwargs):

        """
        Filter the catalog readed in the class instantiation
        verbose: bool:
        **kwargs
        starttime :str: starttime to filter the catalog in format %d/%m/%Y, %H:%M:%S.%f
        endtime :str: endtime to filter the catalog in format %d/%m/%Y, %H:%M:%S.%f
        example:
        wc = WriteCatalog(catalog_path)
        catalog_filtered = wc.filter_time_catalog(starttime="30/01/2022, 00:00:00.0",
        endtime="20/02/2022, 00:00:00.0")
        return :catalog obj:
        """
```

[`filter_geographic_catalog`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
Instance method from WriteCatalog Class
```python
    def filter_geographic_catalog(self, catalog: Union[Catalog, None], verbose= True, **kwargs):

        """
        Filter the catalog readed in the class instantiation or the catalog provided in bu the user when the method
        is called
        verbose: bool:
        catalog obj (optional), if not found it uses catalog from the catalog attribute
        **kwargs --> All keys must be used to filter success.

        lat_min:float
        lat_max:float
        lon_min:float
        lon_max:float
        depth_min:float: km
        depth_max:float: km
        mag_min:float
        mag_max:float

        return :catalog obj:
        """
```

[`write_catalog_surf`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/utils/manage_catalog.py)
Instance method from WriteCatalog Class
```python
    def write_catalog_surf(self, catalog: Union[Catalog, None], output_path):

        """
        Writes in human language the catalog instantiated with the class
        verbose: bool:
        catalog obj (optional), if not found it uses catalog from the catalog attribute
        """

```
### Examples using library

First, We can join all infrmation from located events in your loc folder, with the information of magnitudes and source parameters plus the ourput from Moment Tensor.

```python linenums="1"
from surfquakecore.utils.manage_catalog import BuildCatalog
path_events_folder = "/Volumes/LaCie/surfquake_test/test_nll_loc"
path_source_file = "/Volumes/LaCie/surfquake_test/catalog_output/sources.txt"
output_path = "/Volumes/LaCie/surfquake_test/catalog_output"
bc = BuildCatalog(loc_folder=path_events_file, source_summary_file=path_source_file, output_path=output_path,
                      format="QUAKEML")
bc.build_catalog_loc()

```

Secondly, We can play with the catalog, filtering it (filter_time_catalog and filter_geographic_catalog) or writing a human readable version using write_catalog_surf method.

```python linenums="1"
from surfquakecore.utils.manage_catalog import WriteCatalog

catalog_path = "/Volumes/LaCie/all_andorra/catalog/catalog_obj.pkl"
output_path = "/Volumes/LaCie/all_andorra/catalog/catalog_surf.txt"
wc = WriteCatalog(catalog_path)
print(wc.show_help())
help(wc.filter_time_catalog)
help(wc.filter_geographic_catalog)
catalog_filtered = wc.filter_time_catalog(starttime="30/01/2022, 00:00:00.0", endtime="20/02/2022, 00:00:00.0")

catalog_filtered = wc.filter_geographic_catalog(catalog_filtered, lat_min=42.1, lat_max=43.0, lon_min=0.8, lon_max=1.5,
                                                depth_min=-10, depth_max=20, mag_min=3.4, mag_max=3.9)

wc.write_catalog_surf(catalog=None, output_path=output_path)
#wc.write_catalog_surf(catalog=catalog_filtered, output_path=output_path)

# Now you can also save the filtered catalog or even plot the catalog using common Obspy Methods
#catalog_filtered.plot(projection='local')

```


## <span style="color:darkcyan;">**Build MTI config files**</span>

surfQuake core library incorporates and easy way to generate mti Config Files
The only thing you need is a previously crated catalog and a mti.ini files that serves of template (just copy and paste the example in the explanation of config file from MTI). The routine will create the mti.ini files using the information from the event catalogs (date, latitude, lngitude, depth and magnitude) and from the rest of the fields from the template. This action will write a mti.ini file per event and ready to be used by MTI toolbox.

### <span style="color:darkcyan;">**Build MTI config files from CLI**</span>
#### Usage
```bash
>> surfquake buildmticonfig -c [catalog_file_path] -t [mti_config_template] -o [output_folder] -s [if starttime] -e [if endtime] -l [if lat_min] -a [ if lat_max] -d [if lon_min] -k [if lon_max] -w [if depth_min] -f [depth_max] -g [if mag_min] -p [if mag_max]
```
#### Interactive help

```bash
>> surfquake buildmticonfig -h
``` 

#### Run Build MTI config files from CLI
```bash
>> surfquake buildmticonfig --catalog_file_path /mti_config_test/catalog_obj.pkl --mti_config_template /mti_confis_test/template.ini --output_folder /mti_confis_test --starttime "30/09/2021, 00:00:00.0" --endtime "30/09/2022, 00:00:00.0" --lat_min 38.0 --lat_max 44.0 --lon_min -2.0 --lon_max 4.0 --depth_min -3.0 --depth_max 50 --mag_min 3.0 --mag_max 4.0
```

### <span style="color:darkcyan;">**Build MTI config files from Library**</span>
#### Classes
[`BuildMTIConfigs`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/moment_tensor/mti_parse.py)
```python
class BuildMTIConfigs:
    def __init__(self, catalog_file_path, mti_config: Union[str, MomentTensorInversionConfig], output_path):

        self.catalog_file_path = catalog_file_path
        self.config_mti_template = mti_config
        self.output_path = output_path
        self.catalog = None

        if isinstance(mti_config, str) and os.path.isfile(mti_config):
            self.mti_template_configuration = (load_mti_configuration(mti_config),)

        else:
            raise ValueError(f"mti_config {mti_config} is not valid. It must be a valid .ini file for "
                             f"MomentTensorInversionConfig")
```

#### Methods
[`write_mti_ini_file`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/moment_tensor/mti_parse.py)
```python
    def write_mti_ini_file(self, **kwargs):

        """
        starttime :str: starttime to filter the catalog in format %d/%m/%Y, %H:%M:%S.%f
        endtime :str: endtime to filter the catalog in format %d/%m/%Y, %H:%M:%S.%f
        lat_min:float
        lat_max:float
        lon_min:float
        lon_max:float
        depth_min:float: km
        depth_max:float: km
        mag_min:float
        mag_max:float
        """
```

### Examples using library
```python linenums="1"
from surfquakecore.moment_tensor.mti_parse import BuildMTIConfigs

catalog_path = "/Users/admin/Desktop/all_andorra/catalog/catalog_obj.pkl"
output_path = "/Users/admin/Desktop/all_andorra/mti_configs_created"
mti_template_path = '/Users/admin/Desktop/all_andorra/mti_configs/mti_template.ini'

bmc = BuildMTIConfigs(catalog_file_path=catalog_path, mti_config=mti_template_path, output_path=output_path)
bmc.write_mti_ini_file(starttime="01/01/2021, 00:00:00.0", endtime="30/10/2022, 00:00:00.0", lat_min=40.0,
                       lat_max=44.0, lon_min=0.0, lon_max=4.2, depth_min=-10, depth_max=60,
                       mag_min=3.0, mag_max=4.0)
```