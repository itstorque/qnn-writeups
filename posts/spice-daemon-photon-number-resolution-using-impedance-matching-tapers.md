---
title: "spice-daemon photon number resolution using impedance matching tapers"
date: "2022-08-12"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

There are a bunch of **\[todo!\]** labels in this article that will be removed in the next week or so (Aug 12).

From Stewart's presentation during the spice subgroup meeting, impedance matching tapers can produce different amplitude responses for different incident photon counts, allowing us to use it for photon number resolution. In this post I will demonstrate how to use the spice-daemon workflow to produce a test that implements that photon resolution method alongside some input noise into the system. Before you start, make sure to follow the install directions on the GitHub page for spice-daemon found here **\[todo!\]**: [https://github.com/tareqdandachi/spice-daemon](https://github.com/tareqdandachi/spice-daemon)

The spice-daemon package is meant to be a python package that interfaces with LTSpice to feed in more complicated designs that can't be generated in LTSpice, such as shot noise and variable sized inputs or even modules that provide convenience such as decoupled gaussian noise sources using one element.

### Starting a new spice-daemon circuit

Create a new LTSpice circuit (or use an existing one). For this example, the circuit is called `circuit.asc`. We can then run the setup command for spice-daemon on that file `spice -s circuit.asc`. This command will create a new directory titled `.spice-daemon-data` where most of the daemon files will be placed (except for auto generated symbol files). This directory contains a file called `spice-daemon.yaml`, this is the only file you will need to edit to interface with the spice-daemon package. This YAML file is a human-readable data-serialization language that is commonly used in config files. Tabulation matters, kind of like a markdown list, where each item inherits all of its underlying objects, here is an example:

`module_1:`

 `part_a:`

 `property_1: 500`

 `property_2: this_is_a_string`

 `part_b:`

 `property_1: string`

 `property_2: 3`

Translates to the python dictionary

`{'module_1':``{`

`'part_a' : {'property_1': '500', 'property_2': 'this_is_a_string'}`

`'part_b' : {'property_1': 'string', 'property_2': '3'}`

`}``}`

The one module that is always (and is especially important to change if you are using noise) is the `sim` module. All config files should have them and setting the `T` and `STEPS` of a simulation should happen in here as opposed to in LTSpice's `.tran` directive **\[todo!\]**. If you set `STEPS: None`, then the simulator will use the default timesteps LTSpice wants but won't generate noise - will keep noise at `0`. Any tran directives in there will be automatically commented out by the daemon and replaced by the spice-daemon's tran command!

### Editing spice-daemon.yaml

For this example, we want to have a gaussian noise source and an impedance matching taper. If you want to include an instance of some module in the yaml file, say noise\_sources, we want to declare the noise\_sources module and place all the noise components we want in our circuit under it as parts (do not include two modules or parts of the same name!).

\[caption id="attachment\_24571" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-11-at-4.29.29-PM-1024x552.png) This is the spice-daemon.yaml file setup for the photon resolution example, declaring one klopfenstein taper from 50 to 1000 ohms and a gaussian voltage noise source\[/caption\]

If we wanted to have more tapers, we would place taper\_2 at the same indentation level as taper\_1 and set the same parameters below it as taper\_1. Documentation on the available modules and how they are parameterized are on the README.md **\[todo!\]** of the GitHub page.

### Running the daemon

The daemon is a listener that looks for changes in the yaml config file or checks if entropy sources have been used - the daemon then deals with updating the components as needed. The daemon needs to be running while using LTSpice for it to update components and noise, otherwise you will have static noise (replays the last clip of noise generated) and any changes in component parameters in the config file don't get propagated. To run the daemon on our file, you can simply type `spice circuit.asc`. While the daemon starts up, you will see this while the components are initially being generated `Generating new components...`. Afterwards `Launched daemon... Use LTSpice normally now :)` will indicate that the daemon is listening and is alive.

After generating the new components, you will see two new `.asy` files appear in your working directory, one main `.lib` file in the `.spice-daemon-data` and one `.csv` file. In general, you will have one symbol file per component, one lib file in total and one csv file per noise source.

### Using the taper

In our components library, we can change the top directory to the local directory where we will get all of the components we declared in the yaml file. We can see that there are 2 elements which we can choose to place down. Note that reusing a noise source will use the same entropy source - if you want multiple noise sources that aren't the same, you will have to generate them separately. **\[todo!\]** bug in taper description.

![](images/Screen-Shot-2022-08-12-at-9.44.25-AM-445x500.png)

We can now test the taper on two transmission lines. If the taper doesn't impedance match, then we would get reflections. In this case, we can see a clean pulse traveling through the taper that is matched to both the 50 and 1000 ohm tline.

\[caption id="attachment\_24572" align="aligncenter" width="418"\]![](images/Screen-Shot-2022-08-11-at-4.42.53-PM-418x500.png) Impedance matching example, where the blue pulse travels into a 50 ohm tline, into an impedance matching klopf taper and out into a 1000 ohm tline.\[/caption\]

The taper implementation uses [https://github.com/qnngroup/tapers](https://github.com/qnngroup/tapers). When creating a taper, it often ends with impedance values that aren't exactly what were requested. You can figure out the actual impedance by checking the lib file for the $L, C$ values at start and end of the taper. **\[todo!\]** figure out if the labels on taper should reflect these numbers.

### Photon Number Resolution

We can now setup 4 circuits parameterized by different number of incident photons using a taper. Scoping the pulse on the output side of the taper, we should be able to resolve photon count as a function of amplitude.

\[caption id="attachment\_24573" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-11-at-3.41.31-PM-1024x662.png) Photon resolution example. As more photons are incident on the wire, the amplitude of the pulse on the output side increases due to the impedance matching taper.\[/caption\]

Yay! it works :)

### Adding noise

Adding a noise source after the taper with comparable magnitude to the bias current we get the following signal. Note that the number of noise STEPS should be sufficient enough for the simulation time/steps, if you are unsure, reference the spice-daemon repo **\[todo!\]**.

\[caption id="attachment\_24574" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-11-at-5.06.11-PM-1024x662.png) 1uA noise simulation using spice-daemon noise on the photon number resolution setup.\[/caption\]

Some really horrible noise:

![](images/Screen-Shot-2022-08-11-at-5.15.49-PM-1024x662.png)
