
# <span style="color:darkcyan;">**Source Parameters**</span>

The tool Source Parameters is designed to estimate events source parameters (i.e., `Mw`, `ML`, `seismic moment`, `corner frequency`, `radiated energy`, `source size` and  stress drop`) and the attenuation parameters (t-star, quality factor). 

!!! info Reference
    It is based in the implementation of [<span style="color:#5DADE2;">Satrinano et al., 2023</span>](https://sourcespec.readthedocs.io/en/stable/index.html). A detailed theoretical background can be found [here](https://sourcespec.readthedocs.io/en/stable/theoretical_background.html#overview)

## <span style="color:darkcyan;">**Source Parameters from CLI**</span>

```
    Key Arguments:
        -i, --inventory_file     [REQUIRED] Path to station metadata (XML, RESP)
        -p, --project_file       [REQUIRED] Path to project file
        -c, --config_file        [REQUIRED] Path to configuration for SourceSpec
        -l, --hypocenter_dir     [REQUIRED] Directory containing NonLinLoc *.hyp files
        -o, --output_folder      [REQUIRED] Output folder for source parameter results
        -t, --large_scale        [OPTIONAL] Automatic long cut of teleseism event waveforms
```

### Usage

```bash
### Interactive help
>> surfquake source -h
``` 

```bash
>> surfquake source -i ./metadata/inv_all.xml -p ./project/surfquake_project.pkl -c ./config_files/source_spec.conf -l ./loc -o ./source_output
```



## <span style="color:darkcyan;">**Config file**</span>
The configuration file corresponds to SourceSpec 1.7. This template can be directly used as template.

- Generate a config file from terminal:

```bash
>> surfquake make_config -h
>> surfquake make_config -c sspec_config -o ./configs

``` 

## Source Parameters from Library


### Classes
[`Automag`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/magnitudes/run_magnitudes.py)

```python
class Automag:

    def __init__(self, project: SurfProject, locations_directory: str, inventory_path, source_config: str,
                 output_directory: str, scale: str, gui_mod=None):
        
        """
        Manage SourceSpec program to estimate source parameters.
        :param project: SurfProject object.
        :param inventory_path: Path to metadata file.
        :param source_config: Path to source config file.
        :param output_directory: Path to output folder.
        :param scale: if regional waveforms will cut with small adapted time windows, else will be cut with a 
        long time window
        """
```

[`ReadSource`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/magnitudes/source_tools.py)
```python
class ReadSource:
    def __init__(self, root_path_to_output: str):
        """
        The class methods are designed to scan the output of sourcespec
        root_path_to_output: Root path where sourcespec output is expected
        """
        self.root_path_to_output = root_path_to_output
        self.obsfiles = []

```

### Methods
[`estimate_source_parameters`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/magnitudes/run_magnitudes.py)
```python
# Automag instance method
def estimate_source_parameters(self):
    # Loop over loc folder files and run source parameters estimation
```

[`generate_source_summary`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/magnitudes/source_tools.py)
```python
# ReadSource instance method
def generate_source_summary(self):
    
    """
    # Generate source parameters summary as dataframe
    :return List: List of dictionaries containing source parameters
    """
```

[`write_summary`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/magnitudes/source_tools.py)
```python
# ReadSource instance method
def write_summary(self, summary: list, summary_path: str):
    """
    Transform the summary into txt file using Pandas Dataframe
    :param summary: List of dictionaries containing source parameters
    :param summary_path: path to file output including the name of the file.
    """
```

### Example using library
```python linenums="1"
from surfquakecore.magnitudes.run_magnitudes import Automag
from surfquakecore.magnitudes.source_tools import ReadSource
from surfquakecore.project.surf_project import SurfProject
import os

if __name__ == "__main__":

    cwd = os.path.dirname(__file__)
    ## Project definition ##
    path_to_project = "/test_surfquake_core/project"
    project_path_file = os.path.join(path_to_project, "surfquake_project.pkl")
    print("project:", project_path_file)

    # load the project
    sp_loaded = SurfProject.load_project(path_to_project_file=project_path_file)
    print(sp_loaded)

    # Basic input: working_directory, inventory file path and config_file input
    working_directory = os.path.join(cwd, "source_estimations")
    inventory_path = os.path.join(working_directory, "inventories", "inv_surfquakecore.xml")
    path_to_configfiles = os.path.join(working_directory, "config/source_spec.conf")
    locations_directory = os.path.join(working_directory, "locations")
    output_directory = os.path.join(working_directory, "output")
    summary_path = '/Users/roberto/Documents/SurfQuakeCore/examples/source_estimations/source_summary.txt'

    # Running stage
    mg = Automag(sp_loaded, locations_directory, inventory_path, path_to_configfiles, output_directory, "regional")
    mg.estimate_source_parameters()

    # Now we can read the output and even write a txt summarizing the results
    rs = ReadSource(output_directory)
    summary = rs.generate_source_summary()
    rs.write_summary(summary, summary_path)
```