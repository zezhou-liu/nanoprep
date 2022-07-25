# NanoPrep

[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

Nanopore formation, growth, conditioning and characterization using a [PyMeasure](https://pymeasure.readthedocs.io/en/latest/index.html) interface with a Keithley 2400 Sourcemeter.
Please cite this project if you use it in your research:

```BibTex
@Misc{nanoprep,
  author = {Xavier Capaldi},
  title = {Nanoprep: nanopore formation, characterization, growth and conditioning with Keithley 2400 Sourcemeter},
  howpublished = {\url{https://github.com/xcapaldi/nanoprep}},
  note = {Accessed YYYY-MM-DD},
  year = 2022,
}
```

This program was inspired by the following paper.
I strongly recommend citing it as well if you use this project for research that will be published.

```BibTex
@article{beamish-2012-precis-contr,
  author =	 {Eric Beamish and Harold Kwok and Vincent
                  Tabard-Cossa and Michel Godin},
  title =	 {Precise Control of the Size and Noise of Solid-State
                  Nanopores Using High Electric Fields},
  journal =	 {Nanotechnology},
  volume =	 23,
  number =	 40,
  pages =	 405301,
  year =	 2012,
  doi =		 {10.1088/0957-4484/23/40/405301},
  url =		 {https://doi.org/10.1088/0957-4484/23/40/405301},
  DATE_ADDED =	 {Tue Mar 2 10:06:05 2021},
}
```

## Drivers
The sourcemeter communicates with the computer via a [GPIB-USB-HS+ adapter from National Instruments](https://www.ni.com/en-ca/support/model.gpib-usb-hs-.html).

<img src="docs/gpib-to-usb.jpg" alt="GPIB-USB-HS+" width="300"/>

It has only been tested with Microsoft Windows due to the required drivers.
In particular I've tested it with [NI-VISA](https://www.ni.com/en-ca/support/downloads/drivers/download.ni-visa.html#346210) (v.20.0.0), [NI-488.2](https://knowledge.ni.com/KnowledgeArticleDetails?id=kA03q000000YGw4CAG&l=en-CA) (v.19.0.0) on Windows 10.

## Python dependencies
You will need Python 3.10 or higher installed.
You can check the dependencies in `requirements.txt` which holds the output of `pip freeze`.
If you want to install all the dependencies, just run `pip install -r requirements.txt`.

## Project structure
`nanoprep.py` is the main interface and loads all the protocols to populate the main window.
Unfortunately due to the code structure of PyMeasure, `nanoprep.py` does need to be modified as new protocols are added.
Protocols inherit from the `Protocol` class in `protocol_template.py`.
Protocols are actually relatively ignorant of the whole complexity of the system which was engineered to hopefully make protocol development faster and easier.
Each protocol has a `run` method with many parameters.
These must be supplied via PyMeasure `inputs` in `nanoprep.py`.
They are configured to display dynamically only when the associated protocol is selected to declutter to window.
Protocols can use the `logger`, `recorder` and `progressor` to write to the log, emit data for saving or notify of progress respectively.
The underlying PyMeasure tools have largely been wrapped by classes in the `utilities` directory.
There are helper classes as well for sensitive timing, calculating pore sizes and quickly plotting data.

### Configuration
A key feature of NanoPrep is its configuration file.
It uses the configuration format defined by Python's [configparser](https://docs.python.org/3/library/configparser.html).
There is a default configuration (`default.ini`) which should have an exhaustive list of parameters.
Each protocol can be listed and every default parameter can be set.
The user can write their own configuration and only include the protocols they use.
If you start NanoPrep by dragging a configuration onto the Python file (or a shortcut to that file), it will start by loaded the configuration.
Only the defined protocols will be displayed in the menu and all the fields will be set for them by default.

Here is an example configuration:

```INI
[DEFAULT]
gpib address = 19
compliance current = 1.0
solution conductivity = 11.53
channel conductance = 0.0
pipette offset = 0.0
progress style = absolute
data directory = C:\...

[iv curve]
sweep start = -0.200
sweep step = 0.020
sweep number = 21
sweep duration = 3
sweep discard = 2

[controlled dielectric breakdown]
breakdown voltage = 8
cutoff current = 200e-9
capacitance_delay = 20
```

### Emitted data
The saved data is formatted as a CSV with metadata and header:

```CSV
#Procedure: <__main__.NanoprepProcedure>
#Parameters:
#       Channel conductance: 0 S
#       Compliance current: 1 A
#       Breakdown voltage: 16 V
#       Cutoff current: 6e-07 A
#       Effective pore length: 1.2e-08 m
#       GPIB address: 19
#       Initial portion of sweep to discard in analyis: 2 s
#       Sweep duration: 3 s
#       Number of sweeps: 21
#       First sweep: -0.2 V
#       Step between sweeps: 0.02 V
#       Pipette offset: 0 V
#       Progress style: absolute
#       Protocol: iv curve
#       Solution conductivity: 7.52 S/m
#Data:
Time (s),Voltage (V),Current (A),Estimated diameter (m),State
0.002452900167554617,-0.2,-4.902002e-09,nan,nan
0.26572120003402233,-0.2,-4.781735e-09,nan,nan
0.3914447999559343,-0.2,-4.676931e-09,nan,nan
```

The metadata contains all experimental parameters relevant to the protocol.
As can be seen from the header, you can record Time, Voltage, Current, Estimated pore diameter, and an arbitrary state.
You do not need to supply a value for every field.
Unused fields will be filled by a `numpy.nan`.
State is very useful when you have some protocols that consists of smaller protocols lumped together.
For example, one that combines breakdown with characterization and conditioning.
You could pass states for each of the three processes so that later it is easier to filter the data.

### Quick plotting
`utilities/quick-plot.py` is just a quick-dirty-plotting tool which doesn't perform any analysis or cleaning of the data.
If you have this on a Microsoft Windows machine, you can drag and drop a data CSV directly onto the icon for this script and it will perform the plot.
Feel free to extend to fit your needs using this as a framework.

<img src="docs/quick-plot.png" alt="Result of dropping a data file onto quick-plot.py" width="500"/>

```Python
# import necessary packages
import sys
import csv
import matplotlib.pyplot as plt

# this will allow dragging and dropping csv's to plot on Windows
file = sys.argv[1]

# setup arrays
time = []
voltage = []
current = []

# open csv
with open(file, newline='') as csvfile:
    reader = csv.reader(csvfile)
    for row in reader:
        # ignore protocol details
        if row[0][0] != '#' and row[0][0] != 'T':
            time.append(float(row[0]))
            voltage.append(float(row[1]))
            current.append(float(row[2]) * 10E9)

# plot
plt.plot(time, current, '-', label='data')
plt.xlabel('time (s)')
plt.ylabel('current (nA)')
plt.legend()
plt.show()
```

## Contributor guidelines
Everyone is welcome to contribute to this project.
Unfortunately there is not an easy way to test changes or new protocols without have access to the actual hardware.
I am open to running those tests myself if you are unable.
To contribute, you can fork the project and open a pull request.
I just ask that you add any new dependencies to `requirements.txt` and run [black](https://github.com/psf/black) on any code changes.
Use [Google-style docstrings](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html) (although my current documentation is sparse).

Here are some projects that I think would contribute greatly to the project:

* Apply the [typing](https://docs.python.org/3/library/typing.html) library across the project for clarity.
* Introduce system tests for individual protocols using dummy `recorders`, `loggers`, `progressors` and `sourcemeters`.
* Create a dummy sourcemeter for PyMeasure so we can write actual system tests.
* Documentation (ideally use Google-style docstrings and [lazydocs](https://github.com/ml-tooling/lazydocs).
* Migrate old protocols and write new one.
* Write a configuration validator so users can see what parts of this config are not working.

## License
MIT License
Copyright (c) 2022 Xavier Capaldi