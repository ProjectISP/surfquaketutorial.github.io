# Custom User Scripts in SurfQuake

SurfQuake allows users to inject their own custom Python logic into waveform processing workflows. This functionality is available in both **batch processing (`surfquake process`)** and **quick interactive workflows (`surfquake quick`)**.

---

## Purpose

User scripts provide a flexible, programmable interface to manipulate seismic traces beyond standard configuration options. You can implement your own algorythm in the workflow such as:

- Apply custom filtering, transformations, or corrections
- Modify or inspect waveform data (`tr.data`)
- Inject or update metadata in `tr.stats`. These scripts have direct access to each trace and its metadata (`trace.stats`), allowing you to inspect, modify, or annotate data based on custom logic. Be in mind the modified header of traces in surfquake.
- Run scientific algorithms (e.g., polarization, machine learning)
- Export trace summaries or diagnostics

---

## Script Structure

Each script must define a `run(stream, **kwargs)` function that takes:
- `stream`: an ObsPy `Stream` object containing the current event or waveform group
- `kwargs`: optional keyword arguments, such as `inventory` or `event` info

### Example

```python
import numpy as np

def run(stream, **kwargs):
    '''
    User-defined post-processing hook for each event.

    :param stream: obspy Stream
    :param inventory: obspy Inventory (passed via kwargs)
    :param event: dict with keys like origin_time, latitude, longitude, depth (passed via kwargs)
    '''
    inventory = kwargs.pop('inventory', None)
    event = kwargs.pop('event', None)

    if event is not None:
        print(f"Post-processing {len(stream)} traces from event at {event['origin_time']}")
    else:
        print(f"Post-processing {len(stream)} traces (no event info provided)")

    # Example: calculate global mean amplitude
    all_data = np.hstack([tr.data for tr in stream])
    print("Mean amplitude:", np.mean(all_data))

    # Example: apply a simple differencing operator to each trace
    for tr in stream:
        tr.data = np.diff(tr.data)

    return stream
```

---

## Usage in CLI

### `surfquake process`

```bash
surfquake process --config config.yaml --post_script myscript.py
```

- Executes your script after all config steps are complete.
- Useful for event-level operations or header editing before writing to disk.

### `surfquake quick`

```bash
surfquake quick \
  -w "./data/*.mseed" \
  -i inventory.xml \
  --post_script myscript.py \
  --post_script_stage before
```

- `--post_script`: Path to the user script.
- `--post_script_stage`:
  - `before`: Runs the script before plotting (good for data transforms)
  - `after`: Runs after plotting (useful if user interaction modifies data)

---

## Best Practices

- Always return the modified stream from `run()`.
- Use `tr.stats` to store temporary or derived metadata.
- Print useful info for logging/debugging.
- Test your script independently before integrating it into batch workflows.

---

## Advanced Use

You may combine user scripts with processing configs to create hybrid workflows. For example, normalize traces with a config, then use a user script to detect and annotate phase arrivals using custom logic.

```yaml
# config.yaml
Analysis:
  process_1:
    name: 'normalize'
```

Then:

```bash
surfquake process --config config.yaml --user_script annotate_picks.py
```

---

This feature makes SurfQuake highly adaptable for both research and operational seismic processing pipelines.

