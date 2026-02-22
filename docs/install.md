# <span style="color:darkcyan;">**Installation**</span>

## <span style="color:darkcyan;"> **Processor and OS** </span>

**Linux OS** and **Mac** computers with Intel processors can now run surQuake thanks to testing and programming. 

Additionally, [<span style="color:#5DADE2">*Rosetta*</span>](https://support.apple.com/en-us/102527#:~:text=Rosetta%20is%20not%20an%20app,it%20like%20you%20normally%20would.) allows Mac machines with Apple chips to run applications meant for Mac systems with Intel processors. If this is your case, please install Rosseta before proceed with surfQuake intallation

The user can select whether to install only the surfQuake core or the GUI (which also includes the core). 

- **Automatic installation**: This is only available in surfQuake GUI and requires a previous installation of [<span style="color:#5DADE2">*Anaconda*</span>](https://www.anaconda.com/download).

- **Manual Installation**: In both circumstances (GUI & Core), SurfQuake must be manually installed from the pip repository after creating an environment (anaconda env or pip env).


## <span style="color:darkcyan;"> **Installing the GUI** </span>

### Automatic Installation:

This process will create atomatically an anaconda environment and then will install surfquake.

Open a terminal and type:
 
```bash
>> (base) git clone https://github.com/rcabdia/SurfQuake.git
>> (base) cd SurfQuake/install
>> (base) chmod u+x surfquake_installer.sh
>> (base) ./surfquake_installer.sh
```

After this process, if everything is ok and the alias has been correctly included in your path system, just type in your terminal

```bash
>> surfquake
```
if you have problems because surfquake is not properly recognized by your system path, go to the root path where you have saved surfquake,

```bash
>> (base) cd SurfQuake
>> (base) conda activate surfquake
>> (surfquake) python start_surfquake.py
```

Remenber that installing surfquake GUI also includes the core Library and the Command Line interface. That's means that now you have access to the commands
```bash
>> (surfquake) surfquake -h
```

and you can make python scripts calling classes and methods from surfquake core.


### Manual Installation:
#### Create an environment

##### Anaconda
```bash
>> (base) conda create -n surfquake python=3.9
>> (base) conda activate surfquake
```

##### pip

Warning, if you are using anaconda, normally is automatically activated the base environment in your terminal. So first of all deactivate it!

```bash
>> (base) conda deactivate
```

```bash
>> pip install virtualenv
>> python<version> -m venv surfquake
>> source surfquake/bin/activate # activate your environment
```

#### Execute the Installation

Now, you can proceed with the installation of the full surfQuake program:

```bash
>> (surfquake) cd SurfQuake
>> (surfquake) pip install -r requirements.txt
```

to start the program

```bash
>> (surfquake) cd SurfQuake
>> (surfquake) python start_surfquake.py
```


## <span style="color:darkcyan;"> **Installing the core** </span>


### Create an environment

#### Anaconda env
```bash
>> conda create -n surfquake python=3.9 # surfQuake for 3.9 <= Python <= 3.11
>> conda activate surfquake
```

#### pip env

Warning, if you are using anaconda, normally is automatically activated the base environment in your terminal. So first of all deactivate it! 

```bash
>> (base) conda deactivate
```

To prevent dependencies from becoming incompatible, please make sure that the version of Python you have installed on your system is greater than 3.9 and less than 3.12. Simply follow these steps to view your Python version:

```bash
>> python
Python 3.11.4 (main, Jul  5 2023, 09:00:44) [Clang 14.0.6 ] on darwin
Type "help", "copyright", "credits" or "license" for more information
```


```bash
>> pip install virtualenv # install in your python the virtualenv package (just in case)
>> python3 -m venv ./surfquake # python<version> -m <virtual-environment-name> <venv_location> 
>> source surfquake/bin/activate #s ource <venv_location>
```

### Install the core package
You can install surfQuake core after creating your virtual environment (using either Anaconda env or Pip env). After activate your environment,

```bash
>> (surfquake) pip install surfquake
```
