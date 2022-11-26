---
title: "Simulating arbitrary 2 port devices using scattering parameters in LTspice"
date: "2022-11-10"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

After using the malicious circuits method of measuring stability and using the deconstructed SNSPD model to further test the SNSPD model, I was able to locate the main source of instability in the SNSPD model. The hotspot integrator is the main reason for the instability. While a circuit integrator is less stable than the built-in integrator in most cases, it is even more unstable when including a switch that shorts to ground. Luckily, LTspice has a built-in integrator with assertions to fix this instability. I.e. by running $sdt(f(t), u_0,  p&gt;0)$ if the condition $p&gt;0$ fails, then the integral valuation resets to $u_0$. I ended up making three changes:
<ul>
 	<li>expression for the hotspot growth voltage source from $\dfrac{v_{N3}+|v_{N3}|}{2}i_{B1}$ to an if statement over the values $\{v_{N3}i_{B1}, 0\}$ based on whether the wire is s/c</li>
 	<li>The hotspot integrator into a voltage source with the diff. eq. expression.</li>
 	<li>Removing the resistor in the state "boolean" circuit</li>
</ul>
This is a circuit side-by-side for the modified sections where the left image is the modified model and the right is the existing model:

<img class="aligncenter wp-image-24927 size-full" src="images/Screen-Shot-2022-09-26-at-2.42.02-PM.png" alt="" width="4112" height="2658" />

Here are two simulations run with two reltol values running a test of a bias current current of $15\mu$A for the first $10$ns with a photon incident at $2$ns followed by a bias of $25\mu$A.
<h3>default reltol (1e-3)</h3>
[caption id="attachment_24924" align="aligncenter" width="1920"]<img class="wp-image-24924 size-full" src="images/Screen-Shot-2022-09-26-at-2.26.54-PM-2.png" alt="" width="1920" height="1080" /> left is modified, right is original[/caption]

Note that with low rel tol the original model latches when it isn't supposed to. With low rel tol the original model runs faster (see the number of points on the plots is less) and is also more accurate.
<h3>high reltol (1e-6)</h3>
[caption id="attachment_24925" align="aligncenter" width="1920"]<img class="wp-image-24925 size-full" src="images/Screen-Shot-2022-09-26-at-1.44.36-PM-2.png" alt="" width="1920" height="1080" /> left is modified, right is original[/caption]
