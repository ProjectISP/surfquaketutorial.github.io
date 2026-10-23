# <span style="color:darkcyan;">**Seismic Data Files**</span>

Seismic instruments record ground motion as a sequence of measurements through time. These measurements are stored in **waveform files**, which contain the digital samples of a seismic signal together with basic information needed to interpret their timing and acquisition. Depending on the sensor, the recordings may represent earthquakes, ambient seismic noise, controlled seismic sources, or other vibrations.

In **SurfQuake / Integrated Seismic Program (ISP)**, waveform handling is based on [ObsPy](https://docs.obspy.org/){: target="_blank" rel="noopener noreferrer" }, a Python framework widely used in seismology. Consequently, SurfQuake can work with the seismic waveform formats supported by ObsPy, subject to the requirements of each particular SurfQuake workflow.

!!! info "Waveforms and metadata are complementary"
    A waveform file contains the recorded samples and essential trace information, such as the channel identifier, start time, and sampling rate. **Station metadata**, often stored separately in StationXML, describe the station location, sensor orientation, and instrument response. Both are needed for many quantitative analyses. See the [Metadata](metadata.md) section for further information.

## Common seismic waveform formats

Several file formats are used in observational and exploration seismology. They differ in their structure, metadata fields, compression, and typical applications.

| Format | Common extension | Description and typical use |
| --- | --- | --- |
| **MiniSEED** | `.mseed`, `.msd`, `.seed` | Compact, widely used waveform format for continuous monitoring and data exchange between seismic networks. A `.seed` extension alone does not establish the file type. |
| **SAC** | `.sac` | Seismic Analysis Code format, commonly used for individual traces and earthquake or signal-processing studies. Includes an extensive waveform header. |
| **GSE2** | `.gse`, `.gse2` | Exchange format for seismic waveform data, encountered in network archives and legacy datasets. |
| **SEGY** | `.sgy`, `.segy` | Multi-trace format widely used for active-source seismic surveys and exploration. |
| **SEG2** | `.seg2`, `.dat` | Format often found in near-surface and engineering seismology, especially acquisition-system exports. |
| **Seismic Unix (SU)** | `.su` | Trace-oriented format used in seismic exploration and processing workflows. |
| **Other supported formats** | Varies | ObsPy also provides readers for formats such as SEISAN, AH, CSS, K-NET, and several instrument-specific formats. |

**MiniSEED is a good default for continuous seismic recordings**, while SAC is particularly convenient when working with individual traces and their analysis headers. The appropriate choice depends on the dataset and processing objective.

!!! note "File extensions do not guarantee the format"
    Filename extensions are conventions, not definitive identifiers. ObsPy usually detects the format from the file contents. Support for a format in ObsPy does not necessarily mean every SurfQuake feature accepts every possible file variant or acquisition-specific header.

For an authoritative list, consult the [ObsPy waveform reading documentation](https://docs.obspy.org/packages/autogen/obspy.core.stream.read.html){: target="_blank" rel="noopener noreferrer" }.

## How waveform data are organized

ObsPy uses two principal objects to represent waveform recordings:

| Object | Description | Main contents |
| --- | --- | --- |
| **Stream** | Collection of one or more seismic traces. | A set of `Trace` objects, possibly from different stations, components, or time intervals. |
| **Trace** | One regularly sampled time-series segment. | Sample array (`data`) and a header (`stats`) with identifiers, timing, and sampling information. |

A **Stream** may contain several traces, for example the vertical, north–south, and east–west components from one seismic station. The same channel may appear in multiple traces when a recording is divided into separate time segments.

```text
ObsPy Stream
  ├── Trace: XX.ABC.00.BHZ
  │     ├── data  → waveform samples
  │     └── stats → identifiers, timing, sampling rate
  ├── Trace: XX.ABC.00.BHN
  │     ├── data
  │     └── stats
  └── Trace: XX.ABC.00.BHE
        ├── data
        └── stats
```

- Identifying a seismic trace: NET.STA.LOC.CHA

Waveforms are commonly identified by four codes:

```text
Network.Station.Location.Channel
   XX  .  ABC .   00   . BHZ
```

- **Network (`XX`)**: the seismic network.
- **Station (`ABC`)**: the recording station.
- **Location (`00`)**: identifies a particular sensor or data stream at the station; it may be empty.
- **Channel (`BHZ`)**: identifies the acquisition channel, including its component designation under the relevant naming convention.

These codes connect a waveform to the corresponding station and channel metadata. **They do not, by themselves, specify the recording interval or guarantee that the instrument configuration remained unchanged.**

## What information matters most?

- Sampling rate and recording time

Each regularly sampled trace has a **sampling rate** \(f_s\), measured in samples per second (Hz). The time interval between consecutive samples is

\[
\Delta t = \frac{1}{f_s}.
\]

For example, at 100 Hz the interval between samples is 0.01 s. The theoretical Nyquist frequency is \(f_N=f_s/2\), although the frequency band that can be interpreted reliably also depends on the recording instrument and its acquisition filters.

A trace header normally includes its **start time**, **sampling rate**, and **number of samples** (`npts`). ObsPy uses these quantities to determine the time associated with each sample. Times are normally handled in UTC through `UTCDateTime`.

- Data gaps, overlaps, and continuity

Continuous archives may contain **gaps** (missing time intervals), **overlaps** (repeated or conflicting samples), or segments with different sampling rates. These issues can affect filters, spectral analyses, event detection, and other processing methods.

!!! warning "Check continuity before signal processing"
    Do not assume that all files from one station form an uninterrupted record. Inspect missing intervals and overlaps before merging data. Filling gaps with zeros or interpolated samples changes the time series and can introduce artificial signal content.

- Sample amplitudes and physical units

Seismic waveforms are often stored as **digital counts**. Counts are not necessarily displacement, velocity, or acceleration. Obtaining calibrated physical measurements generally requires the corresponding **instrument response** and an appropriate response-removal procedure.

For more details, see the [Metadata](metadata.md) section.

## Preparing waveform files for SurfQuake / ISP

Before importing data into a processing project, it is good practice to:

- Confirm that the files can be read and contain the expected channels and time intervals.
- Check the sampling rates and identify any gaps or overlaps.
- Ensure that network, station, location, and channel identifiers are consistent.
- Prepare matching **StationXML or other compatible station metadata** when station positions, orientation, or response corrections are required.
- Preserve the original recordings and perform processing on separate copies when practical.

These checks are especially useful for continuous monitoring, earthquake waveform analysis, ocean-bottom seismometer recordings, and ambient-noise processing.

## Remarks

**Seismic waveform files contain the recorded time series and their basic acquisition information.** Different formats serve different applications, but SurfQuake / ISP benefits from ObsPy's common `Stream` and `Trace` interface for working with them.

MiniSEED and SAC are particularly common in observational seismology. Regardless of the format, reliable processing begins by checking **trace identifiers, timing, sampling rates, continuity, and the availability of corresponding station metadata**.
