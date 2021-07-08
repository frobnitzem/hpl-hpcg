---
title: "BLAS"
teaching: 10
exercises: 20
math: true

questions:
- "What blas should I link with?"
- "How can I get all these libraries installed and working together?"
- "What are the limitations of blas and lapack?"

objectives:
- "Understand the differences between blas implementations."
- "Correctly compile, link, and test linear algebra libraries."

keypoints:
- "Linking with vendor-optimized libraries is a pain in the neck."
- "Standard BLAS/LAPACK doesn't use co-processors."
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

* Co-processor accelerators (GPUs and Xeon Phi cards) use
  separate blas libraries.  For GPUs, these are cublas and rocblas.
  For Xeon Phi cards, the Intel MKL will work but special
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

## Blas Options

Not only do different implementation exist, but there are different
compilation options within the same BLAS package
that can change its performance.  Here's a table explaining each
variation:

|  Option | Help | 
|-------- | ---- |
| ilp64   | This changes the integer sizes on things like matrix dimensions<br />and tile sizes to use 64-bit integers.<br />Many systems have 32-bit integers by default,<br />which limits the maximum size a matrix can grow to |
| threads | pthreads and OpenMP are common ways to use multiple processor<br />cores for a single matrix operation. <br />With no threading, blas doesn't use threads.  Thus, only one<br />processor core will be used for each blas call.<br />That can be good if your code uses threads to make<br />multiple blas calls simultaneously.  If your code uses only<br />one thread, though, you will want to have blas use threads. | 
| static/shared | Shared object linking reduces the size of your code, but costs a<br />tiny bit more for every call to BLAS, and introduces the potential for<br />missing libraries if you ever reinstall anything.<br />If your code makes millions of blas calls, you may want to try static linking. |


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

{% include links.md %}
