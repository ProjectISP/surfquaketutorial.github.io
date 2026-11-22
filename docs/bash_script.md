# Bash Scripts with SurfQuake

SurfQuake is used from the **terminal**, by typing commands. That is perfect to explore a few files, but when you have to repeat the same steps many times (every day, for every event, for every station), typing them again and again is slow and error-prone. A **bash script** solves this: you write the commands once in a text file, and run them all with a single command.

This page is for beginners: no previous experience with bash is needed.

---

## What is bash?

- The **terminal** (or console) is the window where you type commands instead of clicking.
- **Bash** is the program inside the terminal that reads your commands and runs them. It is the standard on Linux and macOS (on recent macOS the default is `zsh`, which understands the same commands used here).
- A **bash script** is a plain text file (usually ending in `.sh`) containing a list of commands. Bash runs them one after the other, from top to bottom, exactly as if you had typed them.

So, if you can run a SurfQuake command in the terminal, you can already write a bash script: just put the commands in a file.

!!! note "Windows users"
    Bash is not the default on Windows. You can use it through **WSL** (Windows Subsystem for Linux) or **Git Bash**. Alternatively, write the same commands in a PowerShell or `.bat` file, one per line.

---

## Your first script

### 1. Create the file

Open any text editor (VS Code, gedit, nano, TextEdit in plain-text mode...) and create a file named `run_surfquake.sh` in your working folder:

```bash
#!/bin/bash
# My first SurfQuake script

surfquake explore -w "./data/*"
surfquake quick -w "./data/*" -c ./configs/typical_config.yaml -o ./output -a
```

- The first line, `#!/bin/bash`, tells the system that this file must be run with bash. Always keep it as the first line.
- Lines starting with `#` are **comments**: they are ignored, and you can use them to explain what each part does.
- The other lines are normal SurfQuake commands.

### 2. Make it executable

The first time, give the file permission to be run:

```bash
chmod u+x run_surfquake.sh
```

### 3. Run it

From the folder where the script is:

```bash
./run_surfquake.sh
```

You can also run it with `bash run_surfquake.sh`. Avoid `sh run_surfquake.sh`: on many Linux systems `sh` is not bash, and some lines may fail.

---

## Activating your environment

If you installed SurfQuake in a **conda** environment, the script must activate it before using `surfquake`, otherwise the command will not be found. Inside scripts, conda needs one extra line:

```bash
#!/bin/bash

# Activate the conda environment where SurfQuake is installed
eval "$(conda shell.bash hook)"
conda activate surfquake        # replace 'surfquake' with the name of your environment

surfquake quick -w "./data/*" -c ./configs/typical_config.yaml -o ./output -a
```

If you use a Python virtual environment instead, replace those two lines with:

```bash
source /path/to/your/venv/bin/activate
```

---

## Basic tools to write better scripts

### Long commands on several lines

A command can be split over several lines by ending each line with a backslash `\`. It is much easier to read. The backslash must be the **last character** of the line (no spaces after it):

```bash
surfquake quick \
  -w "./data/*" \
  -c ./configs/typical_config.yaml \
  -o ./output \
  -a
```

### Variables

A variable stores a value (often a path) that you can reuse. Define it with `NAME=value` (no spaces around `=`) and use it with `$NAME`. If you change your data folder, you only change one line:

```bash
DATA=./data
CONFIGS=./configs
OUTPUT=./output

surfquake quick -w "$DATA/*" -c "$CONFIGS/typical_config.yaml" -o "$OUTPUT" -a
```

### Quotes around paths and wildcards

Always write paths with wildcards between **quotes**: `"./data/*"`. Without them, bash expands the `*` itself and passes a long list of files to SurfQuake, which may fail or be slow. With quotes, SurfQuake receives the pattern and finds the files itself. Quotes also protect paths that contain spaces.

### Stop on the first error

By default, bash continues with the next line even if a command fails. Add this line after `#!/bin/bash` to stop the script as soon as a command fails, so a broken step does not produce wrong results downstream:

```bash
set -e
```

### Keep a log

To save everything the script prints (messages, warnings, errors) in a file, run it like this:

```bash
./run_surfquake.sh > run.log 2>&1
```

---

## Running without questions: automatic mode

In a script nobody is watching the screen, so SurfQuake must not open plot windows or ask questions. Use the option **`-a`** (`--auto`) with `quick`, `processing` and `processing_daily`:

- The traces are processed and written to the output folder (`-o`) in HDF5 format.
- No interactive window is opened.
- If you also want the **figures** saved automatically, set `show_plot: false` and a `plot_output_path` in your plotting configuration (see [Plotting](plotting_interaction_documentation.md)).

---

## Example: a complete workflow

This script chains two SurfQuake steps: it processes some waveform files with your own Python script, and then cuts and processes the waveforms of a list of events from a project.

```bash
#!/bin/bash
set -e

# --- Environment
eval "$(conda shell.bash hook)"
conda activate surfquake

# --- Paths (change them once here)
BASE=/home/user/surf_test
CONFIG=$BASE/config/config.yaml

# --- Step 1: process waveform files, with your own post-processing script
surfquake quick \
  -w "$BASE/nuclear/cut_event/*" \
  -c "$CONFIG" \
  --post_script "$BASE/scripts/my_script.py" \
  --plot_config "$BASE/config/plot.yaml" \
  -o "$BASE/output" \
  -a

# --- Step 2: cut and process the waveforms of a list of events
surfquake processing \
  -p "$BASE/project.pkl" \
  -i "$BASE/metadata/inv_all.xml" \
  -e "$BASE/events/events.txt" \
  -c "$CONFIG" \
  -o "$BASE/processed_events" \
  -a

echo "Workflow finished"
```

### The options used

| Option | Description |
|--------|-------------|
| `-w` | Path or wildcard pattern of the waveform files (MiniSEED, HDF5, ...). Always in quotes. |
| `-c` | Processing configuration file (YAML): detrend, taper, filter... See [Processing](processing_overview.md). |
| `--post_script` | Your own Python script, applied to each stream. See [Scripts](scripts.md). |
| `--plot_config` | Plotting configuration file (YAML). |
| `-o` | Output folder for the processed traces (HDF5). |
| `-p` | A SurfQuake project file (`.pkl`). See [Project](project.md). |
| `-i` | Station metadata (StationXML or dataless): station coordinates and instrument response. |
| `-e` | Event file (date, time, latitude, longitude, depth, magnitude). See [Processing](processing_overview.md). |
| `-a` | Automatic mode: no plots, no questions. |

`echo` simply prints a message on the screen.

---

## Repeating a task: loops

A `for` loop repeats commands for every item of a list. For example, to process every event folder separately:

```bash
#!/bin/bash
eval "$(conda shell.bash hook)"
conda activate surfquake

for folder in /data/events/event_*; do
    echo "Processing $folder"
    surfquake quick -w "$folder/*" -c ./configs/typical_config.yaml -o "$folder/processed" -a
done
```

- `folder` takes, one by one, the value of every folder whose name starts with `event_`.
- The commands between `do` and `done` run once for each folder.

Another example: save an image of every spectrogram in a folder.

```bash
for file in ./time_freq/*.spec; do
    surfquake specplot -f "$file" --split -d --save_path ./figures
done
```

---

## Running a script automatically: cron

On Linux and macOS, **cron** runs a script at fixed times, for example every night, without you being there. Open your cron table with:

```bash
crontab -e
```

and add a line like this one, which runs the script every day at 02:00 and saves the messages in a log file:

```
0 2 * * * /home/user/surf_test/run_surfquake.sh >> /home/user/surf_test/run.log 2>&1
```

The five fields are: minute, hour, day of the month, month and day of the week (`*` = every).

!!! warning "Paths in scheduled scripts"
    Cron does not start in your working folder, so relative paths like `./data` point somewhere else. In scripts run by cron, use **absolute paths** (starting with `/`), or add `cd /home/user/surf_test` at the beginning of the script.

---

## Tips

- **Test first**: run each command by hand on a few files, and only then put it in the script.
- **Use `-a`** in scripts, so nothing waits for an answer.
- **Quote** paths with wildcards or spaces: `"./data/*"`.
- **Activate the environment** at the beginning of every script.
- **Use variables** for the paths you repeat.
- **Use `set -e`** to stop at the first error, and keep a **log** of long runs.
- **Use absolute paths** in scripts run by cron or from another folder.
