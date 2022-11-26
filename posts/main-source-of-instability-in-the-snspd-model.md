---
title: "Proposed fixes for the SNSPD model"
date: "2022-09-26"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

After using the malicious circuits method of measuring stability and using the deconstructed SNSPD model to further test the SNSPD model, I was able to locate the main source of instability in the SNSPD model. The hotspot integrator is the main reason for the instability. While a circuit integrator is less stable than the built-in integrator in most cases, it is even more unstable when including a switch that shorts to ground. Luckily, LTspice has a built-in integrator with assertions to fix this instability. I.e. by running $sdt(f(t), u\_0,  p>0)$ if the condition $p>0$ fails, then the integral valuation resets to $u\_0$. I ended up making three changes:

- expression for the hotspot growth voltage source from $\dfrac{v\_{N3}+|v\_{N3}|}{2}i\_{B1}$ to an if statement over the values $\{v\_{N3}i\_{B1}, 0\}$ based on whether the wire is s/c
- The hotspot integrator into a voltage source with the diff. eq. expression.
- Removing the resistor in the state "boolean" circuit

This is a circuit side-by-side for the modified sections where the left image is the modified model and the right is the existing model:

![](images/Screen-Shot-2022-09-26-at-2.42.02-PM.png)

Here are two simulations run with two reltol values running a test of a bias current current of $15\mu$A for the first $10$ns with a photon incident at $2$ns followed by a bias of $25\mu$A.

### default reltol (1e-3)

\[caption id="attachment\_24924" align="aligncenter" width="1920"\]![](images/Screen-Shot-2022-09-26-at-2.26.54-PM-2.png) left is modified, right is original\[/caption\]

Note that with low rel tol the original model latches when it isn't supposed to. With low rel tol the original model runs faster (see the number of points on the plots is less) and is also more accurate.

### high reltol (1e-6)

\[caption id="attachment\_24925" align="aligncenter" width="1920"\]![](images/Screen-Shot-2022-09-26-at-1.44.36-PM-2.png) left is modified, right is original\[/caption\]

Even with a higher timestep, the new model struggles less. Both models find a correct solution, however, the old model has artifacts than can be seen in the third pane on the right, the hotspot resistance jumps to 100x the value it is supposed to be at.
