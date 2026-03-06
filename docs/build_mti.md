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