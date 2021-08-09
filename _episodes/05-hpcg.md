---
title: "HPCG"
teaching: 8
exercises: 10
math: true

questions:
- "How do sparse matrix algorithms divide work using MPI?"
- "What are the key factors affecting their performance?"
- "Why is peak network bandwidth a reasonable target for HPCG?"

objectives:
- "Setup and run the HPCG benchmark."
- "Demonstrate the effect of task placement during job launch."

keypoints:
- "HPCG requires tuning job launch configurations (like NUMA) to achieve peak communication bandwidth."
- "Few-node performance should be an indicator of full-scale performance."
---

Now that you've completed the run of HPL, running HPCG
should be a walk in the park.
~~~
spack install -v hpcg
~~~
{: .language-bash}

You might want to experiment with different compilers.  Some
examples are provided by [amd](https://developer.amd.com/spack/hpcg-benchmark/).

If you change into the bin directory spack installed HPCG into,
you can run `xhpcg` using the default parameter file.
It outputs two summary files.  The one starting HPCG-Benchmark
contains the most run details.

Key summary information is contained in the sections
marked `Performance Summary`, which include run-time,
bandwidth (GB/s), and compute throughput (GFLOP/s).


## What Happened?

The HPCG input file doesn't say a lot about the underlying
algorithm, but what's happening underneath are a series
of computational tasks that are memory-bound.

Memory-bound tasks are things like "scale a vector by 4"
or "add two vectors."  These require relatively
little computation, but much more data-movement.

When solving a sparse linear system, matrix-vector
multiplications can be computed with the help
of just a few elements from neighboring processors.
The conjugate gradient algorithm is just a series
of matrix multiplies and dot products that bring
the answer closer to the solution.  Thus,
the timings for the overall solve are
reflecting this memory bottleneck happening at each step.


## HPCG Input File

The HPCG input file, `hpcg.dat` is comparatively simple.
The two lines provide problem size (3D) and run-time (seconds).
The run-time is approximate, and running longer usually
provides higher performance because slow steps during
startup contribute less to the average.

The configuration file sets problem size as number of points
along x, y, and z-dimensions.  The memory requirement for HPCG
should scale proportionately to their product, `nx*ny*nz`.
The second line contains the approximate target run-time in seconds.

For your final reported HPCG time, your output file will need
to report that the memory used was at least 25% of the system's total
memory and the run-time was at least 1800 seconds.

## NUMA

[Non-uniform memory access](https://software.intel.com/content/www/us/en/develop/articles/optimizing-applications-for-numa.html)
is an architecture for cpu/memory
connection used in modern compute platforms.  It uses
"fast" connections between CPUs and associated memory regions.
In order to achieve peak performance, MPI ranks and processing threads
should be assigned to processors consistently (this assignment
is called process and thread affinity).
This is necessary because the Linux kernel, by default,
moves processes between CPUs to achieve load balance.

When using a batch job scheduler like SLURM,
the scheduler is able to call the `hwloc` library
and use flags to `srun` in order to setup processor
binding for every MPI rank.

On a simple cluster without SLURM, you can launch jobs
directly using mpirun, which can use ssh and a host-file.
Some versions of mpirun, like openmpi, help when deailing with thread affinity.
If you decide to tinker with calling numactl directly,
you can replace `mpirun xhpl` with `mpirun xhpl.sh`,
and write a script, `xhpl.sh` to setup environment
variables and numa options.  Some useful guides are here:
http://www.hpc.acad.bg/numactl/, https://glennklockwood.com/hpc-howtos/process-affinity.html.


{% include links.md %}
