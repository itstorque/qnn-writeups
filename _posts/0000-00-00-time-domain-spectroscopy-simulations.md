---
title: "Time Domain Reflectometry Simulations (originally Time Domain Spectroscopy)"
date: "2022-08-03"
categories: 
  - "general"
  - "superconductivity"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

This post discusses (1) an overview of the simulation model used, (2) an optimal search technique for time domain spectroscopy and (3) some simulation results showcasing time domain spectroscopy of different wires with and without defects.

## Time Domain Spectroscopy

Usually $i\_c$ measurements tell us about the worst defect in a wire. Instead of sending in a current $i\_1$ from one port and searching for the threshold where $i\_1=i\_c$ to determine the worst critical current, we can send in two current pulses $i\_1, i\_2$ from the two ends of the wire to locally test if $i\_1+i\_2=i\_c(x)$ at every position $x$ in the wire, thus allowing us to build a spectrum along the entire wire of what the $i\_c$ at each chunk is.

\[caption id="attachment\_24499" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-02-at-10.47.23-PM-500x330.png) Example of a taper with 3 defects causing dips in the $I\_c$ curve. Two voltage pulses of opposite magnitudes are sent in from two sides parametrized by 3 parameters (two magnitudes and one parameter for the phase difference)\[/caption\]

I anticipate that exterior control on pulse timing is going to be more accurate than SNSPI read-out, although using both can lead to improvements discussed in the optimal search section. The simulation results also suggest that we only care to control the pulse timing (phase offset of two pulses) and their magnitudes, and not so much their rise and fall time or their minimum pulse size.

## Simulation Model

This model is based on a time-domain model designed and optimized in Julia using DifferentialEquations.jl (post on simulator coming soon). The simulator models any superconducting wire as a discrete transmission line made of RLC chunks, where $R=0$  in the s/c state and $R=R(i, t)$ after switching/hotspot formation.

The simulator's linear form was used. The simulator analytically calculates the propagation delay $t\_d$ and runs a simulation for $2t\_d$. Opposite magnitude voltage pulses are sent in from both ends of the wire (opposite voltage pulses traveling in opposite directions have their current components traveling in the same direction). The two pulses are parametrized by $i\_1$, $i\_2$ and $\tau$, such that the two inputs are $i\_1\delta\_1(t)$ and $i\_2\delta\_2(t-\tau)$, where $\delta$ represents an input function such as a single $\sin^2$ pulse, gaussian pulse or step function.

\[caption id="attachment\_24500" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-02-at-10.52.06-PM-500x418.png) The propagation of two pulses sent into a wire from both ends, offset by $\tau$. We can vary $i\_1$ and $i\_2$ until the wire switches. If it switches with both sources on, then we know that the wire switched at the position that the position $\approx \tau t\_d$, where $t\_d$ is the propagation delay.\[/caption\]

The model is constantly being checked for a smooth transition into the superconducting state by interpolating over sign changes in $(i\_k - i\_c) \; \forall \; k$ chunks in the wire. When the wire is detected to switch, the simulation is halted and we assume that we have a way to read that out (experimentally, a bias current will cause two voltage pulses to form where the wire switched).

## Optimal Search

Our parametrization has 3 variables, $i\_1, i\_2$ and $\tau$, leading us to a 3 dimensional space, where we care about a 2D projection of this space (at a single position $x$, the critical current at that position $i\_c(x)$ forms a 2D real number space for any combination of two current magnitudes). Define $\alpha$ to be the position at which the wire has the lowest $i\_c$ (this is where the biggest constriction/defect probably is), such that $i\_c = i\_c(\alpha)$. We can notice first that the phase lag $\tau$ directly corresponds to a translation in $x$ with some uncertainty (a transform $f\_X: \tau \xrightarrow{} X$ exists) - a phase lag causes the intersection of the two pulses to change in position.

We are left with $i\_1$ and $i\_2$ which encodes some partial information on both $i\_c(x=X)$ and $i\_c(\alpha)$. We have a $\mathbb{R}^2 \xrightarrow{}\mathbb{Z}\_2$ map ($i\_1\times i\_2 \xrightarrow{} \{ \texttt{switch}, \texttt{no switch}\}$ that promises to be monotonic - there are no superconducting state bubbles for some $i > i\_c$, i.e. the critical surface bounds the entire superconducting state. This gives us a 2D space we could trivially search using gradient descent and we will always converge since there is only one true local minimum. The two functions we are trying to optimize are remaining in the superconducting state and maximizing $i\_1+i\_2$. We can also see from this that the maximum values of $i\_1 + i\_2$ encodes the most information on $i\_c(x=X)$ (as opposed to being clipped by $i\_c(\alpha)$, you can prove that you recover the most information on $i\_c(x=X)$ when $i\_1+i\_2>2i\_c(\alpha)$).

\[caption id="attachment\_24502" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-02-at-10.58.52-PM-500x327.png) An arbitrary superconducting region inside a wire where the outer path of the blue region represents a critical path in terms of $i\_1$ and $i\_2$ along the entire wire. Maximizing $i\_1+i\_2$ while being on that surface would correspond to $i\_c(x=X)$.\[/caption\]

Since we know the $i\_1\times i\_2 \xrightarrow{} \mathbb{Z}\_2$ map is monotonic, we also know that the boundary is smooth. We can traverse the critical surface and perform an $O(N)$ search on this 2D space using path constriction. Since there are no bubbles due to monoticity, we also know that there is only one critical boundary. The maximum value of $i\_1+i\_2$ on that critical path corresponds to partial information about $i\_c(x=X)$.

We can further show that for the case where the only readout value is whether the wire switched or not, we have a symmetry around $x=X$. I.e. we can't tell whether a pulse sent in from the left of the wire ends up switching the wire to the left or right of the hotspot. This means the $i\_1 \times i\_2$ plane is symmetric about $i\_1=i\_2$ and therefore $i\_c(\alpha)$ (the wire will always switch when either of the current pulses are at or above $i\_c$) forms a box that bounds the information on both sides. We also know that any $(i\_1, i\_2)$ pair in the superconducting state must lie below $i\_1+i\_2=i\_c(x=X)$, therefore we also have a boundary line $i\_1+i\_2=i\_c(x)$. This means that the superconducting region is spanned by the intersection of a triangle and a square, and that the maximum information we learn about $i\_c(x)$ can be extracted on the line $i\_1+i\_2=i\_c(x=X)$, in other words, we have a 1D monotonic search space trying to optimize $i=i\_1=i\_2$, therefore we can perform binary search to find the result in $O(\log N)$ time.

\[caption id="attachment\_24504" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-02-at-11.25.12-PM-500x435.png) The superconducting region is bounded by the intersection of the $\[0, i\_c\]\times \[0, i\_c\]$ square and the triangle under $i\_1+i\_2=i\_c(x=X)$. Thus trying to find $i\_c(x=X)$ would mean you want to get a point inside the square but on the line since it encodes the most information about $i\_c(x=X)$ without including information about $i\_c(\alpha)$. When $i\_c(x=X) \geq 2i\_c(\alpha)$ the line no longer intersects the square and therefore a search won't be able to find the true value of $i\_c(x=X)$. The optimal search algorithm for this setup would involve search along the line $i\_1=i\_2$ which only intersects the critical line once.\[/caption\]Along side faster simulations, this also means we get to do less measurements for this protocol since finding $i\_c$ involves sending repeated measurements for each segment of the wire. With 7 iterations, we can get more than 4 significant figures worth of information on $i\_c(x=X)$ as opposed to on the order of a thousand steps for an algorithm like gradient descent. If every measurement on the actual wire took 20ms, a wire we are testing at 200 positions will take about 2 hours using gradient descent as opposed to half a minute with the modified protocol.

Here is an example simulation of what the $i\_c$ boundary is at a fixed location in a wire:

\[caption id="attachment\_24506" align="alignnone" width="3528"\]![](images/Screen-Shot-2022-08-02-at-11.46.09-PM.png) Simulated regions of superconductivity in a wire segment $X$ after sending in two pulses (of magnitude $i\_1$ and $i\_2$) that intersect at that position. The $x$ and $y$ axes correspond to $i\_1, i\_2$ respectively. The colormap represents that total current in a chunk you can pass through without switching the wire at any position (i.e. colormap is $i\_1+i\_2$ if wire no switch else $0$). From left to right, we have wires with different defects getting progressively worse critical current $i\_c(\alpha)\neq i\_c(x=X)$. The leftmost has no affect of $i\_c(\alpha)$ on the measurable space of $i\_c(x)$. The rightmost shows an example of a really bad defect suppressing our ability to measure $i\_c(x)$.\[/caption\]

### Higher accuracy using SNSPI Oracles

The accuracy of the protocol can be increased by utilizing the SNSPI readout part of the protocol. Assuming we have an oracle that can tell us the impact time $t\_{\text{impact}}$ with some error $\epsilon\_{\text{impact}}$, we can reconstruct more information regarding the current distribution.

A simple oracle with large $\epsilon$ could be a yes-no oracle telling us whether the impact occurred on the left or right of $x=f\_X(\tau)$. Using this, we break the symmetry along $i\_1=i\_2$ and therefore have our errors bounded by the defects on one side for each signal rather than the collective defect. Even with a one defect system, this is useful for deep defects. This is also effective on lines with one shallow defect if the line is tapered. (The construction of such an oracle would check if pulse receive time at the closer end to x=X is $> 2\tau + \{ \epsilon\_{\text{impact}} \}$).

We can also use a more complicated oracle such as a full SNSPI-resolution oracle, however, this is more restrictive on the types of wires it can be applied to (based on length and propagation velocity). These oracles reveal more information of the general location of intersection, helping validate intersections beyond a simple yes-no oracle. However, I don't think that it necessarily expands the measurable space of $i\_c(x)$.

## Simulations

Here are two examples on reconstructing the $i\_c(x)$ spectrums of a tapered wire with 3 defects and 20 random step lines. Here the input pulse $\delta$ is defined as a square pulse of width $\approx 1/7$ of $\dfrac{l}{v\_d}$ (so every pulse spans hundreds of chunks of the discrete tline).

\[caption id="attachment\_24510" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-03-at-11.43.36-AM-500x311.png) Here we have a tapered wire that exhibits a curved down $i\_c$ curve ideally, with 3 defects giving us. The blue curve is a parametrized wire to reflect 3 random defects in the wire with varying magnitude. The orange curve is applying the TDS protocol on the wire as described in this post to regenerate this spectrum.\[/caption\]

\[caption id="attachment\_24511" align="aligncenter" width="500"\]![](images/Screen-Shot-2022-08-03-at-11.46.22-AM-500x345.png) Here a wire is parametrized with an $i\_c$ curve of 20 random steps. The changes are drastic so we expect TDS to not be able to resolve them fully. The orange curve represents the simulation of applying TDS on that $i\_c(x)$ spectrum.\[/caption\]

In the random steps example, we can resolve the steps in a worse manner and see a sloping down characteristic near downwards steps not described in the mathematical model.

\[caption id="attachment\_24512" align="aligncenter" width="1024"\]![](images/Screen-Shot-2022-08-03-at-11.48.10-AM-1024x604.png) These 2 simulations are run on a nanowire with no defects (so flat $i\_c(x)=4$mA curve). The left pair showcases a square input pulse, while the right two have a $\sin^2$ input pulse. The top row shows the voltage pulses found, while the bottom plots show the original $I\_c(x)$ spectrum of the wire in blue and the estimated orange switching current from the given voltage pulse. We can see ripples that form in this TDS method and a constant shift down with the square input pulses.\[/caption\]

This final simulation showcases ripples on a wire with no defects. I think this is either a quirk of the model - number of ripples dependent on simulation parameters or it has physical meaning that differentiates the types of maximas. I tried having different length tlines (by keeping L, C constant and changing the number of chunks) and increasing the resolution of the simulation and the ripples were unchanged in location or magnitude.

## Equipment Realization

It doesn't seem to matter what kind of pulse you send in, gaussian vs. step vs. $sin^2$. As long as we control the peak, the switching happens at that point. Since we are also performing the opposite of a SNSPI readout (opposite of a TDC), the type of input we need can be more specific than a AWG (a DTC?), meaning that we can use a faster [Digital Delay Generator - which is something we have in lab](https://www.thinksrs.com/products/dg645.html). With 25ps jitter and accurate control over pulse magnitude and the ability to XOR two inputs to make a single pulse, it should be good for our use case. We will use the pulse send in time over the SNSPI readout mechanism since we can obtain higher accuracy that way.
