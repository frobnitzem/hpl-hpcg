---
title: "Introduction"
teaching: 10
exercises: 10
math: true

questions:
- "Why are HPL and HPCG used as benchmarks?"
- "What is a linear solve used for?"

objectives:
- "Explain dense and sparse matrices."
- "List some applications that generate these types of problems."

keypoints:
- "Linear systems in both dense and sparse form are a universal theme in scientific computing."
- "Dense and sparse matrices have different optimal algorithms."
---

High-Performance Linpack (HPL) and High-Performance Conjugate-Gradient (HPCG)
are two of the most famous benchmarks for supercomputers.
As a benchmark, they are intended to show the maximum achievable
performance of the parallel machine on an important problem with
real-world applications.

## Where does the problem come from?

Both benchmarks solve for $X$ given $A$ and $B$ in the matrix-vector problem(s),

$A X = B, A \in \mathcal M_{n\times n}, X,B \in \mathcal M_{n\times k}$

The notation $\mathcal M_{n \times k}$ stands for an n by k matrix.
The matrix $A$ is square, and is sometimes referred to as the "operator".
Its properties are very important in determining the algorithm
used to solve for $X$.  Each of the $k$ columns of the matrix
$B$ are called "right-hand side vectors".

Because of the way matrix multiplication is defined, each "right-hand side vector"
corresponds to a column of $X$.  This column of $X$ is also called
the "unknown vector" or "solution vector".
Thus, $AX=B$ represents $k$ independent matrix-vector problems.
We use lower-case
$x$,$b$ to refer to a single $\mathcal M_{n\times 1}$ vector.

Matrix-vector problems like the above come from trying to solve
linear systems of equations, like the Helmholtz equation,

$(\nabla^2 + k^2) X(r) = B(r)$

which represents the scattering of a sound-wave off of
a surface, or a quantum wavefunction at a specific energy.
Discretizing the field $X(r)$ by storing only
some points, $X_i = X(r_i)$, leads to the matrix-vector form.

In this case, the matrix size (also known as dimension), $N$,
is the number of sampled real-space points, $r_i$.
In many applications, linear equations appear at each time-step,
or as a Newton step toward solving a non-linear equation.
It's not difficult to see larger problem domains and more
dimensions lead to very large linear systems with $N^2$ being
as large as an entire supercomputer's working memory!


## HPCG and Sparse Matrices

The properties of the operator, $A$, determine the most efficient
algorithm to use when solving a linear system.
High-Performance Linpack solves dense linear systems and High-Performance
Conjugate Gradient solves sparse ones.

A sparse matrix is one in which only order N ($O(N)$) elements
are non-zero.
The Helmholtz equation above is *sparse* because multiplication by $k^2$
and taking the derivative, $\nabla^2$ are only *local* operations.
This means only a constant number of elements in $X$ are
needed to compute $(A X)\_i$.  In one-dimension,

$$
A = \begin{pmatrix}
(k\Delta x)^2-2 & 1 & 0 & 0 & 1 \\
1 & (k\Delta x)^2-2 & 1 & 0 & 0 \\
0 & 1 & (k\Delta x)^2-2 & 1 & 0 \\
0 & 0 & 1 & (k\Delta x)^2-2 & 1 \\
1 & 0 & 0 & 1 & (k\Delta x)^2-2
\end{pmatrix} \Delta x^{-2}
$$

If processor one holds the first two elements of $x$ and processor two holds the last three elements of $x$, then the two only need to communicate one element of $x$ in order for both to compute their part of the product, $A x$.

The method of conjuate gradients uses the fact that $Ax - b = 0$
in the final solution to iteratively adjust $x$ by computing
$A\tilde x - b$ ($\tilde x$ is an estimate of $x$).
Thus, the HPCG benchmark focuses on this communicate and multiply rows strategy.

Applying the $A$ operator to the right-hand side, $\tilde x$
is done by a sparse matrix multiplication.
Since each row has a constant number of non-zero elements,
that sparse multiplication takes $O(N)$ work
instead of $O(N^2)$ for a dense matrix.
In a "perfect" parallel machine with $P$ processors,
the multiplication can be done in $O(N/P)$ time.

However, the cost of using an iterative method is that
the number of iterations needed to obtain convergence
to machine precision is unknown.  The convergence
of conjugate gradients has been extensively studied,
with the result that the number of iterations is
somewhere between $O(\sqrt \kappa)$ and $O(N)$,
with some algorithms guaranteeing less then $O(\sqrt \kappa)$
iterations.  Here, $\kappa$ is the condition number
of the matrix, equal to the ratio of the largest to smallest
matrix eigenvalue.  There is theoretical evidence that
(for matrices with Gaussian random elements)
the condition number is proportional to the matrix dimension, $N$.
This means in practice that the number of iterations required for convergence
is on the order of $\sqrt N$, and that conjugate gradients on a
"perfect" parallel machine scales as $O(N^{3/2} / P)$.


## HPL and Dense Matrices

Dense matrices, on the other hand, are matrices in which $O(N^2)$ elements
are nonzero.  In this case, much more communication would be needed to multiply
$Ax$.  Rather than use conjugate gradients, dense matrix solves are more
effectively accomplished by performing row-operations on the matrix, $A$.

The computational method, however, is a bit different than doing Gaussian
elimination by hand.  Instead, we just need to apply enough row operations
to make $A$ upper-diagonal (turning $A$ into $U$).
The operations can be collected into a lower-diagonal
matrix, $L$, and a permutation matrix, $P$.  Each
permutation swaps two rows, so $P$ is a matrix containing
$N$ ones.  The complete process is summarized as,

$ A = P L U $

where all matrices are $N \times N$.
This is called a "factorization" of $A$ into an "L-U" decomposition.
The computational work required for L-U decomposition is $O(N^3)$.

Given the factorization, $A x = b$ can be solved by step-wise
solution:

$ x = (PLU)^-1 b = U^{-1} (L^{-1} (P^{-1} b)) $

Permutation is just a communication step.
Each of the next two multiplications take
just about the same amount of work as a
matrix-multiplication, $O(N^3)$.

On a "perfect" parallel machine with $P$ processors,
this should take $O(N^3/P)$ time.


{% include links.md %}
