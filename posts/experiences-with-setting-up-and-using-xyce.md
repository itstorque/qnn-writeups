---
title: "Experiences with setting up and using Xyce"
date: "2022-08-31"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

I setup Xyce Serial on my machine (an M1 Mac) in 3 different ways to see what was the most convenient way to set it up. I built the source using both an M1 native setup and using x86 emulation, everything installed using brew. I also used Alessandro Restelli's [docker image](https://github.com/restelli/xyce_docker_image) and [docker playground](https://github.com/qnngroup/xyce_playground).

## What is Xyce

Xyce is a circuit simulator like LTspice made by Sandia labs. It isn't actually based on SPICE. It's very fast and parallelizable (if you setup Xyce Parallel). It doesn't have a GUI and is more annoying to work with. It can run diff. eq.s and is more capable than LTspice.

## Building Xyce on M1 with natively and with x86 emulation (Horrible)

On both native and emulation, installing the dependencies was straight forward using brew. Building Xyce natively and/or using x86 emulation takes about 35 mins if setup properly - Trilinos takes 15mins and Xyce took 20mins. However, getting it to build took forever. This might be because I am on a Mac and homebrew changed the default installation directory so all commands the Xyce has need to be remapped to the new brew installation location. The libraries used to be installed in /usr/local/lib and the headers used to be in /usr/local/include. Newer editions of homebrew now install to /opt/homebrew/lib and /opt/homebrew/include.

Another issue was Xyce's base dependencies on BLAS and LAPACK not working. BLAS and LAPACK are preinstalled on computers with gcc/clang installed, however, I couldn't get them to link properly and had to install the open versions of them using brew (open-blas and liblapack) - annoying that you have to do that and takes storage/adds dependencies for no reason but worked fine for me. Actually even in the docker env open-blas and  are installed, so I actually recommend you install them regardless of your machine type. Getting fortran to behave on the x86 emulation was hard but natively it was ok. For some reason I couldn't get gcc++ to link properly natively but it is possible to swap (gcc and gcc++) with clang and clang++. Doing so will make the setup command you need to change quite a bit in the build command you run, including the flags, the LDFLAGS passed to the Xyce configuration script need to be altered such that `Wl,-framework,Accelerate` becomes `-framework Accelerate` instead.

Also do note that gcc anmd g++ natively on a mac are not the GNU Compiler Collection but are aliases to clang! So unless you installed gcc8 or some other gcc from brew, you **are using** clang.

Overall, would not recommend either of these two method. It was more annoying because I was on an M1, but I don't think the experience would've been any better on an x86 Mac either. I haven't set it up on Linux or Windows, so I can't speak for those experiences as much, but given that the primary audience for people running Xyce is probably big Linux servers, I imagine that the building experience is not amazing on Windows too (and probably Linux).

## Using Docker (Slightly less bad)

For this method, you will need to install Docker beforehand - I recommend going to the docker website to install it from there. Alessandro Restelli has a built docker image that should have Xyce on it and if you install it and run it with docker it should probably work. To do so you also need anaconda (if you're like me and don't have anaconda, you can install miniconda!) then it should be as easy as creating a docker environment, installing the dependencies and you should have a way of launching Xyce. There are some instructions in [qnngroup/xyce\_playground](https://github.com/qnngroup/xyce_playground). However, this didn't work for me. I had to build my own docker image of Xyce from [restelli/xyce\_docker\_image](https://github.com/restelli/xyce_docker_image). Once you do so, installing the dependencies is also not straight forward. The `xyce_playground/environment.yml` file has a list of dependencies you need, however, a lot of them aren't available on mac (or any non-linux emulating shell? maybe windows? not sure.) Even so, a lot of the dependencies have hashes that aren't compatible with Mac too. What I did was remove all the hash definitions after every package (using regex string matching) and then remove the dependencies that it failed to find. I will paste the final environment.yml file contents I used at the bottom of this post - theoretically it is cross-platform and should work on your end. Then you can properly build the xyce\_plaground and (hopefully have jupyter lab preinstalled, if not install it) and then run a jupyter lab notebook from the conda environment and the xyce\_playground has a tool that should communicate with Xyce and send netlists and receive outputs from it.

## Using Xyce

The experience is fine. It is wayyyy less nice than LTspice. It has no GUI, so you have to write in netlists or import them from LTspice. However, netlists are a little different in Xyce, so you have to modify them a little if you import them. It seems powerful and can run diff. eq. problems on it in parallel with your circuit, but I haven't gotten a chance to play with it a lot. I don't particularly like Xyce and the playground setup is okay, it makes Xyce work, but the setup is annoying and the package layout didn't work for me out of the box. It also doesn't "encourage" me to run multiple simulations - I just want to leave the jupyter page and running long netlists is annoying in the current way things are done, the output is bloated, you can't visualize plots as its being simulated etc. It offered a speed up over LTspice, but I am not sure if its enough to justify how much I hated everything else. Also visualizing plots isn't as nice as LTspice - you can either install an external tool or use matplotlib - not ideal imo.

## How I see Xyce being used

I think it doesn't make sense to have people use Xyce instead of LTspice if they don't absolutely need it. The only time where I would recommend this is if you absolutely need the speedup and are running ridiculously large simulations (millions of chunks, millions of timesteps, order of 1hr...). I also was using Xyce Serial on my 1o core machine, LTspice was comparable because it actually can make use of all the cores. However, I do think that setting up Xyce parallel on a Linux machine with a lot of cores centralized to the group might be a good idea. The infrastructure to communicate with it needs to be setup such that it isn't too bad to use - I think an ability to use LTspice as a front-end for Xyce might be nice - namely, you can build an LTspice circuit, get the netlist and apply some function on it to make it Xyce-y and then send it to Xyce let it solve and download a raw file you can open in LTspice (not sure how hard the last step would be, but I think its doable). I think this infrastructure of making it usable and easy enough such that people prefer it to LTspice is a pretty high bar and making that infrastructure seems like it would take a while. I think Xyce Serial is not worth building. Note that Xyce Serial is easier to build than Xyce Parallel, not sure by how much since I didn't attempt to do so.
