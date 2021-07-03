---
title: "Introduction"
teaching: 10
exercises: 10
questions:
- "Why are HPL and HPCG used as benchmarks?"
- "What are memory and compute-bound kernels?"
objectives:
- "Explain dense and sparse matrices."
- "List some applications that generate these types of problems."
- "Understand bottlenecks to peak performance."
- "Explain matrix tiling and scalapack layout."
keypoints:
- "Most applications are memory-bound, which complicates parallelization."
- "Linear systems in both dense and sparse form are a universal theme in scientific computing."
- "Getting good performance from parallel solvers is hard."
---

High-Performance Linpack (HPL) and High-Performance Conjugate-Gradient (HPCG)
are two of the most famous benchmarks for supercomputers.
As a benchmark, they are intended to show the maximum achievable
performance of the parallel machine on an important problem with
real-world applications.


{% include links.md %}

