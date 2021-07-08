---
title: "HPCG"
teaching: 10
exercises: 20
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

The configuration file sets problem size as number of points
along x, y, and z-dimensions.  The memory requirement for HPCG
should scale proportionately to their product, `nx*ny*nz`.
The second line contains the approximate target run-time in seconds.

For your final reported HPCG time, your output file will need
to report that the memory used was at least 25% of the system's total
memory and the run-time was at least 1800 seconds.

{% include links.md %}
