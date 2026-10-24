# <span style="color:darkcyan;"> **Welcome to** </span>
![Screenshot](img/logo.png)

Surfquake has been designed to processing seismic traces and to streamline the workflow of estimating seismic source parameters. 
The signal processing toolbox is extended with a lage variety of numerical methods while the comprehensive set of workflow toolboxes automates the determination of arrival times, event locations, event magnitudes , attenuation, and moment tensor inversion. The software is programmed in <span style="color:#5DADE2;">*Python 3*</span> and offers the users the possibility of three programming levels for flexibility and customization.

- The **Core Library**  allows users to integrate the core of surfQuake into their preexisting scripts, giving advanced users full control.

- The **Command Line Interface**  gives users access to an upper layer that simplifies the use of the core, enabling task execution through simple commands.

- The **Graphical User Interface (GUI)**: Now SurfQuake user-friendly interface is fully included in [Integrated Seismic Program](https://projectisp.github.io/ISP_tutorial.github.io/){: target="_blank" rel="noopener noreferrer" }, making it accessible to users who prefer visual interaction. 

---


## <span style="color:darkcyan;"> **Cite surfQuake** </span>


- [<span style="color:#5DADE2">*Seismological Research Letters paper*</span>](https://pubs.geoscienceworld.org/ssa/srl/article-abstract/doi/10.1785/0220240186/654285/surfQuake-A-New-Python-Toolbox-for-the-Workflow?redirectedFrom=fulltext)

---

## <span style="color:darkcyan;"> **Case of Study and Supporting Materials** </span>
First: [<span style="color:#5DADE2;">Case of study</span>](https://github.com/rcabdia/test_surfquake), contains a full example of using surfQuake with core Library Python scripts, Core Library bash script and expected results. Of course you can try to run the example using the GUI.

Second: [<span style="color:#5DADE2;">Earth Velocity models examples</span>](https://github.com/rcabdia/ProjectISP_3Dmodels). Event Location format and MTI format.

---


## <span style="color:darkcyan;"> **Keeping Updated** </span>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Links Grid</title>
    <style>
        .links-container {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 15px;
            max-width: 800px;
            margin: auto;
            background: white;
            padding: 20px;
            border-radius: 8px;
        }

        .link-item {
            text-decoration: none;
            display: flex;
            align-items: center;
            background: #f8f8f8;
            padding: 10px;
            border-radius: 8px;
        }

        .link-item img {
            width: 40px;
            height: auto;
            margin-right: 10px;
        }

        .link-item span {
            font-size: 16px;
            color: black;
            font-weight: bold;
        }

        .link-item:hover {
            background: #eaeaea;
        }

        @media (max-width: 600px) {
            .links-container {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
    <div class="links-container">
        <a href="https://github.com/rcabdia/SurfQuakeCore"
           target="_blank" rel="noopener noreferrer" class="link-item">
            <img src="img/github.png" alt="GitHub">
            <span>Open-source code</span>
        </a>

        <a href="https://twitter.com/ProgramSeismic"
           target="_blank" rel="noopener noreferrer" class="link-item">
            <img src="img/twitter.png" alt="Twitter">
            <span>Twitter</span>
        </a>

        <a href="https://github.com/ProjectISP/ISP/issues"
           target="_blank" rel="noopener noreferrer" class="link-item">
            <img src="img/questions.png" alt="Questions and Issues">
            <span>Questions and Issues</span>
        </a>
    </div>
</body>
</html>

---


## <span style="color:darkcyan;"> **ToolBoxes** </span>
<div class="sq-map">
  <section class="sq-category sq-category--start" aria-label="Getting Started">
    <h3>Getting Started</h3>
    <div class="sq-items">
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/seismic_data_files/">Seismic Files</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/metadata/">Metadata</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/project/">Project</a>
    </div>
  </section>
  <section class="sq-category sq-category--tutorial" aria-label="Tutorial Book">
    <h3>Tutorial Book</h3>
    <div class="sq-items">
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/tutorials/tutorial1/">Basic Signal Processing</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/tutorials/tutorial2/">Time-Frequency Analysis</a>
    </div>
  </section>
  <section class="sq-category sq-category--signal" aria-label="Signal Processing">
    <h3>Signal Processing</h3>
    <div class="sq-items">
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/processing_overview/">Overview</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/examples/">Examples</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/trace_header_documentation/">Trace Header</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/explore_data/">Data Availability</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/scripts/">My Scripts</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/processing_config/">Processing Traces</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/processing_stream_config/">Processing Stream</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/beamforming_documentation/">Beamforming</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/particle_motion_documentation/">Particle Motion</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/spectral_analysis_documentation/">Time-Frequency Analysis</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/characteristic_functions_documentation/">Characteristic Functions</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/plotting_interaction_documentation/">Plotting Interaction</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/bash_script/">Bash Script Interaction</a>
    </div>
  </section>
  <section class="sq-category sq-category--event" aria-label="Event Analysis">
    <h3>Event Analysis</h3>
    <div class="sq-items">
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/picker/">Phase Picker</a>
      <div class="sq-subgroup">
        <h4>Event Associator</h4>
        <div class="sq-items">
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/associate/">Clustering Associate</a>
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/coincidence/">Coincidence Trigger</a>
        </div>
      </div>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/locate/">Event Location</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/source/">Source Parameters</a>
      <div class="sq-subgroup">
        <h4>Focal Mechanism</h4>
        <div class="sq-items">
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/focmec/">First Motion Polarity</a>
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/mti/">Moment Tensor Inversion</a>
        </div>
      </div>
    </div>
  </section>
  <section class="sq-category sq-category--noise" aria-label="Noise Processing">
    <h3>Noise Processing</h3>
    <div class="sq-items">
      <div class="sq-subgroup">
        <h4>PPSD</h4>
        <div class="sq-items">
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/ppsd_DB/">Database</a>
          <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/ppsd_plot/">Plotting &amp; Statistics</a>
        </div>
      </div>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/ambient_noise/">Ambient Noise Processing</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/dispersion_curves_processing/">Dispersion Curves Processing</a>
    </div>
  </section>
  <section class="sq-category sq-category--utils" aria-label="Utilities">
    <h3>Utilities</h3>
    <div class="sq-items">
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/create_metadata/">Create Metadata</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/manage_catalog/">Manage Catalog</a>
      <a class="sq-link" href="https://projectisp.github.io/surfquaketutorial.github.io/build_mti/">Build MTI Config</a>
    </div>
  </section>
</div>



---

## <span style="color:darkcyan;"> **Cite surfQuake** </span>


Cabieces, R., Junqueira, T. C., Harris, K., Relinque, J., Satriano, C. & Vackář, J. (2025). surfQuake: A new Python toolbox for the workflow process of seismic sources. Seismological Research Letters, 96(5), 3231-3243.