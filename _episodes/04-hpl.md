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
> > ## Solution
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

## HPL Parameters

An example `HPL.dat` parameter file is below:

```
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
24576        Ns
2            # of NBs
2048 4096    NBs
0            PMAP process mapping (0=Row-,1=Column-major)
2            # of process grids (P x Q)
1 3          Ps
6 2          Qs
16.0         threshold
1            # of panel fact
2            PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
4            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
2            RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
1            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
0            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

The syntax of the file is line-based, with editable parameters on the left,
followed by explanatory comments on the right.  HPL only uses square matrixes
and tiles, so you only set N and NB (respectively).
It's fine to leave output going to stdout and just use `xhpl | tee HPL.log`
to save a copy to file.  Obviously the processor layout has to
use all MPI ranks.

The remaining parameters deal with properties of the LU factorization
algorithm used.  PFACT and RFACT deal with the direction of progress
through the matrix (right- or left-moving).  BCASTs and
DEPTHs change the patterns used to communicate tiles
between ranks.
The transpose options change the storage (and thus access order)
of the LU decomposition products, L1 and U.
Equilibration refers to running an initial
computation to "wake up" the hardware.  Memory alignments larger
than 8 (but still a power of 2) might help.

To understand the parameters completely, you can refer to the
following reference: [Dongarra, Faverge, Ltaief and Luszczek, "Achieving numerical accuracy and high performance using
recursive tile LU factorization with partial pivoting" Concurrency Computat.: Pract. Exper. (2013)](http://www.icl.utk.edu/files/publications/2013/icl-utk-574-2013.pdf).

{% include links.md %}
