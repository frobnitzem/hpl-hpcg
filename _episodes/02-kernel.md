---
title: "Computational Kernels"
teaching: 10
exercises: 30

questions:
- "What are memory and compute-bound kernels?"

objectives:
- "Run the OSU MPI and Mixbench GPU benchmarks."
- "Understand bottlenecks to peak performance."

keypoints:
- "Most applications are memory-bound, which complicates parallelization."
- "Compute-bound applications depend on peak theoretical flops."
- "Getting good performance from parallel solvers is hard."
---

High-performance compute clusters make use of specialized network hardware
in order to maximize message bandwidth and minimize latency.
In most cases, this means specialized "network fabric" libraries
from the hardware vendors need to be installed.  Then, some
version of the MPI library should be built "on top" of these
vendor communication libraries.

This linking process can be confusing - so it's important
to read the vendor and MPI library instructions carefully.
Then benchmark your MPI by using it to exchange data
and timing it.  If the timings don't match what the vendor
says can be done, it probably means you are not compiling
MPI or your application with the right options.

It could also mean you are launching your MPI process incorrectly,
causing the wrong assignment of MPI ranks to processors and
other machine resources.

> ## Run the OSU MPI Benchmarks
>
> Download the latest version of the [OSU Microbenchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/).  Extract, compile and link with your version of MPI,
> and run the one-sided **osu_get_latency** and **osu_get_bw** benchmarks using
> 2 MPI ranks on the same host.
>
> Plot the results to show latency and bandwidth as a function
> of packet size.
>
> > ## Solution
> > ~~~
> > wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.7.1.tgz
> > tar xzf osu-micro-benchmarks-5.7.1.tgz
> > mkdir osu.default && cd osu.default
> > ../osu-micro-benchmarks-*/configure CC=mpicc CXX=mpicxx CFLAGS=-I$(PWD)/../src/osu-micro-benchmarks-*/util --prefix=$PWD
> > cd libexec/osu-micro-benchmarks/mpi/one-sided
> > mpirun -n 2 ./osu_get_latency >latency.dat
> > mpirun -n 2 ./osu_get_bw >bw.dat
> > gnuplot <<.
> > set term eps enh color lw 2
> > set out "bw.eps"
> > set logscale x
> > set logscale y
> > set xlabel "Message Size / bytes"
> > set ylabel "Bandwidth / MB/sec"
> > plot "bw.dat" u 1:2 w l
> >
> > set out "latency.eps"
> > set ylabel "Latency / us"
> > plot "latency.dat" u 1:2 w l
> > .
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

Now that we've covered how to test communication speed, it's time
to turn to local computation speed.  Graphics processing units (GPUs)
are constructed with matrix (image) processing in-mind.
They achieve this by using very large vector-sizes and optimizing
for performing single instruction multiple data (SIMD) operations
while pre-fetching data from memory.

Reading and writing data to GPU memory takes many more clock cycles
than performing operations like add, multiply, divide, on
data that's already present.
This means that you have to do lots of operations on data that's
already sitting in the SIMD units in order for your kernel
to be compute-bound.

The number of floating point operations done on every byte
read is called "Arithmetic Intensity."
It controls the trade-off between reading a new batch
of data and operating on the data that's present.

If you do just one operation for every byte read, then
the run-time will depend on the GPU's maximum memory bandwidth.
At some value of arithmetic intensity, however,
you'll be doing so many operations on every byte of data
that the memory access will be completely hidden.
In this case, the performance flattens to a "roofline",
at the peak floating point operations the processor
is capable of performing.

You can see this behavior by running an invented
kernel with an adjustable number of flops per byte,
and plotting flops vs. arithmetic intensity.

> ## Run Mixbench
>
> Download the latest version of the [Mixbench Microbenchmark](https://github.com/ekondis/mixbench)
> and produce a "roofline" plot of achieved flops vs arithmetic intensity.
>
> > ## Solution
> > ~~~
> > cd $HOME
> > git clone https://github.com/ekondis/mixbench.git
> > mkdir mixbench/build && cd mixbench/build
> > cmake -DCMAKE_CUDA_ARCHITECTURES=80 ../mixbench-cuda
> > make -j
> > ./mixbench-cuda-alt | tee alt.log
> > grep ^ *[0-9] alt.log* | sed -e s/,//g >alt.dat
> > gnuplot <<.
> > set term eps enh color lw 2
> > set out "roofline.eps"
> > set xlabel "Arithmetic Intensity / FLOP/BYTE"
> > set ylabel "Bandwidth / MB/sec"
> > plot "bw.dat" u 2:4 w l title "single"
> > plot "bw.dat" u 6:8 w l title "double"
> > .
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

## Peak performance

Both of the micro-benchmarks above show how communicating data
can easily become a bottleneck for algorithm run-time.
Even the very fast memory movement between GPU memory
and GPU processors can occupy the time of
hundreds of floating point operations.

When part of a matrix or vector needs to be communicated
between MPI ranks before a solution step can proceed,
then the whole computation runs the risk of running at
the same speed as the network bandwidth.  Even small-data operations
like broadcasting the result of a local vector-vector dot product
incur a latency cost.

To write an effective parallel algorithm, data movement
must be overlapped with computation as much as possible.
Further, scattering and gathering data between MPI ranks
should make use of faster links between ranks on the same
node as much as possible.

Finally, all these operations
should be done as explicitly as possible so that the processors
access local memory.  Asking two processors to implicitly
be aware of memory updates from one another causes
the processes to interrupt each other.
Assigning every MPI rank to a specific processor and
bank of memory (using system NUMA controls) prevents
these hidden interruptions.


> ## Check for NUMA Issues
>
> Add MPI init, send/receive, and finalize calls to mixbench.
>
> Create a batch script that launches this MPI job over
> your entire cluster.  Check the point-to-point bandwidth
> achieved between your send/receive calls
> and make sure that every rank is getting full
> use of the GPU.
>
> If you have issues at this stage, it's likely that your
> mpirun launch configuration is incorrect.
>
{: .challenge}

{% include links.md %}
