---
title: "Generating Optimal Tapers using DiffEq.jl"
date: "2022-09-19"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

From reading [the original paper on klopf tapers](https://les-mathematiques.net/phorum/file.php/4/109040/A_Transmission_Line_Taper_of_Improved_Design-Klopfenstein.pdf) it seems like there are a bunch of areas where we can improve on.

## Analytical Solution Theory

In terms of the equations, the conversion of the non-linear first order diff eq to a linear one involves an approximation we need not assume. The original paper assumes a solution by constraining the order of the chebyshev polynomials, this causes the discretization of the taper impedance transformations. This is the main diff. eq. the paper tries to solve

$\dfrac{d\rho}{dx} = \rho(x) \mu(x) + \dfrac{1}{2}\dfrac{d\ln Z\_0(x)}{dx}$.

One cool thing to note is notice that from the structure of the diff. eq in the paper that impedance transformations are invariant under multiplication. I.e. showing a taper $T=(Z\_0, Z\_1, \cdots Z\_N)$ from $1\Omega \xrightarrow{} 20\Omega$ exists, we know a taper exists from $50\Omega\xrightarrow{} 1000\Omega$ exists by multiplying the impedance of the original taper by 50 each $50T = (50Z\_0, \cdots 50Z\_N)$. This means that instead of impedance transformations being a function on $\mathbb{R}^2$, they actually are a function on $\mathbb{R}^+$ (this means that you have to prove a taper geometry works for many less values to prove it works and is efficient for all possible values). This result also implies that we can reuse a lot of taper calculations between different applications.

There are two ways of solving this numerically, either by minimizing the above diff. eq. or by minimizing a spectrum of $S\_{xy}$. The first method is cool and optimizes over continuous functions, however, I chose the latter since with the right optimizations the spectrum optimization could be much faster and take advantage of hypo-parametrization of inputs and harmonic balance symbolic recomputation.

## An Optimal Taper Design

Another possible way of solving this problem is gradient descent. This is much more involved as we need to achieve multiple different goals.

1. Efficient simulation of tapers to extract the $S\_{xy}$ parameters of the wire
2. Meaningful loss parametrization that takes into account both bandwidth and losses
3. Realistic regularization
4. Decreasing the dimensionality of the variable to be optimized

To tackle 1, I used a harmonic balance simulation in the frequency domain that takes in as an input $2N$ variables ($N$ is on the order of $1000$) that represent $L$ and $C$ for individual inductors and capacitors that make up our discrete tline. This FD simulator is a combination of the TD julia simulator I've been working on and Kevin O'Brien's package JosephsonCircuits.jl. Each simulation is run with 10 signal and pump modes, $\omega\_p=2\pi\cdot 7.12$MHz, $\omega\_s=2\pi \cdot (10^4, 10^10)$. The linear portion of the taper's characteristic is used, i.e. it is always s/c-ing.

We can then calculate the $S\_{xy}$ parameters from the simulation result. I started out with the main loss in this simulation is defined as $\dfrac{1}{N} \sum\_{x\neq y} \sum\_{f\in\omega\_s} S\_{xx}\[f\] - S\_{xy}\[f\]$. This accounts for bandwidth and transmittance of the taper. Later on I moved to a better defined linear loss function where we define two subspaces of $(0, \omega\_s)$, $\Omega\_{sub}$ and $\Omega\_{sup}$ that represent the frequencies which the filter is supposed to either pass or dampen. We can then define corresponding sigmoid functions $\sigma\_{t}$ over an input $S\_{xy}$ value that checks whether it is close to the band ripple ($t=-20$dB for this example) or $t=-1\text{dB}$. Our final loss function would look like $\dfrac{1}{|\Omega\_{sub}|} \sum\_{\omega \in |\Omega\_{sub}|} \sigma\_{-1} (S\_{11}(\omega)) + \dfrac{1}{|\Omega\_{sup}|} \sum\_{\omega \in |\Omega\_{sup}|} \sigma\_{-20} (S\_{11}(\omega))$

Realistic regularization that makes sure the taper is smooth and doesn't have non-fabricate-able features can be added. It could bound the values for inductance and capacitance (and enforce realisticness, such as $L, C > 0$). However, I chose not to employ any of these and instead define the problem in a manner where these features are promised. Ideally in a more complete version of this, regularization would be well defined and enforced and scaled properly to allow for proper fast convergence.

For this problem I decided to merge typical use-cases of regularization with the problem definition to decrease the dimensionality of the variables to be optimized (parameters). I did this using flat on-grid cubic splines parameterized by $N\_p << N$. Therefore using an $N\_p=5$ means that the vector $\[L, C\]^T$ of length $2N$ can be parametrized by a vector $p$ of length $|p|=2N\_p=10$ using some one way fitting function. This definition is powerful since we can restructure the code to use up less memory and perform more efficient gradients. It also encodes powerful promises about our lines topology. As long as $N\_p<<N$:

- The flatness condition on our spline guarantees that the edges are smoothly transitioning into $Z\_{in}, Z\_{out}$ and don't have sharp edges.
- The splines promise a differentiable and smooth transition in impedance and shunt admittance
- The slope of the interpolation between two points is bounded by at most a quadratic

These properties remove the need for a regularization term that guarantees smoothness, "realistic"-ness and boundedness. The only case that slips under the rug is the $L, C > 0$ condition, which has two ways of solving it: (1) a double cover such as absolute values or (2) regularization constriction. In an ideal implementation, you would use 1 as this would let us have a pure loss function that is "more linear" in convergence and you don't need to worry about scaling your set of constructor loss functions properly. In this proof-of-concept however, a double cover was not used because finding a double cover is easy but preserving gradients along inflections is hard. For example the absolute value function is a continuous homomorphism, however, a gradient at a point $\epsilon$ near zero with a magnitude $>\epsilon$ could cross the zero inflection point (i.e. gradient of $-2\epsilon$ at $\epsilon$ would map to $|\epsilon-2\epsilon|=|-\epsilon|=\epsilon$ causing instability in convergence).

I chose to include a second loss function I call the suggestive loss function which has a higher magnitude in the beginning and "suggests" a path for the optimizer to take. The idea behind this suggestive loss function is penalizing beginning and end impedances that are mismatched from $Z\_{in}$ and $Z\_{out}$. In the beginning, you could be stuck anywhere on an optimization curve and without a suggestive loss function, you might get a locally optimal taper since smooth deformations of a taper (in a bezier parameter space of a taper) map to non-smooth alterations in the $S$ parameters. There is a worry that the locally best solution is not matched at the ends, however, if matching does happen at the ends, then smooth bc-local (input and output impedance are preserved) deformations map to smooth changes on the $S$ parameter curves. $Z\_{offset} = \left |Z\_{in} - \sqrt{\dfrac{L\_1}{C\_1}}\right |+\left |Z\_{out} - \sqrt{\dfrac{L\_N}{C\_N}}\right |$. One thing I noticed is we can layer sigmoids in the main loss function to help suggest optimization paths since the derivative of the sigmoid is low, i.e. apply two sigmoids at the band ripple $-20$dB but one of them that transitions 10 times slower allowing for a smoother gradient transition.

I used Optim.jl to run an implementation of the NelderMead algorithm on an initial solution of a klopfenstein taper for similar parameters. The nelder mean algorithm converges slower than gradient descent, however, it allows for the complex non-linearity exhibited by the mapping of taper params to $S\_{xy}$ sigmoid transition sums. Theoretically, there is nothing stopping us from using a more efficient mapping that allows for much faster convergence when taking into account less-localized gradients. Note that the klopf taper will almost always do worse than the optimal solution due to the analytical section above. All together, running a single instance of our loss function that parameterizes a taper, interpolates the splines, simulates it, calculates the losses and gradients takes less than a tenth of a second on x86 emulated julia v1.8.1 on the M1 mac for 1000 chunks. After optimizing for 1000 steps (a feature of using a small input parameter space is needing less optimization steps for convergence) I got some interesting results. This is mostly a proof of concept rather than a use case but it seems like the results I have gotten show oscillations in $Z, \epsilon$ as opposed to a gradual increase as in the klopf and erikson tapers.

## Preliminary Result

Here is a preliminary result after 9,000 steps of optimization on an $N=10$ parameter space.![](images/optim_taper_preliminary_results-1024x662.png)

The optimal taper seems to be a higher order filter than a klopf taper for not much optimization. It also has end points that match the input and output impedance. After the design frequency, it is more consistent with the amount of damping applied to high frequency signals.

## Possible Improvements

Smooth alterations in the taper don't map to smooth alterations in the $S$ parameters, taking the loss function as a sum of $S$ parameters remove degrees of freedom that promises a new linearity and would cause more local-minimas to occur. Since loss functions are by definition scalar, we need to provide a fake sense of orthogonality of spectrum. I haven't thought about this too much, but maybe multiplying our output spectrum with a stochastic variable before summing will add orthogonality to terms of different frequency. This might be very wrong though.

Compare it to Erikson taper and variational method of solving for tapers.

Fix the loss function and optimize for longer, possibly swap loss functions throughout computation.
