---
title: "HPL"
teaching: 10
exercises: 20
math: true

questions:
- "How do dense matrix algorithms divide work using MPI?"
- "What are the key factors affecting their performance?"
- "Why is peak performance a reasonable target for HPL?"

objectives:
- "Setup and run the HPL benchmark."
- "Explain matrix tiling and scalapack layout."

keypoints:
- "HPL requires tuning matrix and tile sizes to achieve peak performance."
- "Weak scaling requires very large problem sizes for large supercomputers."
---

We've covered blas and lapack, but have carefully avoided talking
about blacs and scalapack.  What's the difference?
Well, there's a big difference.  Scalapack is synonymous
with linear algebra over dense matrices, distributed over MPI.

To divide up the work, scalapack divides up the matrix
among processors.
It does that using a very specific "scalapack layout",
that you should understand if you want to tune HPL.
First, scalapack thinks of a matrix as a collection of
blocks of size MB by NB.  To picture this, imagine
the map of a city laid out on a regular grid - like Albuquerque.
The matrix elements are the buildings, and blocks are, well,
entire city blocks.  So every matrix element is assigned to
one block, and we stop worrying about the elements and only
talk about the blocks.

Next, the blocks get assigned to processors.
This is done in round-robin fashion, processor 1 gets
block 1, 2 gets block 2, and so on until processor
P gets block P.  Then we start at the beginning
and block P+1 goes to processor 1 again.  Think of this
as a circular mapping.
Matrices have 2 dimensions, so the processor layout
is actually P by Q.  The assignment happens independently
in each dimension, so block (m,n) gets assigned
to (m mod P, n mod Q).

This block division makes sense because now
data exchange and element-wise operations are
done on entire blocks at a time - so processors
can leverage vectorization and economies of scale.

> ## Run HPL
>
> Running HPL isn't all that different than the above.
> Consult [advancedclustering](https://www.advancedclustering.com/act_kb/tune-hpl-dat-file) and the [HPL calculator](https://hpl-calculator.sourceforge.net/hpl-calculations.php) to create an input file for HPL and modify the processor
> layout (p,q), block size (nb), and total number of blocks (n/nb and m/nb).
> 
> Next, create a node-file listing the nodes available to run,
> and launch HPL with mpirun.
>
> > ~~~
> > export OMP_NUM_THREADS=4
> > mpirun -nodefile nodes -np 16 hpl hpl.dat
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

At this point, you've gone through the basics of installing
mpi and blas, and testing both for local speed, and invoking
HPL on-top of those libraries.

However, there is still a lot of work left.
We haven't covered using the GPU with HPL.
To do that, you'll want to look for NVIDIA's
optimized HPL code.

In general, you will want to arrive at a tuned
benchmark run by following these steps:

1. Find the best MPI and CPU BLAS configuration by
   compiling many different ways and testing with micro-benchmarks.
2. Find the best block-size parameter (NB) for HPL
   by running with P=Q=1 and N=8192 (single MPI rank).
3. Find the best matrix size (N) using P=Q=1 (single MPI rank),
   and keeping N a multiple of NB.  The total matrix memory
   should fill about 80-95% of the memory available to each
   MPI rank.
4. Use all available processors, with one MPI rank per processor.
   Try a range of P and Q values where P * Q = (total MPI ranks).
5. Investigate hardware-specific methods for overclocking
   while monitoring power consumption.
6. Iterate by trying small changes and looking for performance gains.

{% include links.md %}
