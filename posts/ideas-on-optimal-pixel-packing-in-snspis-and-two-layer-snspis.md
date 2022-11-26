---
title: "Ideas on optimal pixel packing in SNSPIs and two-layer SNSPIs"
date: "2022-08-31"
categories: 
  - "electronics"
  - "general"
  - "superconductivity"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

I had some thoughts on SNSPI chip layout with regards to increasing their resolution using space-packing curves that I laid down, initially inspired by a bi-layer SNSPI operated similar to the TDC idea. This is mostly a dump of multiple ideas I had (and talked about with a bunch of QNN members), the most baked of these ideas is the section titled _Bi-Layer SNSPI Microstrip Architecture_. I actually found out after beginning this post about both of the [fractal](https://opg.optica.org/abstract.cfm?uri=QUANTUM-2022-QW3B.1) paper and the [thermally coupled row-column SNSPD paper](https://pubs.acs.org/doi/10.1021/acs.nanolett.0c00246) - both of which I think provide useful insights on thermal coupling and fabrication constraints if any of these ideas are to be considered more.

## Bi-Layer SNSPI

The main idea is have 2 SNSPI layers, a photon can interact with one of them and the other activates thermally. If it can interact with both layers, then this might increase detection accuracy, but I will be ignoring that interaction. In the simple case, the photon interacts with the top layer, which heats up and transfers the heat down the insulating stack to the second SNSPI layer, which will switch due to heating - similar to how TDC would work. The reason this whole post happened was I was trying to find an orthogonal basis of measurement, where we can increase the spatial resolution of a single SNSPI by extracting more info. Two exact stacks on top of each other will have the same spatial resolution given a hotspot formation - the second layer will most likely switch due to the thermal energy at the location of the first layer hotspot. A simple solution would be rotating the bottom stack by 90 degrees, which will increase the accuracy of each cell by some amount. The issue is, this is still constrained by our readout accuracy and also the improvements aren't necessarily linear on the entire stack - you can imagine certain locations on the stack not having a significant increase in accuracy due to the lower stack.

We can have two long meander SNSPIs on different layers oriented as rows and columns on the different stacks. Notice that the timing resolution limit is oriented along the wire, so in an infinitely long wire case, a photon incident on a wire has all of its position resolution error oriented along the wire direction. So having two perpendicular wires, the error spaces are orthogonal causing their intersection to be the point of incidence. This is the main idea behind this post, but I will also explore a discretized error scheme for bi-layer snspis.

## Discretized Bi-Layer SNSPI

For context regarding the upcoming explanation, I imagined the resolution length being bound to a square shaped region referencing a pixel (i.e. when discretizing the output to "pixels" using something like a TDC, every $p$-folds represent the same pixel when detected). Note: This greatly reduces how much improvement we can get from such a layout, but I thought it was worth thinking about and is generalizable enough to extend to the non-discretized case when $p<1$.

The solution I was exploring involves mapping every square $k\times k$ pixel to a rectangular equivalent pixel of dimensions $\sqrt{2} k, \dfrac{k}{\sqrt{2}}$. That way, two rectangles make a square when stacked and their rotation by 90 degrees is a square where each rectangle is "orthogonal" to the other two rectangles. You can imagine stacking them would result in 4 smaller squares ("logical pixels") of side length $ \dfrac{k}{\sqrt{2}}$ each. The meander folds are arbitrary in this illustration and I keep them generalized for this post.

![](images/packing_snspi_mapping_pixels-500x217.png)

We can then use an optimal packing curve such as a hilbert curve that maximizes the local and global resolution (more on that in the next section) and also find a curve who's pixels are orthogonal (i.e. for any pixel formed by the hilbert curve, every pixel formed by the orthogonal curve is a $\pi/2$ rotation of it). Here is an example of a curve and its orthogonal pair:

![](images/packing_snspi_curves-486x500.png)

A generalized hilbert curve's orthogonal pixel layout isn't necessarily a complete curve (maybe it is? not sure), i.e. in the above example, there are two separate curves in the orthogonal picture. In my representation, that is fine since you can connect the curves external to the active region, however, not sure if this will affect our ability to fabricate the device. The result on the left is a grid of smaller pixels where using the mutual information between the two layers gives you more accurate information on the location of the photon incidence.

![](images/packing_snspi_superposed-500x237.png)

For the rest of my analysis, I generalized the SNSPI meander as a $p$-fold per "pixel" to keep it generalized. Here is a visual example of how to construct a $13$-fold pixel layout on the orthogonal view below using by tracing each curve on its pixel layout:

![](images/packing_snspi_final-500x244.png)

## Generalized Hilbert Curve Packing

I haven't worked out a general algorithm for generating orthogonal hilbert curves, and didn't attempt to find a proof of existence (or not existence) of a complete hilbert curve orthogonal to its pair, even for the above example, it wasn't complete - I chose the first example I found joining two curves. One reason for choosing a space packing curve is to uniformly fill the space (not have unfilled gaps) in a mathematically consistent way regardless of the shape/size of sensor, which we can do using generalized hilbert (gilbert) curves. The other reason is once we design one unit of a cell, we can easily extend it using turtle/gdspy to a full meander layout without much work since these curves are well defined.

I will talk about the improvements gained by using a gilbert curve in two ways, local and global. Local refers to either one fold or the improvement in resolution of one fold due to a nearby fold. Global refers to the improvement across the entire device due to the overall layout of folds. I will talk about improvements in terms of the error being a photon hitting a position $(x, y)$, but causing an activation in a region $\epsilon(x, y)$ around the incident location, to quantify error on one axis $a$, I will call this $\epsilon\_a$. The idea is readout on incidence at position $(x, y)$ due to all possible errors/noise combinations will give you a readout in some epsilon ball around $(x, y)$.

### Local Improvements

I originally had an analysis for this section but decided to leave it out since (1) I am not content with the error bound on it, (2) it didn't match my intuition and (3) it had features that can't be fabricated. I also think a microstrip layout is more appropriate for such a device, and the way to properly calculate this now would be dependent on the microstrip layout constraints mentioned in the microstrip architecture section. I will probably revisit this in another post if I continue to work on optimal packing of SNSPIs.

### Global Improvements

For the usual SNSPI layout, the error along $\epsilon\_x$ is smooth, i.e. a small shift in position of incidence of a photon, causes a smooth small change in timing we can measure. However, a small error along $y$ makes a huge change in readout (i.e. if the photon moves vertically, it instead activates an adjacent wire, causing a much larger time difference than if that shift happened along the horizontal axis). This asymmetry is mitigated in the gilbert packing. The lower bound is higher than $\epsilon\_x$, but much lower than $\epsilon\_y$. On average, over the entire ball, the variance in error is lower, the mean is lower, but the minimum is higher than a standard layout SNSPI. Intuitively, most chunks are close to each other in a gilbert curve, so when you perturb either of $x$ or $y$ you get a smaller overall change in where on the flattened wire your photon hit.

## Smooth Space-Packing Curves

Instead of using gilbert curves, we can use round space filling curves (like in [fractal SNSPD](https://opg.optica.org/abstract.cfm?uri=QUANTUM-2022-QW3B.1) paper). The design in the fractal paper loses the global correction presented in the above section. However, it has less sensitivity to polarization. I also believe that it has less capacitive coupling with meanders since the capacitive coupling is distributed along tangents of close by circles instead of long parallel meanders. With fab constraints, I imagine that packing them optimally would cause them to be less dense than gilbert and regular SNSPI layouts. I haven't quantified any of these metrics in a meaningful way yet. Orthogonality in smooth curves is easier to achieve (at least in the examples I was messing around with) as they can be restricted to complete rotations. However, they seem less corrective than gilbert curves, the dot product of the error on all positions on the top layer isn't zero with all points on the lower layer, its only orthogonal at some points in the meander. Generating these meanders was hard for me to think about doing, both mathematically and in a fabrication sense - but this could be because I am still struggling with choosing a pattern that works.

## Bi-Layer SNSPI Microstrip Architecture

For this section, we will take a $p$-fold SNSPI where $p<1$, i.e. our time resolution allows us to resolve less than a length of one fold (think of a regular SNSPI architecture). The idea remains that we want two SNSPI layers with maximized thermal coupling and minimal capacitive coupling. One possible architecture is a superconductor-gnd-superconductor stack, that way the two s/c layers have almost no capacitive coupling. This stack can also provide good thermal coupling through the two dielectric (20um each) and a ground layer. Patterning repeated lines in a microstrip architecture shouldn't be much harder than a CPW geometry SNSPI if the microstrip array is tightly packed. Tight packing would increase the resolution but has negative effects on capacitive and thermal coupling. In terms of capacitive coupling, we have a coupling between every $1$-fold of the SNSPI, i.e. we can model an SNSPI with 10 folds as 10 transmission lines with capacitors between adjacent tlines in alternating $\hat x$ direction. This provides exponentially many pathways for a signal from point A to point B (now its a grid where the signal can traverse in any way as opposed to constraining the propagation along a single line). The minimal energy curve along the grid doesn't necessarily preserve timing delays, but there are two possible solutions I can think of:

1. Construct a fingerprint for a device structure, where every switching event causes multiple peaks and the spectrum of peak timing corresponds to different locations on the wire
2. Constrain the packing such that the cross-talk between the lines is minimized, this would impact resolution, hopefully not such that it is worse performing than a single layer CPW geometry. But this is something we can properly simulate using the Julia simulator and Sonnet.

I decided to go with the second solution since it seems easier on the readout end and I am not sure what the noise statistics on (1) would look like (noise is not symmetric on the edges of the grid and is propagated - not like noise on an fft spectrum which is smooth in frequency). I ran a toy example in LTspice using the following circuit:

\[caption id="attachment\_24754" align="aligncenter" width="1024"\]![](images/coupled-microstrip-toy-1024x605.png) 4 discrete tlines with 8 chunks each, each tline chunk is coupled with the adjacent tline chunk using a coupling capacitor whose capacitance is defined as a ratio to the capacitance of the chunk to the ground. In this circuit, the coupling capacitance is 10 times less than the capacitance to ground.\[/caption\]

With a meander capacitive coupling 10 times weaker than that that to ground, we get an okay amount of timing resolution between the nodes $g\_n$. For a pulse input, we can see how the peaks are separated by 8 discrete coupled chunks:

![](images/coupled-microstrip-pulse-1024x662.png)

And for a step input (more like the signal we will get from an SNSPI switching):

![](images/coupled_microstip_on-1024x662.png)

We can directly compare the coupled (10 times less than to ground) and uncoupled meanders along $g\_2$ and the output node.

![](images/Screen-Shot-2022-08-31-at-11.11.39-AM-1024x604.png)

The addition of capacitive coupling between meanders slows down the pulse (hence increases the timing resolution) and is consistent between the different folds (the spacing between $g\_i$ and $g\_{i+1}$ is the same along the rising edge - inclusion of capacitive coupling adds a constant skew rate between every $g\_i$ node, therefore lumped into the propagation velocity, causing a slower speed). So a readout that detects switching above anywhere on the rising edge should resolve timing without errors introduced between the coupling provided it is small enough. Using sonnet, we can simulate and find the capacitive coupling between the meander and the ground by considering the worst possible scenario where a single fold views all the folds on each side as one giant piece of metal.

\[caption id="attachment\_24761" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-31-at-11.24.09-AM-500x122.png) Showing 2 microstrip architectures sandwiching a ground. The bottom tline is perpendicular to the top tline. All contributions of the left and right metal is assumed to be one infinite plate as an upper bound on the capacitive coupling between the meander and itself compared to the coupling with ground. The grey lines showcase the capacitive couplings we care about.\[/caption\]

The thickness of the dielectric (+ width of the fold) constrains the ground capacitance. The thickness of the ground layer constrains the thermal coupling of the two SNSPIs. The meander bend radius is constrained by the ground capacitance. And the bend radius also constrains the thermal coupling in the meander. I haven't quite thought about how this affects the impedance of the line for this analysis.

## SNSPD shunted onto a TDC

Another possible design could be laying out a line of biased SNSPDs and use the switching's thermal energy to switch a tline. This can also double as the time-to-digital converter readout too.
