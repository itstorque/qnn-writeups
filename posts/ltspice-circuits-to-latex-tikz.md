---
title: "LTspice circuits to $\LaTeX$ tikz"
date: "2022-09-23"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

I made a crude [LTspice schematic and symbol to $\LaTeX$ pgp/tikz converter](https://github.com/tareqdandachi/ltspice-tikz). By running the command `ltspice-tikz circuit.asc` the tikz source is copied into your clipboard. To create the symlink to use the command ltspice-tikz instead of python3 draw, run the following command: `sudo ln -s $PWD/spice.py /usr/local/bin/spice`. You can also import `draw.py` and use the function `circ_to_latex(circuit, component_name=False, component_value=False, local_dir=dir, entire_page=True)` to get more customization and automatically generate tikz plots. You can also connect it to your own TeX interpreter and spice-daemon to automatically generate pdf previews of your circuits everytime you hit save. It is very crude as of now, feel free to make modifications on the package as you see needed.

![](images/Screen-Shot-2022-09-23-at-2.46.44-PM.png)

Unlike other existing options, you don't have to make a tikz picture for every component that isn't part of the standard library as this recursively checks symbol references and redraws the symbols from source. Here is a link to the github page: [https://github.com/tareqdandachi/ltspice-tikz](https://github.com/tareqdandachi/ltspice-tikz).

Components are packaged into tikzsets and wires are drawn manually, this means that it should be relatively straight forward to move this from tikz to circuit-tikz, but I chose not to because I am not an avid user of circuit-tikz (the only difference is you need to be aware of ltspice wiring topology for circuit-tikz which is pretty cool and easy to exploit for fake connections).
