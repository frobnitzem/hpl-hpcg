---
title: "Using GPUs"
teaching: 10
exercises: 0
math: true

questions:
- "What modifications to the benchmarks are required to utilize GPUs?"

objectives:
- "Demonstrate methods for using GPUs with HPL and HPCG."

keypoints:
- "Using GPUs requires modifications to the standard HPL and HPCG programs."
---

Using GPUs is more difficult than CPU-only codes because
separate compilers are needed, and because computations run
on the GPU are "offloaded" from the CPU.  An offloaded
computation is a function that is packed up and sent to the GPU,
executed there (usually while the CPU does something else),
and then eventually waited on by the CPU.

Importantly for this exercise, the standard HPL code
distributed by netlib does not include support for running
computations on GPUs.  Instead, hardware vendors have supplied
these themselves.  NVIDIA offers custom programs that run
HPL through its (free) developer program.  More recently,
they have also created a [container running HPL and HPCG](https://ngc.nvidia.com/catalog/containers/nvidia:hpc-benchmarks).

Rather than work with these, I suggest trying the free alternative
[HPL-GPU](https://github.com/davidrohr/hpl-gpu) or (older)
[HPL-CUDA](https://github.com/avidday/hpl-cuda).
The second was easier to compile.  In addition to following
the instructions, you should also set the `TOPdir` variable
to the top directory.

To make the most performant binary, it is important to
enable (or at least check) what hardware-optimizations exist.
For CPU-s it's well-known that you can check the processor
options from `/dev/cpuinfo`.  For GPUs, the most important
property is the compute capability.
A quick reference chart is [here](https://arnon.dk/matching-sm-architectures-arch-and-gencode-for-various-nvidia-cards/).


{% include links.md %}
