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
- "Linking with vendor-optimized libraries is a pain in the neck."
---

The compilers, GPU, and network are working well.  Now it's time
to run a real high-intensity problem.

The following code [shamelessly adapted from stack-overflow](https://stackoverflow.com/questions/36063993/lapack-dgetrs-vs-dgesv)
uses the lapack library to solve a small
matrix-vector problem.  It's the single-node, CPU version
of the problem HPL solves.

~~~
// https://stackoverflow.com/questions/36063993/lapack-dgetrs-vs-dgesv
#include <stdio.h>
#include <vector>
#include <stdlib.h>

using namespace std;

extern "C" {
  void daxpy_(int* n,double* alpha,double* dx,int* incx,double* dy,int* incy);
  double dnrm2_(int* n,double* x, int* incx);

  void dgetrf_(int* M, int *N, double* A, int* lda, int* IPIV, int* INFO);
  void dgetrs_(char* C, int* N, int* NRHS, double* A, int* LDA, int* IPIV, double* B, int* LDB, int* INFO);
  void dgesv_(int *n, int *nrhs, double *a, int *lda, int *ipiv, double *b, int *ldb, int *info);
}

void print(const char* head, int m, int n, double* a){
  printf("\n%s\n---------\n", head);
  for(int i=0; i<m; ++i){
    for(int j=0; j<n; ++j){
      printf("%f ", a[i+j*m]);
    }
    printf("\n");
  }
  printf("---------\n");
}

int main(int argc, char *argv[]) {
  int dim = 5;
  if(argc > 1)
    dim = atoi(argv[1]);

  vector<double> a(dim * dim);
  vector<double> b(dim);
  srand(1);              // seed the random # generator with a known value
  double maxr = (double)RAND_MAX;
  for (int r = 0; r < dim; r++) {  // set a to a random matrix, i to the identity
    for (int i = 0; i < dim; i++) {
      a[r + i*dim] = rand() / maxr;
    }
    b[r] = rand() / maxr;
  }

  int info;
  int one = 1;
  char N = 'N';
  vector<int> ipiv(dim);

  vector<double> a1(a);
  vector<double> b1(b);
  dgesv_(&dim, &one, a1.data(), &dim, ipiv.data(), b1.data(), &dim, &info);
  print("B1",dim < 5 ? dim : 5,1,b1.data());

  vector<double> a2(a);
  vector<double> b2(b);
  dgetrf_(&dim, &dim, a2.data(), &dim, ipiv.data(), &info);
  dgetrs_(&N, &dim, &one, a2.data(), &dim, ipiv.data(), b2.data(), &dim, &info);
  print("B2",dim < 5 ? dim : 5,1,b2.data());

  double d_m_one = -1e0;
  daxpy_(&dim,&d_m_one,b2.data(),&one,b1.data(),&one);
  printf("\ndiff = %e\n", dnrm2_(&dim,b1.data(),&one));

  return 0;
}
~~~
{: .language-cpp}

To actually compile and use this, however, you'll need
to install a BLAS and LAPACK library.
This is where the actual fun begins.  There are many,
many different implementations of BLAS available.

* On Mac OSX, you can use the system-wide installation
  by adding the compile flag `-framework Accelerate`.

* On Linux, you'll find libopenblas, libblas, libatlas,
  and libgslcblas.  All three provide different implementations
  of the same BLAS functions.  Both openblas and atlas
  include lapack.  But, if you install libblas or libgslcblas,
  you don't get the lapack functions.  Instead, liblapack
  is a separate Linux package compatible only with libblas.

* Intel offers BLAS/LAPACK through their Math Kernel Library (MKL).
  There are so many different variations provided by MKL
  that Intel provides a [link line advisor](http://software.intel.com/en-us/articles/intel-mkl-link-line-advisor) to find the exact library names
  to link with depending on your variation.  Note, the most
  common variation is for 64-bit processors with 32-bit
  integers and openmp.

* AMD has written its own BLAS optimized for AMD platforms
  called [blis](https://developer.amd.com/amd-aocl/blas-library/).
  Blis has a lapack implementation called libFLAME.

* Co-processor accelerators (GPUs and Intel MIC cards) use
  separate blas libraries.  For GPUs, these are cublas and rocblas.
  For Intel MIC cards, the Intel BLAS will work but special
  run-time setup is needed.

* Cray systems provide BLAS and LAPACK through libsci.
  Their libsci-acc includes support for co-processor
  accelerators.

* Both Intel MKL and Cray Libsci-Acc provide scalapack,
  which is an MPI-distributed version of lapack.

* Rather than using blas and lapack, I recommend
  that new C++ codes target [blaspp](https://icl.bitbucket.io/blaspp/)
  and [lapackpp](https://bitbucket.org/icl/lapackpp).
  These packages wrap the old Fortran blas and lapack
  interfaces into a nice templated C++ design.
  They are only wrappers, however, so
  installing them requires pointing them to an
  already-installed version of blas and lapack.

Information about the Linux packages can be obtained
on Debian-based systems by querying the package manager
with commands like `aptitude search blas`, and
`aptitude show liblapack3`.
On Redhat-based systems, use the corresponding
`yum search` and `yum info`.


## Compile Your Own Blas/Lapack

For getting the most performance, it is recommended
not to use a Linux package, but to use a vendor
package or to compile your own BLAS.

Compiling your own can be extremely frustrating.
Because of this, I recommend installing blas
using [spack](spack.readthedocs.io).
In fact, you can compile MPI, HPL, and HPCG using
spack too.  The search and info commands
for spack are `spack list hpl`
and `spack info hpl`.


> ## Install HPL on OpenBlas and OpenMPI
> 
> Use spack to install the openblas and openmpi libraries.
> Then, instruct spack to use those two as dependencies
> when installing HPL.
>
> > ~~~
> > cd $HOME
> > git clone --depth=1 https://github.com/spack/spack.git
> > source spack/share/spack/setup-env.sh
> > spack install openblas threads=openmp
> > spack install openmpi +cuda fabrics=auto
> > spack spec -lINt hpl ^openblas threads=openmp ^openmpi +cuda fabrics=auto
> > spack install -v hpl ^openblas threads=openmp ^openmpi +cuda fabrics=auto
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

Pay special attention to the configure and compile lines spit out
using `spack install -v`.  These can be helpful guides if you later
decide to run the compile and install process yourself
rather than let spack handle it.

> ## Run The Solve Test Program
>
> Now that everything is installed, you should be able to run HPL.
> But first, let's try and compile the `lapack-dgetrs-vs-dgesv`
> program starting out this lesson.
>
> The key idea is to provide the directory containing lapack
> to the compiler during linking.
>
> > ~~~
> > BLAS=$(spack find --format '{prefix}' openblas)
> > g++ -o solve solve.cc -L$BLAS/lib -lopenblas
> > time ./solve 2048
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

You should explore the effect of varying `OMP_NUM_THREADS` and
launching the above using MPI.  It's also important
to note that this program is not using the GPU.
Thus, a much faster solve could have been achieved
if cublas were being called instead of openblas.

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
