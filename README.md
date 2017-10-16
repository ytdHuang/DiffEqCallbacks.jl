# DiffEqCallbacks.jl

[![Join the chat at https://gitter.im/JuliaDiffEq/Lobby](https://badges.gitter.im/JuliaDiffEq/Lobby.svg)](https://gitter.im/JuliaDiffEq/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Build Status](https://travis-ci.org/JuliaDiffEq/DiffEqCallbacks.jl.svg?branch=master)](https://travis-ci.org/JuliaDiffEq/DiffEqCallbacks.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/a3o1a4l4xqcwuw86?svg=true)](https://ci.appveyor.com/project/ChrisRackauckas/diffeqcallbacks-jl-ufx45)
[![Coverage Status](https://coveralls.io/repos/JuliaDiffEq/DiffEqCallbacks.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/JuliaDiffEq/DiffEqCallbacks.jl?branch=master)
[![codecov.io](http://codecov.io/github/JuliaDiffEq/DiffEqCallbacks.jl/coverage.svg?branch=master)](http://codecov.io/github/JuliaDiffEq/DiffEqCallbacks.jl?branch=master)

[![DiffEqCallbacks](http://pkg.julialang.org/badges/DiffEqCallbacks_0.5.svg)](http://pkg.julialang.org/?pkg=DiffEqCallbacks)
[![DiffEqCallbacks](http://pkg.julialang.org/badges/DiffEqCallbacks_0.6.svg)](http://pkg.julialang.org/?pkg=DiffEqCallbacks)

This is a library of callbacks for extending the solvers of DifferentialEquations.jl.

## Usage


To use the callbacks provided in this library with DifferentialEquations.jl solvers,
just pass it to the solver via the `callback` keyword argument:

```julia
sol = solve(prob,alg;callback=cb)
```

For more information on using callbacks, [see the manual page](http://docs.juliadiffeq.org/latest/features/callback_functions.html).

## ManifoldProjection

This projects the solution to a manifold, conserving a property while
conserving the order.

```julia
ManifoldProjection(g;nlsolve=NLSOLVEJL_SETUP(),save=true)
```

- `g`: The residual function for the manifold: `g(u,resid)`. This is an inplace function
  which writes to the residual the difference from the manifold components.
- `nlsolve`: A nonlinear solver as defined [in the nlsolve format](linear_nonlinear.html)
- `save`: Whether to do the standard saving (applied after the callback)

## AutoAbstol

Many problem solving environments [such as MATLAB](https://www.mathworks.com/help/simulink/gui/absolute-tolerance.html)
provide a way to automatically adapt the absolute tolerance to the problem. This
helps the solvers automatically "learn" what appropriate limits are. Via the
callback interface, DiffEqCallbacks.jl implements a callback `AutoAbstol` which
has the same behavior as the MATLAB implementation, that is the absolute tolerance
starts at `init_curmax` (default `1-e6`), and at each iteration it is set
to the maximum value that the state has thus far reached times the relative tolerance.

To generate the callback, use the constructor:

```julia
AutoAbstol(save=true;init_curmax=1e-6)
```

`save` determines whether this callback has saving enabled, and `init_curmax` is
the initial `abstol`. If this callback is used in isolation, `save=true` is required
for normal saving behavior. Otherwise, `save=false` should be set to ensure
extra saves do not occur.

## Domain Controls

The domain controls are efficient methods for preserving a domain relation for
the solution value `u`. Unlike the `isoutofdomain` method, these methods use
interpolations and extrapolations to more efficiently choose stepsizes, but
require that the solution is well defined slightly outside of the domain.

### PositiveDomain

```julia
PositiveDomain(u=nothing; save=true, abstol=nothing, scalefactor=nothing)
```

### GeneralDomain

```julia
GeneralDomain(g, u=nothing; nlsolve=NLSOLVEJL_SETUP(), save=true,
                       abstol=nothing, scalefactor=nothing, autonomous=numargs(g)==2,
                       nlopts=Dict(:ftol => 10*eps()))
```

## StepsizeLimiter

The stepsize limiter lets you define a function `dtFE(t,u)` which changes the
allowed maximal stepsize throughout the computation. The constructor is:

```julia
StepsizeLimiter(dtFE;safety_factor=9//10,max_step=false,cached_dtcache=0.0)
```

`dtFE` is the maximal timestep and is calculated using the previous `t` and `u`.
`safety_factor` is the factor below the true maximum that will be stepped to
which defaults to `9//10`. `max_step=true` makes every step equal to
`safety_factor*dtFE(t,u)` when the solver is set to `adaptive=false`. `cached_dtcache`
should be set to match the type for time when not using Float64 values.

## SavingCallback

The saving callback lets you define a function `save_func(t, u, integrator)` which
returns quantities of interest that shall be saved. The constructor is:

```julia
SavingCallback(save_func, saved_values::SavedValues;
               saveat=Vector{eltype(saved_values.t)}(),
               save_everystep=isempty(saveat),
               tdir=1)
```
- `save_func(t, u, integrator)` returns the quantities which shall be saved.
- `saved_values::SavedValues` contains vectors `t::Vector{tType}`,
  `saveval::Vector{savevalType}` of the saved quantities. Here,
  `save_func(t, u, integrator)::savevalType`.
- `saveat` Mimicks `saveat` in `solve` for ODEs.
- `save_everystep` Mimicks `save_everystep` in `solve` for ODEs.
- `tdir` should be `sign(tspan[end]-tspan[1])`. It defaults to `1` and should
  be adapted if `tspan[1] > tspan[end]`.

## PeriodicCallback

`PeriodicCallback` can be used when a function should be called periodically in terms of integration time (as opposed to wall time), i.e. at `t = tspan[1]`, `t = tspan[1] + Δt`, `t = tspan[1] + 2Δt`, and so on. This callback can, for example, be used to model a digital controller for an analog system, running at a fixed rate.

A `PeriodicCallback` can be constructed as follows:

```julia
PeriodicCallback(f, Δt::Number; kwargs...)
```

where `f` is the function to be called periodically, `Δt` is the period, and `kwargs` are keyword arguments accepted by the `DiscreteCallback` constructor.
