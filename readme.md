[Stan](https://mc-stan.org/)
is a/the state-of-the-art ecosystem for scalable high-performance Bayesian inference for
differentiable posteriors.
Dynamic [Hamiltonian Monte Carlo (HMC)](https://en.wikipedia.org/wiki/Hamiltonian_Monte_Carlo)
with its associated ([improved](https://arxiv.org/abs/1701.02434)) [No U-Turn Sampler (NUTS)](https://arxiv.org/abs/1111.4246)
can scale ridiculously well to high-dimensional problems (thousands or more unknown parameters)
and has [several built-in diagnostics](https://mc-stan.org/docs/2_27/cmdstan-guide/diagnose.html)
to monitor the quality of the obtained results.

However, efficient *and* unbiased exploration of the posterior requires selecting
an appropriate

* time step size for the
[numerical integrator](https://mc-stan.org/docs/2_27/reference-manual/hamiltonian-monte-carlo.html#leapfrog-integrator),
* ["metric"](https://mc-stan.org/docs/2_27/reference-manual/hamiltonian-monte-carlo.html#generating-transitions)
used in sampling initial momenta at each Markov Chain (MC) transition,
* [parametrization](https://mc-stan.org/docs/2_27/stan-users-guide/reparameterization-section.html) and
* degree of [approximation](https://arxiv.org/abs/2004.11408) (not always but "often" applicable).

Furthermore it is well-known that,
while HMC is [asymptotically consistent](https://betanalpha.github.io/assets/case_studies/markov_chain_monte_carlo.html#1_hello_monte_carlo_our_old_friend),
in practice it may get trapped in "insignificant" but [metastable](https://en.wikipedia.org/wiki/Metastability) modes of the posterior
(see a [simple ODE model](https://mc-stan.org/users/documentation/case-studies/planetary_motion/planetary_motion.html) for an example).

Once a good model, an appropriate parametrization and potentially an appropriate degree of approximation has been found
current best/default/well-tested practice is to first let the sampler
[warm-up](https://mc-stan.org/docs/2_27/reference-manual/hmc-algorithm-parameters.html#automatic-parameter-tuning)
in three distinct phases to

* find the
infamous "[typical](https://discourse.mc-stan.org/t/the-typical-set-and-its-relevance-to-bayesian-computation/17174) [set](https://mc-stan.org/users/documentation/case-studies/curse-dims.html)"
(phase I)
* adapt the metric (phase II) and
* adapt the time step size (phase I+II+III)

and to then "start sampling for real", discarding all previous draws.
Usually, most of the time and energy is spent adapting the metric,
which reflects in the relative "sizes" (numbers of MC transitions) allocated
to each of the three phases, which with default settings are

* 75 (phase I),
* 25+50+100+200+500=875 (phase II) and
* 50 (phase III)

where phase II is itself split into a series of windows which sequentially double in size.

[Pathfinder (August 2021)](https://arxiv.org/abs/2108.03782) promises to alleviate some of the potential problems,
providing approximate draws from the posterior and a reasonable initialization for the covariance matrix
at a fraction of the cost of Stan's default first warm-up phase
while being trivially parallelizable and being able to recover from insignificant modes.

We propose a new warm-up procedure, which

* is generally computationally more efficient than Stan's current default,
* automatically avoids insignificant modes,
* allows automatic reparameterization,
* allows automatic refinement and coarsening of approximations,

and facilitates iterative model development by

* making it easy to spot and debug coding errors or issues,
* making it easy to spot prior misspecifications,
* making it easy to spot model misspecifications which may e.g. lead to [non-identifiabilities](https://en.wikipedia.org/wiki/Identifiability),
* making it easy to spot and debug obstructions to efficient posterior exploration,
* firmly incorporating [prior, intermediate and posterior predictive checks](https://cran.r-project.org/web/packages/bayesplot/vignettes/graphical-ppcs.html) and
* allowing user intervention at every step of the process.

Reductions in warm-up wall time *of a prototype* range from significant (-20%) to astonishing (-97%) as do
reductions in required warm-up computational work (-50% to -99%) while generally not
negatively impacting sampling cost and sometimes extremely
reducing sampling cost (+10% to -99%) as measured in the number
of leapfrog steps (work) needed per [minimal estimated effective sample size](https://mc-stan.org/docs/2_27/cmdstan-guide/stansummary.html)
(precision) for a [range of posteriors](https://github.com/stan-dev/posteriordb).

Using this warm-up procedure for rapid prototyping allowed me
without any subject matter knowledge

* to successfully and rapidly fit
the so called [Monster model](https://github.com/nsiccha/monster),
a hierarchical pharmacological/toxicological ODE model which so far had frustrated
attempts to be analyzed by core developers of Stan and
[Torsten](https://metrumresearchgroup.github.io/Torsten/) (an extension
of Stan aimed at facilitating the "analysis of pharmacometric data", i.e.
the analysis of hierarchical pharmacological ODE models) and
* troubleshoot, improve and much more rapidly fit among other things epidemiological models,
hierarchical biological models and [approximate Gaussian Process models](https://github.com/nsiccha/birthday).
