# <span style="color:darkcyan;">**Installation**</span>

## Create an environment

### Anaconda env
```bash
>> conda create -n surfquake python=3.9 # surfQuake for 3.9 <= Python <= 3.11
>> conda activate surfquake
```

### pip env

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

## Install the core package
You can install surfQuake core after creating your virtual environment (using either Anaconda env or Pip env). After activate your environment,

```bash
>> (surfquake) pip install surfquake
```

## <span style="color:darkcyan;"> **Notes about Processor and OS** </span>

**Linux OS** and **Mac** computers with Intel processors can now run surQuake thanks to testing and programming. 

Additionally, [<span style="color:#5DADE2">*Rosetta*</span>](https://support.apple.com/en-us/102527#:~:text=Rosetta%20is%20not%20an%20app,it%20like%20you%20normally%20would.) allows Mac machines with Apple chips to run applications meant for Mac systems with Intel processors. If this is your case, please install Rosseta before proceed with surfQuake intallation
