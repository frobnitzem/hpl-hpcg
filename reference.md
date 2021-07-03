---
layout: reference
---

## Glossary

* gemm = general matrix multiply
* gesv = general matrix vector solve
* LU-decomposition = factorization of a matrix into the product of lower and upper-triangular matrices
  - This is used for dense matrices.
* pivot = operation of swapping two rows of a matrix in order to move a chosen element (the pivot element) onto the diagonal
* dense matrix = matrix where O(N^2) elements are nonzero.
* sparse matrix = matrix where most of the elements are zero.
  - Typically a compressed "sparse" storage format is used for matrices containing only O(N) nonzero elements.
* matrix block / tile = small sub-matrix consisting of a contiguous range of rows and columns
* dot product = product of two vectors, summed over all elements
* multigrid = hierarchical method of solving a matrix-vector problem based on coarsening to a smaller size and eventual refinement to the original size
* conjugate gradient = optimization method employing successive vector directions based on the gradient of an objective function
* preconditioner = approximate matrix inverse leaving a matrix-vector problem "closer" to a solution
* arithmetic intensity = floating point operations per byte of data transferred to the processor doing the operations
* GFLOPS = 1024^3 floating point operations per second

## References

* https://www.netlib.org/benchmark/hpl/
* https://github.com/avidday/hpl-cuda
* https://www.hpcg-benchmark.org
* https://github.com/hpcg-benchmark/hpcg
* http://www.netlib.org/utk/papers/outofcore/node3.html
* https://ulhpc-tutorials.readthedocs.io/en/latest/parallel/mpi/HPL/
* https://ulhpc-tutorials.readthedocs.io/en/latest/parallel/hybrid/HPCG/

* [LAPACK matrix types](https://www.netlib.org/lapack/lug/node24.html)

{% include links.md %}
