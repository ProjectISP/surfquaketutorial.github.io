# Bash Script Interaction with SurfQuake

SurfQuake provides a command-line interface (CLI) that integrates seamlessly with bash scripting. This allows users to automate large-scale waveform processing, post-analysis scripting, and plotting tasks efficiently.

## Example: Bash Workflow for Seismic Event Processing

Below is a sample `bash` script that chains together two major SurfQuake commands: `quick` and `processing`.

```bash
#!/bin/bash

# Activate your conda or virtual environment
source activate surfquake  # Replace with your environment manager if needed (e.g., 'conda activate surfquake')

# Step 1: Quickly process individual waveform files with a custom post-processing script
surfquake quick \
  -w "/Users/admin/Desktop/surf_test/nuclear/cut_event/*" \
  --post_script /Users/admin/Desktop/surf_test/scripts/my_script.py \
  -c /Users/admin/Desktop/surf_test/config/config.yaml \
  --plot_config /Users/admin/Desktop/surf_test/config/plot.yaml \
  -o /Users/admin/Desktop/surf_test/output \
  -a 

# Step 2: Process a set of previously detected or catalogued events
surfquake processing \
  -p /Users/admin/Desktop/surf_test/project.pkl \
  -i /Users/admin/Desktop/surf_test/metadata/inv_all.xml \
  -e /Users/admin/Desktop/surf_test/events/eventos.csv \
  -c /Users/admin/Desktop/surf_test/config/config.yaml \
  -o /Users/admin/Desktop/surf_test/processed_events
```

## Key Components Explained

| Parameter | Description |
|----------|-------------|
| `-w` | Path or glob pattern to waveform files (MiniSEED, HDF5, etc.) |
| `-c` | Configuration YAML file specifying preprocessing steps (filtering, tapering, normalization, etc.) |
| `--post_script` | Path to a Python script that can modify with your own processing |
| `--plot_config` | YAML file that defines plotting layout, styling, and pick output |
| `-o` | Output directory for saving processed or modified traces |
| `-p` | Path to a `.pkl` file with previously created surfquake project object|
| `-i` | Station inventory file (StationXML format) used for geolocation and instrument correction |
| `-e` | CSV file with origin times, locations, and hypocenter parameters for seismic events |
| `-a` | For automatic processing, no plotting waveforms |

## Tips for Reliable Bash Integration

- **Use absolute paths** to avoid ambiguity, especially when running the script from `cron` or remote sessions.
- **Quote paths** that include wildcards (`*`) to ensure the shell doesn't prematurely expand them.
- **Always activate your environment** if SurfQuake is installed via Conda or a virtualenv.
- **Use `--auto` flag** if you want the quick command to run without any user interaction (headless processing).

## Advanced: Scheduling or Looping

You can wrap this script into a `cron` job for periodic processing, or loop over multiple event folders:

```bash
for folder in /data/events/day_*; do
  surfquake quick -w "$folder/*.mseed" -c config.yaml -o "$folder/processed"
done
```
