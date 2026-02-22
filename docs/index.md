# <span style="color:darkcyan;"> **Welcome to** </span>
![Screenshot](img/logo.png)

has been designed to streamline the workflow of estimating seismic source parameters. Comprehensive set of toolboxes automates the determination of arrival times, event locations, event magnitudes , attenuation, and moment tensor inversion. The software is programmed in <span style="color:#5DADE2;">*Python 3*</span> and offers the users the possibility of three programming levels for flexibility and customization.

- The **Core Library**  allows users to integrate the core of surfQuake into their preexisting scripts, giving advanced users full control.

- The **Command Line Interface**  gives users access to an upper layer that simplifies the use of the core, enabling task execution through simple commands.

- The **Graphical User Interface (GUI)**: Wraps the SurfQuake core in a user-friendly interface, making it accessible to users who prefer visual interaction. The GUI is connected to a *SQLite database* to store all results, ensuring easy retrieval and management of data.

---

## <span style="color:darkcyan;"> **News** </span>

Combine the flexibility of command-line tools, Python scripts, and Bash to supercharge your seismic trace processing and workflow:

- [<span style="color:#5DADE2">*surfquake 0.1.6 by October 2025: New Coincidence Trigger tool*</span>](https://projectisp.github.io/surfquaketutorial.github.io/coincidence/)

- [<span style="color:#5DADE2">*surfquake 0.1.6 by October 2025: New First Motion Polarity tool*</span>](https://projectisp.github.io/surfquaketutorial.github.io/focmec/)

- [<span style="color:#5DADE2">*surfquake 0.1.6 by September 2025: New Signal Processing Module*</span>](https://projectisp.github.io/surfquaketutorial.github.io/processing_overview/)

Take a look at our recently published paper!

- [<span style="color:#5DADE2">*Seismological Research Letters paper*</span>](https://pubs.geoscienceworld.org/ssa/srl/article-abstract/doi/10.1785/0220240186/654285/surfQuake-A-New-Python-Toolbox-for-the-Workflow?redirectedFrom=fulltext)


---

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
            margin-right: 10px;
        }
        .link-item span {
            font-size: 16px;
            color: black;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="links-container">
        <a href="https://github.com/rcabdia/SurfQuakeCore" target="_blank" class="link-item">
            <img src="img/github.png" width="40">
            <span>See surfQuake open-source core code</span>
        </a>
        <a href="https://twitter.com/ProgramSeismic" target="_blank" class="link-item">
            <img src="img/twitter.png" width="40">
            <span>Follow us on Twitter</span>
        </a>
        <a href="https://www.youtube.com/@robertocabiecesdiaz3775/videos" target="_blank" class="link-item">
            <img src="img/youtube.png" width="40">
            <span>Subscribe on YouTube</span>
        </a>
        <a href="https://github.com/ProjectISP/ISP/issues" target="_blank" class="link-item">
            <img src="img/questions.png" width="60">
            <span>Questions and Issues</span>
        </a>
        <a href="mailto:isp@roa.es?subject=Subscription Request&body=Hello, I would like to subscribe to surfQuake news and future workshops" class="link-item">
            <img src="img/mail.png" width="60">
            <span>Subscribe to news and future workshops</span>
        </a>
    </div>
</body>
</html>


## <span style="color:darkcyan;"> **ToolBoxes** </span>
- [<span style="color:#5DADE2">*Create your Project* </span>](https://projectisp.github.io/surfquaketutorial.github.io/project/)
- [<span style="color:#5DADE2">*Signal Processing* </span>](https://projectisp.github.io/surfquaketutorial.github.io/processing_overview/)
- [<span style="color:#5DADE2">*Phase Picker* </span>](https://projectisp.github.io/surfquaketutorial.github.io/picker/)
- [<span style="color:#5DADE2">*Event Associator* </span>](https://projectisp.github.io/surfquaketutorial.github.io/associate/)
- [<span style="color:#5DADE2">*Coincidence Trigger* </span>](https://projectisp.github.io/surfquaketutorial.github.io/coincidence/)
- [<span style="color:#5DADE2">*Event Location* </span>](https://projectisp.github.io/surfquaketutorial.github.io/locate/)
- [<span style="color:#5DADE2">*Source Parameters* </span>](https://projectisp.github.io/surfquaketutorial.github.io/source/)
- [<span style="color:#5DADE2">*First Motion Polarity* </span>](https://projectisp.github.io/surfquaketutorial.github.io/focmec/)
- [<span style="color:#5DADE2">*Moment Tensor Inversion* </span>](https://projectisp.github.io/surfquaketutorial.github.io/mti/)
- [<span style="color:#5DADE2">*Data Base* </span>](https://projectisp.github.io/surfquaketutorial.github.io/db/)
- [<span style="color:#5DADE2">*Utils* </span>](https://projectisp.github.io/surfquaketutorial.github.io/utils/)

## <span style="color:darkcyan;"> **How to follow surfQuake Tutorial** </span>

The first step is to select which programming level is the most convenient for you. SurfQuake is divided into five Toolboxes: Picking, Association, Locate, Source, and Moment Tensor Inversion (MTI). Each toolbox link below contains a description of the software according to your programming level and an example.

Let's start with [<span style="color:#5DADE2">*Project* </span>](https://projectisp.github.io/surfquaketutorial.github.io/project/) and then continue with the next toolboxes.

## <span style="color:darkcyan;"> **Case of Study and Supporting Materials** </span>
First: [<span style="color:#5DADE2;">Case of study</span>](https://github.com/rcabdia/test_surfquake), contains a full example of using surfQuake with core Library Python scripts, Core Library bash script and expected results. Of course you can try to run the example using the GUI.

Second: [<span style="color:#5DADE2;">Earth Velocity models examples</span>](https://github.com/rcabdia/ProjectISP_3Dmodels). Event Location format and MTI format.


## <span style="color:darkcyan;"> **Cite surfQuake:** </span>


Cabieces, R., Junqueira, T. C., Harris, K., Relinque, J., Satriano, C. & Vackář, J: SurfQuake: A new Python toolbox for the workflow process of seismic sources, Seismological Research Letters, may 2025.
