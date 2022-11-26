---
title: "LTSpice Arbitrary Noise Sources"
date: "2022-08-02"
categories: 
  - "general"
  - "spice"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

This post is about a python package I strung together here: [https://github.com/tareqdandachi/spice-noise-daemon](https://github.com/tareqdandachi/spice-noise-daemon).

## Setup

Run `git clone https://github.com/tareqdandachi/spice-noise-daemon` in a directory of your preference. Then run the command `sudo ln -s $PWD/spice_noise.py /usr/local/bin/spicenoise` and you should be able to run `spicenoise -h` anywhere on your system.

## Creating noise sources

Run `spicenoise -s [circuit name].asc` on a circuit. This will generate a new `noise/` directory and a noise config file `noise_sources.yaml` in that directory. This file defines parameters regarding your noise sources and overall noise simulation.

Now if you run `spicenoise [circuit name].asc` it will generate 3 types of files:

1. a `.lib` file (in `noise/`) that contains subcircuits of all the noise sources you defined in your noise config yaml file.
2. corresponding `.csv` files (in `noise/`) that contain a sample of noise that follows the respective distributions defined for each subcircuit.
3. a bunch of `.asy` files (in the circuit directory) that are symbols you can import from the add components view.

Now if you open LTSpice and try to insert a new component (using the local components library, you can change that using the top of the components menu), you should see one noise source per entry in your `sources:` section of the yaml file.

## Using noise sources

Running the command from the previous section generates the following output in the terminal:

`Generating new noise component sources... Launched noise daemon... Use LTSpice normally now :)`

This is telling us that a new daemon is launched that checks for when you run a simulation. When a simulation is run, the daemon changes the noise `.csv` files so that you can get new entropy every time you run the simulation without having to type the command everytime. This means that you should make sure that the daemon is still running (on the specified file!) while LTSpice is running.

The entropy section of the yaml file is used to know how many data points to generate. Always make sure T is at least as big as your maximum simulation time (defined in your `.tran` command usually) with a sufficient number of STEPS.

## Types of noise sources

This is subject to change, so reference the `README.md` file in the repository,  but the three main types of noise currently defined are:

- gaussian: takes in mean and std
- poisson: takes in scale and lambda
- one\_over\_f: takes in power, fmin and scale

New noise types can be added from the python source or using a noise type `custom` that is parametrized by a python command such as `np.gaussian(...)`. See `README.md` for more details

The way you create a new noise source is by adding a new indented block under sources defining the `source_type`and `noise` parameters and the corresponding required values.

`entropy:`

 `T: 0.000001`

 `STEPS: 1000`

`sources:`

 `noise_source_1:`

 `source_type: current`

 `noise:`

 `type: one_over_f`

 `power: 1`

 `fmin: 0`

 `scale: 7e-7`

This is an example of defining 3 noise sources.

![](images/Screen-Shot-2022-08-02-at-11.43.00-AM-444x500.png)

 

## Simulating SNSPD dark counts

This is a quick example of biasing an SNSPD with a noisy source. (Red is the SNSPD current source, i.e. photon arrival times, Green is the readout).

![](images/Screen-Shot-2022-08-02-at-11.48.46-AM-1024x476.png)

Playing with the parameters, without having to rerun the `spicenoise` command (by just hitting run in LTSpice) the daemon generate new random sequences and gives us the following outputs:

![](images/Screen-Shot-2022-08-02-at-9.25.16-AM-500x458.png)
