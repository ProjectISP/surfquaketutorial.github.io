# <span style="color:darkcyan;">**Create your project**</span>

In surfQuake a project is simply a python object that can store the path to valid seismogram files plus the associated metadata in its attributes . This strategy allows to proceed with fast filters or join different projects. Project is the necessary input for the toolboxes Picker, Associator, Event Locator, Seismic Source and MTI.

Here we will explain how the user can manage a project and be ready to proceed with the rest of toolboxes.


## <span style="color:darkcyan;"> **Project from CLI** </span>

### Overview
This command allows you to create a seismic project, which is essentially a dictionary
storing the paths to seismogram files along with their corresponding metadata.
    
### Usage

    Key Arguments:
        -d, --data           [REQUIRED]         Path to waveform data directory (or file pattern)
        -s, --save_path      [REQUIRED]         Directory to save the project file
        -n, --name           [REQUIRED]         Name of the project (e.g., "my_experiment")
        -v, --verbose        [OPTIONAL]         Print detailed file discovery and indexing logs

```bash
# Interactive help
>> surfquake project -h
``` 

```bash
>> surfquake project -d "./testing_data" -s "./Volumes/LaCie/test_surfquake_core/testing_data/projects" -n "/surfquake_project_new.pkl" --verbose
```


## <span style="color:darkcyan;"> **Project from library** </span>

### Classes
In this section, we will explain the class SurfProject and we will explain how to manage your project from a simple example:

[`SurfProject`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)


```python 
class SurfProject:

    def __init__(self, root_path: Union[str, List[str]]):

        """

        SurfProject class is designed to be able to storage the path to seismograms
        files plus the file metadata information (i.e. sampling_rate, starttime...)

        Attributes:
        - root_path (str): The root path to the folder where the user have the data files.

        Methods:
        - __init__(root_path): Initialize a new instance of MyClass.
        - load_project(path_to_project_file: str): Load a project from a file storage in hard-drive
        - save_project(path_file_to_storage: str): Saves a project as a pickle file in hard-drive
        - search_files(verbose=True, **kwargs): Create a project. It can be used filters by nets,
        stations, channels selection and/or filter by timestamp
        - filter_project_keys(**kwargs): Filter a project (once is crated) using regular expressions.
        """


```

### Methods

[`search_files`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)

```python


    def search_files(self, format="NONE", verbose=True, **kwargs):

        """
        Args:

        - verbose (bool): Description of arg1.
        - nets (str): String with the name of selected nets to be filtered (i.e., "WM,ES")
        - stations (str): String with the name of selected stations to be filtered (i.e., "ARNO,UCM,EMAL")
        - channels (str): String with the name of selected channels to be filtered (i.e., "HHN,HHZ,HHE")
        - starttime (str "%Y-%m-%d %H:%M:%S" ): String with the reference starttime, upper time spam threshold
        (i.e.,"2023-12-10 00:00:00")
        - endtime (str "%Y-%m-%d %H:%M:%S" ): String with the reference endtime, lower time spam threshold
        (i.e.,"2023-12-23 00:00:00")

        Returns:
        - type: Description of the return value.
        """
```

[`filter_project_keys`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)

```python
    def filter_project_keys(self, **kwargs):

        """
        Args:
        - net (str): String with the name of selected nets to be filtered (i.e., ".")
        - station (str): String with the name of selected stations to be filtered (i.e., "ARNO|UCM|EMAL")
        - channel (str): String with the name of selected channels to be filtered (i.e., "HH.")
        """

```

[`filter_project_time`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)

```python

    def filter_project_time(self, starttime: str, endtime: str):

        """
        - starttime (str, "%Y-%m-%d %H:%M:%S"): String with the reference starttime, upper time spam threshold
        (i.e., "2023-12-10 00:00:00")

        - endtime (str, "%Y-%m-%d %H:%M:%S" ): String with the reference endtime, lower time spam threshold
        (i.e., "2023-12-23 00:00:00")

        """
```

[`save_project`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)

```python
def save_project(self, path_file_to_storage: str)->bool
# Saves the project object as a pickle file.
```

[`load_project`](https://github.com/rcabdia/SurfQuakeCore/blob/main/surfquakecore/project/surf_project.py)

```python
def load_project(path_to_project_file: str):
```

### Attibutes

```python
project :Dict
data_files :List
```

Next, the example of using this class and its methods. This example script is available in SurfQuakeCore/examples/manage_project_new.py

```python linenums="1"
from multiprocessing import freeze_support
from surfquakecore.project.surf_project import SurfProject
import time

path_to_data = "/Volumes/LaCie/test_surfquake_core/testing_data"
path_to_project = "/Volumes/LaCie/test_surfquake_core/testing_data/projects/surfquake_project_new.pkl"

if __name__ == '__main__':

    freeze_support()
    sp = SurfProject(path_to_data)
    #sp.search_files(starttime="2022-01-30 23:55:00", endtime="2022-02-01 00:30:00", stations="SALF,VALC", channels="HHZ")
    sp.search_files(verbose=False)
    #sp_original_project = copy.copy()
    sp.filter_project_keys(station="SALF|VALC|CEST")
    sp_original1 = sp.copy()
    sp_original1.filter_project_keys(station="SALF")
    sp_original2 = sp.copy()
    sp_original2.filter_project_keys(station="VALC")

    sp_join = sp_original1 + sp_original2
    print("With no filter")
    print(sp_join)
    sp_join.filter_project_time(starttime="2022-01-30 23:55:00", endtime="2022-02-01 00:30:00")
    print("With filter")
    print(sp_join)
    sp_join.save_project(path_file_to_storage=path_to_project)
    time.sleep(5)
    sp_loaded = SurfProject.load_project(path_to_project_file=path_to_project)
    print(sp_loaded)
```

The first step is create the object from the class SurfProject 

```python
sp = SurfProject
```

The necessary input to create the sp object is the root path where you have storage the seismogram files. Then, you can proceed to apply the method "search_files". This method includes the possibility of filter the inclusion of files inside the project.

```python
sp.search_files(starttime="2022-01-30 23:55:00", endtime="2022-02-01 00:30:00", stations="SALF,VALC", channels="HHZ")
print(sp) # To see the contain of the project
```

Additionally, once the project has been created, you can also filter it by using [´regular expressions´](https://docs.python.org/3/library/re.html) net, station, channel using:

Note: some util Regex info at [Wiki Regex](https://en.wikipedia.org/wiki/Regular_expression) and [Python keywords](https://www.educative.io/answers/how-to-implement-wildcards-in-python)
```python
sp.filter_project_keys(station="SALF|VALC|CEST")
```

or filterintg the time spam using:

```python
sp_join.ilter_project_time(starttime="2022-01-30 23:55:00", endtime="2022-02-01 00:30:00")
```

Adding projects using "+" symbol

```python
sp_original1 = sp.copy()
sp_original1.filter_project_keys(station="SALF")
sp_original2 = sp.copy()
sp_original2.filter_project_keys(station="VALC")

sp_join = sp_original1 + sp_original2

```

Finally, you can save the project by,
```python
sp_join.save_project(path_file_to_storage=path_to_project)
```
and loading it

```python
sp_loaded = SurfProject.load_project(path_to_project_file=path_to_project)
```
