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

This post will demonstrate a method and an accompanying script that converts any passive or active linear 2-port device into a constant sized model with $9$ total nodes using its S-parameters. The code, examples and explanation circuits are in this repo: <a href="https://github.com/itstorque/spice-arb2port">https://github.com/itstorque/spice-arb2port</a>.
<h2>Why I pursued this in the first place?</h2>
In my simulator I precompute an entire taper using its $S_{xy}$ parameters. Assuming linearity of the device over $i, v$ for a given $f$, we can use this $O(1)$ model in a context outside of the Julia simulator without built-in optimizations that handle that. I was particularly interested in simulating a taper with $N$ chunks in LTspice using a constant sized node count allowing for precomputation of characteristics outside of LTspice. A more ambitious goal is simulating arbitrary 2 port linear passive device in LTspice.

From basic circuit theory, any 2 port RLC network can be written as a laplacian (and due to generality of the form, you can include any passive or active components into your network). Since the S-parameters can describe the behavior of a device, we can imagine there must be a transform from S-parameters to the Laplacian. Since LTspice has a Laplacian property for some elements, you can imagine having a resistor have a laplacian that describes its behavior. If you were to do that using RLCs, you would need to define each of the linear components from their basis directive, i.e. set the resistance (capacitance/inductance) to <code>R=1 Laplace=...</code> (<code>Q=x Laplace=...</code>/<code>Flux=x Laplace=...</code>). Then you can post process the S parameters and extract this information into each element, giving you a ~16 element model. This can be done with less elements (one complex resistor instead of RLC for each S-parameter) however LTspice doesn't support complex numbers. It isn't directly obvious how to implement this, however, it seemed possible.

What encouraged me to pursue this is we can notice any 2 port network has 4 entries in its S-parameter matrix. This means it has 4 complex degree of freedom, or 8 real degrees of freedom. The S-parameters also include an extra definition of loss/gain (1-transmissions-reflections) and also has a reference impedance (usually 50 ohms) that isn't encoded into the matrix. We can introduce a 9th degree of freedom for the reference impedance and use it to inject energy into and out of the circuit. Then you can imagine that with a total of 9 nodes (one of which is a ground), we can encode any 2 port network theoretically.
<h2>The encoded circuit</h2>
The resulting circuit has 9 nodes, 4 resistors and 4 voltage-dependent frequency-dependent sources. For this weird encoding, we use an E-source.
<blockquote>A digression on sources in SPICE in general. SPICE started out with two types of sources, V and I (independent voltage and current) sources. Later (early) iterations included many more sources:
<ul>
 	<li>E sources: Voltage Dependent Voltage Source</li>
 	<li>G sources: Voltage Dependent Current Source</li>
 	<li>F sources: Linear Current Dependent Current Source</li>
 	<li>H sources: Linear Current Dependent Voltage Source</li>
</ul>
These sources have a defined way of operating through proportionality. Eventually, some sources became more complicated and started supporting the <code>POLY</code> command for analog behavioral modeling. As SPICE software became more advanced, B sources (behavioral sources) were introduced which can have arbitrary uncoupled inputs and a non-trivial function over them to control the output. Fun fact about B sources: current and voltage B sources are the same component. Also B sources have two largely undocumented modes of operation BP and BR (behavioral power source and behavioral resistor). I won't go into more weird sources in LTspice in this post, but there are very interesting tiny differences. In this post we use an E source because of a cool feature in the way LTspice lets us model frequency dependent behaviour.</blockquote>
E-sources have 2 pairs of ports, the basic syntax describes that in the following manner: <code>Ex node+ node- control+ control- &lt;gain&gt;</code>. The syntax we will be using hinges on a special mode of operation of E sources:

<code>Ex node+ node- FREQ {{V(control+, control-)}}= DB
+ (freq1, mag1, phase1)
+ (freq2, mag2, phase2)
...</code>

Where the magnitudes are in dB and the tuples map a given frequency to a magnitude-phase pair. By choosing the right geometry, we can imagine that 4 E-sources on the 9 nodes can loosely define our circuit (the addition of resistors stiffens it). Each E-source corresponds to a certain S parameter entry in the matrix. Here is what the circuit topology we are building looks like:

<img class="aligncenter wp-image-25005 size-full" src="images/Screenshot-2022-11-10-at-1.55.53-AM.png" alt="" width="3326" height="1142" />

<img class="aligncenter wp-image-25006 size-full" src="images/Screenshot-2022-11-10-at-1.56.39-AM.png" alt="" width="2004" height="1712" />

These two circuits are equivalent, the top one is easier to visualize at first, but I find the folded circuit to be easier to debug with when intuiting about it.
<h2>How to generate an example</h2>
The script in the repository generates S parameters from a taper using scikit-rf, you can either generate a circuit in scikit-rf or put in your experimental/calculated S-parameters into the 4 S arrays as well as the corresponding frequencies in <code>freqs</code>. Running the script updates the lib file and by dragging the symbol into a circuit in the same folder you should have a working 2 port model for the circuit you measured/calculated. The code encodes the magnitude (in dB) and phase of each S parameter into the long list of frequency dependent behavior so that LTspice can use that when using the .tran or .ac command.
<h2>Testing using the .ac and .net directive</h2>
To make sure the frequency dependent behavior is correct, you can measure the S parameters inside LTspice of a device. Place your 2 port symbol in series with a voltage source before it <strong>with an AC amplitude of 1 </strong>and a 50 ohm resistor after it (lets call them V1 and R1). Change the series resistance of the voltage source to 50 ohms (this is important).

<img class="aligncenter size-large wp-image-25007" src="images/Screenshot-2022-11-10-at-2.12.53-AM-1024x519.png" alt="" width="1024" height="519" />

Now you need to add two spice directives:
<ul>
 	<li><code>.ac &lt;scale&gt; &lt;lower freq&gt; &lt;freq step size&gt; &lt;max freq&gt;</code> for instance you can do <code>.ac oct 10K 1K 100Mega</code> to measure the AC response of the circuit between 10KHz and 100MHz along an octave.</li>
 	<li><code>.net i(R1) V1</code> which generates a network analysis of the device between V1 and R1. There are ways to change the impedances you are measuring against, however, if you set V1's series resistance to 50 ohm and R1 to 50 ohm, LTspice is smart and deduces that you want a 50 ohm analysis. Note that this doesn't always hold, so if you want to do something more exotic make sure to research the net directive.</li>
</ul>
After placing these two directives and running a simulation, you should be able to right click on the plot and click "add trace" where you will have many more options than usual that include the Z and S parameters of the device (for example S11(v1) for the S11 parameters of the 2 port device). This should reproduce your input S parameters to verify the correct operation and simulation of the device in LTspice.
<h2>Exponential Taper Example</h2>
Using scikit-rf, I generated some randomly parametrized exponential taper to verify the correct operation of the model. We can see we get the following S21 and S11 plot:

<img class="alignnone wp-image-25008 size-full" src="images/Screenshot-2022-11-10-at-2.15.20-AM.png" alt="" width="2372" height="1388" />

This roughly resembles an exponential taper... yay :))

The taper parameters were:<span style="font-size: 0.4em;line-height: 0.4em;font-style: monospace">
<span style="color: #808080">freq = rf.Frequency(20, 100000, unit='MHz', npoints=10000)</span><span style="color: #808080">
w1 = 20*rf.mil # conductor width [m]</span><span style="color: #808080">
w2 = 90*rf.mil # conductor width [m]</span><span style="color: #808080">
h = 20*rf.mil # dielectric thickness [m]</span><span style="color: #808080">
t = 0.7*rf.mil # conductor thickness [m]</span><span style="color: #808080">
rho = 1.724138e-8 # Copper resistivity [Ohm.m]</span><span style="color: #808080">
ep_r = 10 # dielectric relative permittivity</span><span style="color: #808080">
rough = 1e-6 # conductor RMS roughtness [m]</span><span style="color: #808080">
taper_length = 200*rf.mil # [m]</span></span><span style="font-size: 0.4em;line-height: 0.4em;font-style: monospace"><span style="color: #808080">
taper_exp = rf.taper.Exponential(med=MLine, param='w', start=w1, stop=w2,</span><span style="color: #808080">
length=taper_length, n_sections=50,</span><span style="color: #808080">
med_kw={'frequency': freq, 'h': h, 't':t, 'ep_r': ep_r,</span><span style="color: #808080">
'rough': rough, 'rho': rho}).network</span>
</span>
<h2>Why is this useful + future integration</h2>
This is particularly useful as it converts an <code>N</code> sized model to an <code>O(1)</code> model. Usually a taper can be simulated using thousands of inductors and capacitors or transmission lines (ideally a lossy tline using convolutions instead of a lossless one...). This provides multiple kinds of speed-ups.
<ul>
 	<li>First of all, large netlists tax the topology checker and optimizer and as a result you get less performant circuits and topology errors that might slide under the rug. A 2 port model like that adds a constant sized number of nodes (and therefore calculations) regardless of the length of the taper or number of chunks making it easy for the checker to handle.</li>
 	<li>Second of all, instead of calculating the voltages across each node in a taper, now you only calculate it across a constant number of nodes, speeding the computation up dramatically.</li>
 	<li>Convergence across a constant sized number of nodes is much easier than a linear number of nodes. As a result you get better and faster convergence.</li>
 	<li>Computation of the frequency dependence uses logical lines which has an inherent speed up under the hood compared to individual separable RLC contributions for each element.</li>
</ul>
If simulating a device with a taper in less than an hour is not convincing enough, some other advantages to this way of modeling include the ability to plug in the exact electrical characteristics of equipment in the lab into your SPICE simulation and getting better simulation results since unfortunately chaining Laplace functions in LTspice is not very reliable (in fact any Laplace directive is not too reliable).

Next steps to improving this would be having a better spec on what frequencies must be sampled to get an optimal performance in both the ac and tran simulation environment. It would also be nice to characterize some devices from the lab into LTspice and see how SNSPD readout behaves under it - an extension to the work done by the SPICE sub-team.

One side note is I hand-wavily expect that scaling this to $n$-ports would be exponential in $n$ which is not terrible since it is still constant in size. We can then model devices like a bias tee from provided S-parameters.

I think generalizing this for non-linear devices is possible if this is to be migrated to a much more involved b-source which would be a little more involved than using g-sources.