---
title: "Large Scale Simulations in LTSpice"
date: "2022-09-14"
categories: 
  - "general"
  - "snspd"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

This all started with noticing a ton of artifacts in the SNSPD model when performing critical current sweeps while developing a new spice-daemon feature to generate IV curves. One big dragging point for all the simulations we run is its lack of scalability and while the idea isn't to use LTSpice for large scale simulation in the long term, this provides some information on how to do it in LTspice and is hopefully generalizable to future solvers.

This post is structured in the following manner:

1. Simulation artifacts in the current SNSPD model
2. Simulation stability and malicious simulations in LTSpice
3. Janky intuition for why the SNSPD model breaks
4. Differential equations in LTSpice
5. Tracking state in LTSpice
6. Proposed SNSPD modified models
7. "Dumbification" of the SNSPD

## 1\. Simulation artifacts in the current SNSPD model

The artifacts can be seen in the most stable SNSPD model configuration with a parallel readout resistor and a bias current. By sweeping the bias current, we can note that for some of the currents end up causing extra excitations. These excitations are an artifact of the time stepping algorithm which I will discuss in more depth in section 2.

\[caption id="attachment\_24852" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-22-at-2.19.53-PM-1024x662.png) Running a transient simulation stepping through a different bias current value every simulation. The switching current of the device is overwritten to be 20uA and the step size is increased to 1e-14.\[/caption\]

To make sure that it isn't the step directive that is causing this behavior due to some form of operating point solution sharing (even though there shouldn't be any), we can demonstrate that a faulty bias current solution exists in a regular simulation too.

![](images/Screen-Shot-2022-08-22-at-2.22.40-PM-1024x662.png)

 

In this, we can see that with a nice current source value of $13\mu$A, we can still see the artifacts, so it isn't the stepping command. This also gives us a clearer view of what is happening in the faulty simulation.

There are multiple reasons this isn't "real physics" and is an artifact. We have one photon hitting a single segment wire at time 20ns, using an energy argument, the highest energy the nanowire will have is at 20ns+$\tau$. However, when the hotspot starts cooling down we see new switching events, meaning that more energy must be pumped back into that segment of the wire. Given that the simulation setup (without the SNSPD) has no memory (no inductors, capacitors, state machines, switches, etc.) there is no way for the energy to be pumped back into a segment of wire at a later time due to the linearity of the IVP. Another reason is the peaks have different heights, in a setup with no memory, the peaks of a SNSPD switching must be the same regardless of the time of activation, time since activation and energy pumped in. Another couple of factors on why this isn't physics is long term reactivation and pre-activation, in that I was able to construct malicious examples where you see the SNSPD switching before a photon hits the wire and a long time after the photon hits in a memoryless setup.

\[caption id="attachment\_24854" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-22-at-7.00.56-PM-1024x662.png) Malicious simulation causing a wire switch more than 50ns after the original pulse in a setup with no memory.\[/caption\]

 

We know that these aren't physics and they don't seem to be behavior governed by the model if it were simulated as expected, however, we can also check that. By changing the LTSpice default simulator to using a Gear method, the artifact is gone. Also by varying the simulation parameters in LTspice, such as vtol, trtol, etc. we can engineer the peaks to happen at different times, remove some and add some - we can also find a set of parameters for every simulation where it would converge correctly, more on that in section 2.

What's even worse, is these time stepping bugs are inconsistent between Mac and Windows installations of LTspice. While regular SPICE defines trtol to be 7, LTspice chose it to be 1 since it "behaves better" for SMPS macromodels. However, for some reason they were inconsistent with that choice causing it to default to 2 on Mac. Which means that artifacts are inconsistent between Windows and Mac unless if you double check to make sure trtol matches.

## 2\. Simulation stability and malicious simulations in LTSpice

This section discusses where the instability comes from in LTspice through the concept of malicious simulations. The idea behind a malicious simulation is overloading parts of the simulation program so that they function in a manner you can control. The idea lies behind exploiting implementation features to find issues with the ways we solve differential equations by introducing circuit elements that shouldn't affect the output of the circuit but end up doing so. For instance, in LTspice you can't finely control time stepping through simulation arguments or the tran directive, however, by introducing a highly non-linear element at a time $t\_n$, you can cause the simulator to spend more time evaluating the overall IVP around $t\_n$ by changing the ratio of successful and failing iteration steps. If we have two circuits $C$ and $M$ that share no nodes (or information between them), the outputs in circuit $C$ when $C$ is simulated aren't necessarily the same as when $C+M$ are simulated. From an information perspective, $I(C+M)=I(C)+I(M)$ in the differential equation, however, since there are shared variables in the LTspice implementation, $I(C+M)=I(C)+I(M)+\Delta$, where $\Delta$ is some mutual information. By choosing different $M$s, we can analyze the stability of solving $C$. We test the convergence in simulating a circuit $C$ with our element $e$ by checking that simulating $C+M$ is consistent with simulating $C+M'$ ($M, M'$ are any two circuits that don't share nodes or elements with $C$ and don't contain $e$), by the Lax-Richtmyer equivalence theorem, this shows the models ability to converge for all topologically valid circuits with $e$ in them and that the model's $\Delta\xrightarrow{} 0$.

\[caption id="attachment\_24896" align="aligncenter" width="1024"\]![](images/malicious_circuit-1024x562.png) Two possible malicious circuits $M\_i$ for a test circuit $C$. Circuit $M\_1$ is weakly coupled to $C$ and circuit $M\_2$ is completely decoupled from $C$. However, the implementation of the solver allows for time coupling of the two circuits and allows for the exchange of information that introduces simulation transients.\[/caption\]

While this is exploiting the fact we are choosing specific decoupled circuits $\{M\}$ that break $C$, this is important for finding cases where elements in $C$ would misbehave without exhausting all possible combinations of circuits with each element in $C$. Using this we can back-solve and find the IVP setups that break the nanowire model for example. This is even more important when we are trying to simulate large ensembles of an element, such as hundreds of thousands of nanowire models. Especially given that timing-based instability such as the nanowire example demonstrated above look enough like physics to the extent that 1 nanowire with 1 photon hitting it was convincing enough that it was an actual output. Once this output gets distorted, manipulated, multiple photons start interacting and thousands of elements are present, there is no way of telling whether an element is misbehaving since we know that $C+C$ doesn't necessarily simulate $C$ in the same fashion and by extension larger networks might have different instabilities than a single element, making it impossible to know what is misbehaving.

One quick workaround for the broken nanowire model is changing reltol to 1e-06. I am not content with calling this a fix because it still showcases the same artifacts as before, its just that every malicious $M$ causes less parametrizations of $C$ to break. So it isn't necessarily that the model is being simulated better, but the fact that there are fewer broken cases in small sized circuits and therefore it is harder to find them when working at a small scale. Once we have exponentially many elements it will be much harder to keep track of small physics-like errors. The other reason I am not too happy with this workaround is a reltol of 1e-3 should be enough to cover a hotspot activation from $10^{-3}\Omega \xrightarrow{} 1\Omega$, which is larger than any transition that should happen in the nanowire models resistance at a singular timestep. This means that this implementation for some reason has a limit to resolution that will make simulating larger networks harder. I think this workaround is fine for now, but we should consider a more stable model for larger networks.

## 3\. Janky intuition for why the SNSPD model breaks

It is hard to exactly say what a malicious circuit $M$ does to $C$ that breaks it. In the case of the snspd model, adding unrelated non-linearities causes the spikes to form. The initial spike is almost always never the "correct" height it should be at, usually either overshoots or undershoots. Then when a timestep correction happens the integrators charge is restored to what it should've been before the switching event since it never was supposed to cause a trigger. This is purely a function of the time stepping in the modified trapz method. So when you include a highly non-linear element such a spike current source at around the time a SNSPD photon happens, this unstability occurs. Note that the photon input is also a non-linear event, which would also explain why timestepping sometimes fails in the SNSPD model in the default circuit. The corrective function of the capacitor causes it to retrigger multiple times. Pre-firing occurs when it overshoots by a huge margin in the beginning. Very delayed firing happens because of undershooting for a long time. Usually the last or second to last peak is the correct height for a trigger which often signifies there was no over or under shooting. We can see the stuck at undershooting event happen when our reltol is really high as in here:

![](images/Screen-Shot-2022-08-22-at-7.10.09-PM-1024x662.png)

We know this isn't a switching event that gets stuck due to sinusoidal oscillations that can form of different periods in a memoryless DC biased network. This is another undershoot example where a lower current also gets stuck. This is supposed to signify it is not switching that causes this but in fact it is the instability since a memoryless circuit switched and "latched" (not really latching becasue of the sinusoidal behavior) at a lower current.

![](images/Screen-Shot-2022-08-22-at-11.27.08-AM-1024x662.png)

This is an example of an SNSPD pre-firing way before the trigger photon in the purple trace.

![](images/Screen-Shot-2022-08-22-at-12.39.27-PM-1024x662.png)

## 4\. Differential equations in LTSpice

LTspice has a bunch of cool built-in functions that aren't very well known. This section will discuss the functions sdt(), ddt() and idtmod() all of which are Verilog-A compatible. sdt($f$\[, $c$, $q$\]) is an integral over time, it takes in the function instance to integrate $f$, an initial constant $c$ and an assert condition $e$. If $e\neq 0$, then the integrator resets to $c$ and starts accumulating again. ddt($f$) is a time derivative and takes in the function instance to derive $f$. idtmod($f$\[, $c$, $q$, $A$) is similar to idt(...) where it takes in the function instance to integrate $f$, an initial constant $c$, a modulo $q$ and an offset $A$ (its equivalent to $A+$idt($f$, $c$, NODE>=$q$).

For the nanowire, to include a thermal model, we can use a behavioral voltage source to model the differential equation $\dfrac{dT}{dt}=-a(T-T\_S)+b i\_{hs} v\_{hs}$ with an initial condition $T(t=0)=T\_s=2K$ as follows:

![](images/Screen-Shot-2022-09-13-at-11.42.00-PM-500x137.png)

This treatment makes the models less susceptible to malicious circuits since the time stepping solver treats internal functions differently than charging and discharging capacitors and inductors. This helps decouple the treatment of the circuit solver to additional logic which is also simulated in LTspice.

Note that this can all be made pretty using the .func directive. In LTspice, you can define functions you can use as follows:

`.func func1() 2*pow(time(), 2) .func func2(scale) scale*pow(time(), scale)`

.func Rhotspot\_dyn() IF(V(photon\_in)>0, + IF(buf(ddt(V(photon\_in))), + V(photon\_in), + exp(7\*V(photon\_in))), + 0)

Another option is using the table built-in function. Unlike functions, tables are restricted to one input parameter, i.e. `table(input_var, input_val_1, output_val_1, input_val_2, output_val_2, ...)`. Using this you can precompute differential equations that are too hard to model or recreate in LTspice using more powerful tools. There isn't a default way to import a file directly into a table without listing all the values, one possible workaround I haven't tried is using the .include spice directive which I suspect with the right escaping can put an entire properly formatted file into the table function. If a value is between two input values, it will linearly interpolate the output from the two inputs.

Note that it is possible to have a 2 parameter table by applying a mapping carefully from $(x, y) \xrightarrow g\_x x + g\_y y$ using two generators. The choice of generators is important to preserve the linearity of interpolation but in theory an interpolant is possible and for discrete cases, this isn't an issue. One way of relaxing the constraints on the generator is by forcing the discretization of inputs using the ceil/floor function alongside some input generator and a finite table with a 1:1 match:query ratio and no interpolation costs.

If your differential equation is over time and linear, then you could also use a PWL source and the trigger option to play data whenever a certain event happens. For example, you can trigger an SNSPD spike whenever the current hits a certain value, with that method, you can control the height of the peak but not the decay time within LTspice. You can change the decay time using something like spice-daemon - but this isn't implemented yet.

## 5\. Tracking state in LTSpice

The most basic state tracking in LTspice is usually in the form of floating voltage nodes. In the current snspd model, the state is tracked using switches. One other option is using the .machine directive in LTspice. It is a powerful directive in that you can encode any DFA/NFA (finite automaton) into it. There is virtually no documentation about it except for one example on the internet. From experimenting with it, here are the highlights of how it operates in LTspice.

1. Start and end your machine with the .machine directive and .endmachine directive. Defining all the states, transitions and outputs happens inside these two commands.
2. Define your states using this syntax `.state name value`. Every state has an associated value which is useful for outputting.
3. The first state you define is the initial state your automaton will start with.
4. Define the state transitions using `.rule from_name to_name condition`. If you have external variables in the condition, such as params, you need to put them in between curly braces e.g. ${I\_c}$. Not necessary for voltage and current nodes.
5. To output a state value, use the `.output (output_node) state` to output a state onto a certain node (with the brackets around output\_node!) and the word state is a keyword!. You can also output other things such as expressions by replacing the word state with an expression. Here the value you defined for a state is what will be output.

Here is how you could swap out the two voltage switches in the SNSPD Basic Curve Fit model using the .machine directive:

`R1 vs_state 0 1 .machine .state vs 0 .state vs1 1 .state nvs 1`

`.rule vs0 vs1  V(N2,source) >  {Vthresh}` `.rule vs0 nvs1 V(N2,source) < -{Vthresh}` `.rule vs1  vs0 V(N2,source) <  {Vthresh}` `.rule vs0 nvs1 V(N2,source) > -{Vthresh}`

`.output (vs_state) state` `.endmachine`

Now the voltage at node vs\_state will be 0 if the wire is s/c and 1 if it switched, internal to the machine, it stores whether the wire went above or below the threshold. Note that the transitions to and from the normal state are separate and therefore you can encode retrapping in a simpler fashion with this setup. Note that if a state machine output is connected to a pin, current monitoring for the pin will not be correct

This is a better way of state tracking than using switches. You can then have a resistor and an IF statement based on vs\_state if you need an actual switch.

What this means for the solver is it can apply less timestep guesses on the network, which is both a good and a bad thing. But the matrices are smaller for each instance and the number of operations per SNSPD is going to be less. This means that a given simulation will have less operations per iteration but more likely to have more iterations. This is a weird tradeoff that I evaluated experimentally (for both the i and v circuits). For 8-40 snspd networks, the use of the state machine achieves between a 20-75% speed-up. Larger than that the matrix compiler has a harder time optimizing both, but you end up with an average of 15-25%. Depending on the network topology these are very variable, but in general this small change can be expected to yield about a 20% speed increase. Optimizing the model further can yield up to a 90% speed-up of the BCF model.

Some useful commands for this are buf(x) and inv(x), which test whether $x>0.5$ or $x<0.5$ respectively.

## 6\. Proposed SNSPD modified models

Remove dependence on switches completely. Replace with machine if it requires complex state storage like the BCS model. In the dynamic model the switch is for the integrator which can be replaced by the built-in integrator with a conditional reset instead of a resistor shorting to ground. For the BCS model, some nodes can be removed and merge the actions of multiple components such as the switches and resistor. We can also remove the sensor voltage source. For the dynamic model the entire integrator circuit is removed. This actually fixes a lot of the convergence issues. Another possible modification would be the way photon incidence is handled but this won't speed things a lot at the current stage of the SNSPD model. With these, the dynamic model can be sped up to 30% in a setup with 272 SNSPDs. These modifications don't change the way the SNSPD model functions, we can check that by seeing that simulations output similar results between both models and their modified counterparts.

## 7\. "Dumbification" of the SNSPD

This is mostly inspired by discussions with SPICE subgroup, Cameron and Reed. For a large model, we can trade off the accuracy of individual features such as the thermal evolution of a hotspot to a precomputed evolution. This evolution can be modified on the fly but is produced by heuristics rather than solving a differential equation. For example, I replicated the behavior of the SNSPD using a resistor using multiple methods and all of which provided a huge speed improvement. The seemingly most complex and most parametrizable of which is about 100x faster than the regular model, with similar accuracy and behavior to when the SNSPD functions and is less prone to malicious circuits.

![](images/Screen-Shot-2022-09-13-at-9.46.09-PM-1024x662.png)

This example setup shows the growth of Rhotspot\_dyn() scaled to the dynamics of the actual SNSPD. Funnily enough, the SNSPD model was misfunctioning in this case while we can see the simpler resistor based model was working fine. The yellow trace is the original SNSPD models readout, while the green pulses are the new resistor based SNSPD models readout resistor. The number of non-linearities in the circuit caused the SNSPD to trip and have multiple peaks.

There is another resistor based thermal model which isn't fully encapsulating of the thermal model of an SNSPD as of now, but it also demonstrates the ability to simulate the entire SNSPD dynamics using only resistors and built-in functions.

Another method of building this is having a triggered PWL source, however, this greatly affects the time stepping algorithm by setting a minimum resolution on it related to the number of samples in the signal but also adds the interpolation cost of finding in between time steps.

One other method I tried was building this is using tables. This also suffers from the problem of interpolation cost. Overall I would recommend heuristic parametrization over precomputing tables. However, table precomputing might be a good option for non-time based data, such as setting an Ic based on temperature for a heuristic model, allowing us to encapsulate complicated dynamics into a simple resistor model.

## Conclusion

The resistor model is promising, and with the addition of something like spice-daemon, we could overpower precomputation to model it more realistically on a much larger scale. The optimizations applied were 2 days of work and looking into a new concept with no much prior knowledge, so I am sure there are many more optimizations that can speed up performance and scalability of these models. These changes are also completely compatible with Xyce and should be easily translatable to the julia I was working on if I choose to continue work on it. Possible extensions of these findings are faster tapers and a dumbified ntron models for LTspice. These findings don't matter on the small scale, but once you throw in a couple of tapers you can see the simulator slow down tremendously. Wanting to simulating thousands of devices in an efficient manner should be a short-term goal for our model designs if we plan on scaling to hundreds of thousands to millions in the near future.

In short, use LTspice diff and sdt instead of circuit integrators if you can, they are more stable and inflict less operations, especially the more complicated the network is. If you are using the old snspd model, make sure to use reltol of 1e-6. If you are seeing different transients than someone else in the lab, make sure your settings (even the default ones are different!) are the same, especially trtol. Avoid switches, they are heavy on the computation side, replace with IF statements if possible. If you can't use IF statements, try using a state machine using the machine directive.
