# <span style="color:darkcyan;">**Ambient Noise Processing**</span>

Ambient Noise Processing is a passive seismic technique based on the cross-correlation of continuous background seismic noise between station pairs. SurfQuake provides a complete workflow including:

1. Building a waveform dictionary from traces files
2. Creating frequency-domain spectral matrices
3. Cross-correlating and stacking station pairs
4. Optional rotation of horizontal components

The ANT module is designed for large-scale continuous seismic datasets and supports parallel processing, spectral whitening, time normalization, instrument response removal, and multiple stacking strategies.

```
References



Ambient Noise Processing workflow:

Bensen, G. D., et al. Processing seismic ambient noise data to obtain reliable broad-band surface wave dispersion measurements. Geophysical journal international, 2007, vol. 169, no 3, p. 1239-1260.

Special signal processing. Phase Wave Stack and phase Cross-correlation

Schimmel, M. Phase cross-correlations: Design, comparisons, and applications. Bulletin of the Seismological Society of America, 1999, vol. 89, no 5, p. 1366-1378.

Clock Synchronization:

Cabieces, R., et al. Clock drift corrections for large aperture ocean bottom seismometer arrays: application to the UPFLOW array in the mid-Atlantic Ocean, Geophysical Journal International, Volume 239, Issue 3, December 2024, Pages 1709–1728.

Fast Horizontal Components Rotation:

Lin F.-C., Moschetti M.P., Ritzwoller M.H., 2008. Surface wave tomography of the western United States from ambient seismic noise: Rayleigh and Love wave phase velocity maps, Geophys. J. Int., 173, 281–298. doi: 10.1111/j.1365-246X.2008.03720.x 10.1111/j.1365-246X.2008.03720.x
```

---

# <span style="color:darkcyan;">**ANT Create Dictionary CLI**</span>

This command scans a MiniSEED archive recursively and builds a dictionary-based project file containing waveform paths organized by network, station and channel.

The generated pickle file is later used as input for the ANT processing pipeline.

```     
        Key Arguments:
            -d,  --data_path         [REQUIRED] Root directory containing MiniSEED files
            -i,  --inventory_file    [REQUIRED] Path to StationXML inventory file
            -s,  --save_file         [REQUIRED] Path to save the ANT dictionary pickle
            -nt, --net               [OPTIONAL] Network filter, wildcards allowed (default: *)
            -st, --station           [OPTIONAL] Station filter, wildcards allowed (default: *)
            -ch, --channel           [OPTIONAL] Channel filter, wildcards allowed (default: *)
            -w,  --workers           [OPTIONAL] Number of parallel workers
```

## Usage

```bash
>> surfquake ant_create_dict -d [data_path] -i [inventory_file] -s [save_file]
```

## Interactive help

```bash
>> surfquake ant_create_dict -h
```

## Examples CLI

```bash
# Process all stations and channels
>> surfquake ant_create_dict -d ./mseed -i ./meta/inventory.xml -s ./output/data_dict.pkl

# Process only BHZ channels for network II
>> surfquake ant_create_dict -d ./mseed -i ./meta/inventory.xml \
-s ./output/data_dict.pkl -nt II -ch BHZ

# Use wildcard filters and multiple workers
>> surfquake ant_create_dict -d ./mseed -i ./meta/inventory.xml \
-s ./output/data_dict.pkl -nt "IU" "II" -ch "BH*" -w 8
```

---

# <span style="color:darkcyan;">**ANT Process Matrix CLI**</span>

This command processes the waveform dictionary created by `ant_create_dict` and generates frequency-domain spectral matrices for each station/channel.

The workflow includes:

1. Reading continuous waveform files
2. Gap checking and merging
3. Optional instrument response removal
4. Optional decimation
5. Time-domain normalization
6. Spectral whitening
7. FFT transformation into frequency-domain matrices

Results are stored as pickle files and later used by the cross-correlation stage.

## Usage

```bash
>> surfquake ant_process_matrix -c [config.json]
```

## Generate template configuration

```bash
>> surfquake ant_process_matrix --generate config_process_matrix.json
```

## Interactive help

```bash
>> surfquake ant_process_matrix -h
```

## Example configuration

```json
{
    "project_file": "./output/data_dict.pkl",
    "inventory_file": "./meta/inventory.xml",
    "output_path": "./output/matrices",
    "processing_window": 900,
    "remove_response": true,
    "units": "VEL",
    "waterlevel": 90,
    "decimate": true,
    "factor": 5,
    "time_norm": true,
    "method": "running avarage",
    "timewindow": 25,
    "whiten": true,
    "freqbandwidth": 0.02,
    "prefilter": true,
    "filter_freqmin": 0.01,
    "filter_freqmax": 0.4,
    "filter_corners": 4
}
```

---
*processing window*: (int) Noise time window in seconds. It is important to keep a long window to be able to capture the longest period of the EGFs. Be into account your minimum interstation distance to correctly select this parameter.

*units*: (str: DISP, ACC and VEL are available). The instrument response is removed and seismograms converted the physical unit output. This is important when you are mixing different types of instruments, because EGFs can have phase shifts.

*waterlevel* (int). Basically this parameter controls the deconvolution in the intrument response removal. Set to 90 is safe!

*decimate*: (bool). It is recommended to decimate your waveforms due to the later RAM memory usage and because computational efficiency.

*factor*: (int). Target sampling rate in Hz after decimation, if decimate is set to True.

*time_norm*: Apply time normalisation (see method). This is necassary in many cases due to seismogram noise use to be contaminated of transient signals.

*method*: (str: running avarage, 1 Bit and PCC). Time normalization methods or special phase cross correlation. If PCC is applied then whitening is skipped (because it is not totally necessary to equilibrate noise frequency band power).

*whiten*: (str). Apply spectral whitening using a moving avarage in the spectral domain.

*prefilter*: (bool). Apply a zero-phase bandpass pre-filter before noise cross correlation. recommended.

*filter_freqmin*: (float). Min frequency of pre-filter.

*filter_freqmax*: (float). Max frequency of pre-filter.

*filter_corners*: (int). Number of poles of the pre-filter.
---

## Examples CLI

```bash
# Generate template config
>> surfquake ant_process_matrix --generate config_process_matrix.json

# Run ANT spectral matrix processing
>> surfquake ant_process_matrix -c config_process_matrix.json
```

---

# <span style="color:darkcyan;">**ANT Cross Stack CLI**</span>

This command reads the spectral matrices generated by `ant_process_matrix` and computes cross-correlations between station pairs.

The processing workflow includes:

1. Identification of common days between stations
2. Frequency-domain cross-correlation
3. Inverse FFT transformation
4. Daily and full stacking
5. Optional horizontal rotation

The resulting cross-correlations are stored in HDF5 format.

## Usage

```bash
>> surfquake ant_cross_stack -c [config.json]
```

## Generate template configuration

```bash
>> surfquake ant_cross_stack --generate config_cross_stack.json
```

## Interactive help

```bash
>> surfquake ant_cross_stack -h
```

## Example configuration

```json
{
    "input_path": "./output/matrices",
    "output_path": "./output/stacks",
    "channels": ["Z"],
    "stations": [],
    "stack": "Linear",
    "power": 2.0,
    "autocorr": false,
    "min_distance": 1000.0,
    "daily_stacks": false,
    "overlap": 50.0,
    "workers": null,
    "rotate": false,
    "rotate_daily": false
}
```

---
*input_path* (str), path to the folder where the storaged matrix files from the pre-processing steps are saved.

*output_path* (str), path to the folder where stack files will be saved.

*channels:* (list), Channels list to effectuate the cross correlation and stacking. Example ["N", "E", "Z"].

*stations:* (list), Stations list to effectuate the cross correlation and stacking. Example ["STA1", "STA2"]. If empty ALL stations will be processed.

*stack*: Stacking methods: Linear, PWS, nrooth.

*power*: Exponent for PWS or nrooth stacking.

*autocorr*: if user want to compute autocorrelations. This is mandatory if user wants to get Radial (RR) and  transversal (TT) EGFs.

*min_distance*: Threshold interstation distance to compute the cross and stacking.

*daily_stacks*: If you want to also compute partial stacks. Very useful for Synchronize clocks. (20 days stack).

*overlap*: Shift in time daily stacks. Example, if set to 50.0 the shift will be 0.5*20, then every 10 days.

*workers*: Number of parallel worker processes (default: null = cpu_count - 1).

*rotate*: Run ZNE rotation after stacking. User will obtain RR and TT components, used for Rayleigh and Love waves.

*rotate_daily*: If user wants to rotate daily stacks.

---


## Examples CLI

```bash
# Generate template config
>> surfquake ant_cross_stack --generate config_cross_stack.json

# Run cross-correlation and stacking
>> surfquake ant_cross_stack -c config_cross_stack.json
```

## Plotting EGFs

It is possible now to plot the EGFs with command *quick*. We show how to plot it as record section.
Moreover ISP can also plot the output EGFs using the [ANT module](https://projectisp.github.io/ISP_tutorial.github.io/ant/).

```bash
# Plotting EGFs in a record section
>> surfquake quick -w "./output/stacks/*ZZ*" --plot_config "./plot_conf.yaml"
```
It is mportant to add the option record_option: egf, so that plot the EGFs as a record section

```yaml
plotting:
  traces_per_fig: 3
  sort_by: distance
  vspace: 0.05
  show_legend: true
  title_fontsize: 9
  plot_type: record
  record_option: egf
  sharey: False
  show_arrivals: False
  pick_output_file: ./picks.csv
```