# Timings and parallelization

This section summarizes the options DFTK offers
to monitor and influence performance of the code.

```@setup parallelization
using DFTK
using PseudoPotentialData
a = 10.26  # Silicon lattice constant in Bohr
lattice = a / 2 * [[0 1 1.];
                   [1 0 1.];
                   [1 1 0.]]
pseudopotentials = PseudoFamily("cp2k.nc.sr.lda.v0_1.semicore.gth")
Si = ElementPsp(:Si, pseudopotentials)
atoms = [Si, Si]
positions = [ones(3)/8, -ones(3)/8]

model = model_DFT(lattice, atoms, positions; functionals=LDA())
basis = PlaneWaveBasis(model; Ecut=5, kgrid=[2, 2, 2])

DFTK.reset_timer!(DFTK.timer)
scfres = self_consistent_field(basis, tol=1e-5)
```

## Timing measurements

By default DFTK uses [TimerOutputs.jl](https://github.com/KristofferC/TimerOutputs.jl)
to record timings, memory allocations and the number of calls
for selected routines inside the code. These numbers are accessible
in the object `DFTK.timer`. Since the timings are automatically accumulated
inside this datastructure, any timing measurement should first reset
this timer before running the calculation of interest.

For example to measure the timing of an SCF:
```@example parallelization
DFTK.reset_timer!(DFTK.timer)
scfres = self_consistent_field(basis, tol=1e-5)

DFTK.timer
```
The output produced when printing or displaying the `DFTK.timer`
now shows a nice table summarising total time and allocations as well
as a breakdown over individual routines.


!!! note "Timing measurements and stack traces"
    Timing measurements have the unfortunate disadvantage that they
    alter the way stack traces look making it sometimes harder to find
    errors when debugging.
    For this reason timing measurements can be disabled completely
    (i.e. not even compiled into the code) by setting the package-level preference
    `DFTK.set_timer_enabled!(false)`. You will need to restart your Julia session
    afterwards to take this into account.

## Rough timing estimates
A very (very) rough estimate of the time per SCF step (in seconds)
can be obtained with the following function. The function assumes
that FFTs are the limiting operation and that no parallelisation is employed.

```@example parallelization
function estimate_time_per_scf_step(basis::PlaneWaveBasis)
    # Super rough figure from various tests on cluster, laptops, ... on a 128^3 FFT grid.
    time_per_FFT_per_grid_point = 30 #= ms =# / 1000 / 128^3

    (time_per_FFT_per_grid_point
     * prod(basis.fft_size)
     * length(basis.kpoints)
     * div(basis.model.n_electrons, DFTK.filled_occupation(basis.model), RoundUp)
     * 8  # mean number of FFT steps per state per k-point per iteration
     )
end

"Time per SCF (s):  $(estimate_time_per_scf_step(basis))"
```

## Options for parallelization
At the moment DFTK offers two ways to parallelize a calculation,
firstly shared-memory parallelism using threading
and secondly multiprocessing using MPI
(via the [MPI.jl](https://github.com/JuliaParallel/MPI.jl) Julia interface).
MPI-based parallelism is currently only over ``k``-points,
such that it cannot be used for calculations with only a single ``k``-point.
There is also support for [Using DFTK on GPUs](@ref).

The scaling of both forms of parallelism for a number of test cases
is demonstrated in the following figure.
These values were obtained using DFTK version 0.1.17 and Julia 1.6
and the precise scalings will likely be different
depending on architecture, DFTK or Julia version.
The rough trends should, however, be similar.

```@raw html
<img src="../scaling.png" width=750 />
```

The MPI-based parallelization strategy clearly shows a superior scaling
and should be preferred if available.


## MPI-based parallelism
Currently DFTK uses MPI to distribute on ``k``-points only.
This implies that calculations with only a single ``k``-point
cannot use make use of this.
For details on setting up and configuring MPI with Julia
see the [MPI.jl documentation](https://juliaparallel.github.io/MPI.jl/stable/configuration).

1. First disable all threading inside DFTK, by adding the
   following to your script running the DFTK calculation:
   ```julia
   using DFTK
   disable_threading()
   ```

2. Run Julia in parallel using the `mpiexecjl` wrapper script from MPI.jl:
   ```sh
   mpiexecjl -np 16 julia myscript.jl
   ```
   In this `-np 16` tells MPI to use 16 processes and `-t 1` tells Julia
   to use one thread only.
   Notice that we use `mpiexecjl` to automatically select the `mpiexec`
   compatible with the MPI version used by MPI.jl.

As usual with MPI printing will be garbled. You can use
```julia
DFTK.mpi_master() || (redirect_stdout(); redirect_stderr())
```
at the top of your script to disable printing on all processes but one.

!!! info "MPI-based parallelism not supported everywhere"
    While most standard procedures are now supported in combination with MPI,
    some functionality is still missing and may error out when being called
    in an MPI-parallel run.
    In most cases there is no intrinsic limitation it just has not yet been
    implemented. If you require MPI in one of our routines, where this is not
    yet supported, feel free to open an issue on github or otherwise get in touch.

## Thread-based parallelism
Threading in DFTK currently happens on multiple layers
distributing the workload
over different ``k``-points, bands or within
an FFT or BLAS call between threads.
At its current stage our scaling for thread-based parallelism
is worse compared MPI-based and therefore the parallelism
described here should
only be used if no other option exists.
To use thread-based parallelism proceed as follows:

1. Ensure that threading is properly setup inside DFTK by adding
   to the script running the DFTK calculation:
   ```julia
   using DFTK
   setup_threading()
   ```
   This disables FFT threading and sets the number of
   BLAS threads to the number of Julia threads.

2. Run Julia passing the desired number of threads using the flag `-t`:
   ```sh
   julia -t 8 myscript.jl
   ```

For some cases (e.g. a single ``k``-point, fewish bands and a large FFT grid)
it can be advantageous to add threading inside the FFTs as well. One example
is the Caffeine calculation in the above scaling plot. In order to do so
just call `setup_threading(n_fft=2)`, which will select two FFT threads.
More than two FFT threads is rarely useful.

## Advanced threading tweaks
The default threading setup done by `setup_threading` is to select
one FFT thread, and use `Threads.nthreads()` to determine the number of DFTK
and BLAS threads. This section provides some info in case you want to change these defaults.

### BLAS threads
All BLAS calls in Julia go through a parallelized OpenBlas
or MKL (with [MKL.jl](https://github.com/JuliaComputing/MKL.jl).
Generally threading in BLAS calls is far from optimal and
the default settings can be pretty bad.
For example for CPUs with hyper threading enabled,
the default number of threads seems to equal the number of *virtual* cores.
Still, BLAS calls typically take second place
in terms of the share of runtime they make up (between 10% and 20%).
Of note many of these do not take place on matrices of the size
of the full FFT grid, but rather only in a subspace
(e.g. orthogonalization, Rayleigh-Ritz, ...)
such that parallelization is either anyway disabled by the BLAS library
or not very effective.
To **set the number of BLAS threads** use
```julia
using LinearAlgebra
BLAS.set_num_threads(N)
```
where `N` is the number of threads you desire.
To **check the number of BLAS threads** currently used, you can use `BLAS.get_num_threads()`.

### DFTK threads
On top of BLAS threading DFTK uses Julia threads in a couple of places
to parallelize over ``k``-points (density computation) or bands
(Hamiltonian application). The number of threads used for these
aspects is controlled by default by `Threads.nthreads()`, which can be
set using the flag `-t` passed to Julia or the *environment variable*
`JULIA_NUM_THREADS`. Optionally, you can use `setup_threading(; n_DFTK)`
to set an upper bound to the number of threads used by DFTK,
regardless of `Threads.nthreads()` (for instance, if you want to
thread several DFTK runs and don't want DFTK's threading to
interfere with that).

### FFT threads
Since FFT threading is only used in DFTK inside the regions already parallelized
by Julia threads, setting FFT threads to something larger than `1` is
rarely useful if a sensible number of Julia threads has been chosen.
Still, to explicitly **set the FFT threads** use
```julia
using FFTW
FFTW.set_num_threads(N)
```
where `N` is the number of threads you desire.
By default no FFT threads are used, which is almost always the best choice.
