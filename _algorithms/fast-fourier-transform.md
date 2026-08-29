---
title: Fast Fourier Transform (Cooley-Tukey Radix-2)
authors: [Toluwanimi Odemuyiwa, John Owens]
summary: A step-by-step tensor algebraic derivation and EDGE cascade showing how tensor rank-splitting and Einsum contractions transform the classical $O(N^2)$ Discrete Fourier Transform into the $O(N \log N)$ Cooley-Tukey Fast Fourier Transform.
tags: [fft, fourier-transform, einsum, tensor-algebra, divide-and-conquer, cooley-tukey]
status_intro: |
  The **Discrete Fourier Transform (DFT)** is one of the foundational operations in scientific computing and signal processing. In its classical definition, computing the frequency spectrum of an $N$-point sequence via direct matrix-vector multiplication requires $O(N^2)$ complex operations.

  The **Fast Fourier Transform (FFT)** reduces this complexity to $O(N \log N)$. While traditionally taught using scalar index divide-and-conquer formulas or signal-flow butterfly graphs, the FFT has a natural and transparent representation as **multilinear tensor algebra (Einsum)**.

  This page presents the tensor algebraic derivation of the Cooley-Tukey Radix-2 FFT, showing how reshaping 1D arrays into higher-dimensional hypercube tensors factors a dense $O(N^2)$ matrix into $L = \log_2 N$ sparse, embarrassingly parallel Einsum butterfly contractions.
problem_statement: |
  Given an input sequence $x \in \mathbb{C}^N$, compute the discrete frequency spectrum $X \in \mathbb{C}^N$ defined by:

  $$X_k = \sum_{n=0}^{N-1} x_n \, \omega_N^{k \cdot n}, \quad k \in \{0, 1, \ldots, N-1\}$$

  where $\omega_N = e^{-j \frac{2\pi}{N}}$ (with $j = \sqrt{-1}$) denotes the primitive $N$-th root of unity.

  In standard matrix-vector Einsum notation, this direct contraction is:

  $$X_k = (F_N)_{k, n} \cdot x_n :: \bigwedge \times(\cap) \bigvee +(\cup)$$

  where $(F_N)_{k, n} = \omega_N^{k \cdot n}$ is the dense $N \times N$ DFT matrix. Evaluating this directly requires $N^2$ complex multiply-accumulate operations ($O(N^2)$).
working_example_part1: |
  ### Two-Factor Tensor Decomposition ($N = N_1 N_2$)

  Suppose $N$ factors as $N = N_1 \cdot N_2$. We decompose the 1D time and frequency coordinates into 2D indices:

  - **Time Index (Decimation in Time):** $n = n_1 N_2 + n_0$, where $n_1 \in \{0, \ldots, N_1 - 1\}$ and $n_0 \in \{0, \ldots, N_2 - 1\}$. This reshapes $x \in \mathbb{C}^N$ into a 2D tensor $x_{n_1, n_0} \in \mathbb{C}^{N_1 \times N_2}$.
  - **Frequency Index:** $k = k_0 N_1 + k_1$, where $k_1 \in \{0, \ldots, N_1 - 1\}$ and $k_0 \in \{0, \ldots, N_2 - 1\}$. This reshapes $X \in \mathbb{C}^N$ into a 2D tensor $X_{k_0, k_1} \in \mathbb{C}^{N_2 \times N_1}$.

  #### Exponent Factorization and the Twiddle Matrix
  Substituting these indices into the exponent of the root of unity yields:

  $$k \cdot n = (k_0 N_1 + k_1)(n_1 N_2 + n_0) = k_0 n_1 N + k_1 n_1 N_2 + k_1 n_0 + k_0 n_0 N_1$$

  Because $\omega_N^N = e^{-j 2\pi} = 1$, the cross-term $\omega_N^{k_0 n_1 N} = 1$ vanishes:

  $$\omega_N^{k \cdot n} = 1 \cdot \omega_{N_1}^{k_1 n_1} \cdot \omega_N^{k_1 n_0} \cdot \omega_{N_2}^{k_0 n_0}$$

  This produces a **3-step Einsum cascade**:

  1. **Column-wise Sub-DFTs along $n_1$ ($N_2$ independent length-$N_1$ transforms):**
     $$A_{k_1, n_0} = (F_{N_1})_{k_1, n_1} \cdot x_{n_1, n_0} :: \bigwedge \times(\cap) \bigvee +(\cup)$$
  2. **Twiddle Factor Modulation (Elementwise Hadamard product):**
     $$B_{k_1, n_0} = A_{k_1, n_0} \cdot T_{k_1, n_0} :: \bigwedge \times(\cap) \quad \text{where } T_{k_1, n_0} = \omega_N^{k_1 n_0}$$
  3. **Row-wise Sub-DFTs along $n_0$ ($N_1$ independent length-$N_2$ transforms):**
     $$X_{k_0, k_1} = (F_{N_2})_{k_0, n_0} \cdot B_{k_1, n_0} :: \bigwedge \times(\cap) \bigvee +(\cup)$$

  #### Cost Reduction
  The total operational cost drops from $N^2$ to:
  $$\mathcal{C}_{2\text{-Factor}} = N_2 N_1^2 + N_1 N_2 + N_1 N_2^2 = N(N_1 + N_2 + 1) \ll N^2$$

  For Radix-2 ($N_2 = 2, N_1 = N/2$), the cost is $\frac{N^2}{2} + 3N$, cutting the quadratic complexity in half in a single step.
working_example_part2: |
  ### Complete 8-Point Butterfly Walkthrough ($N=8 = 2^3, L=3$)

  Let $x = [1, 2, 3, 4, 5, 6, 7, 8]^T$. Representing time indices in 3-bit binary $n = 4b_2 + 2b_1 + b_0$, we reshape $x$ into a $2 \times 2 \times 2$ hypercube tensor.

  #### Step 0: Bit-Reversal Permutation
  Reversing the rank order maps $(b_2, b_1, b_0) \to (b_0, b_1, b_2)$:
  $$x^{(0)} = [x[0], x[4], x[2], x[6], x[1], x[5], x[3], x[7]]^T = [1, 5, 3, 7, 2, 6, 4, 8]^T$$

  #### Stage 1: Butterfly along rank $b_2$ ($W_2 = [1]$)
  Contracting the fundamental 2-point butterfly kernel $F_2 = \begin{bmatrix} 1 & 1 \\ 1 & -1 \end{bmatrix}$ along rank $b_2$:
  $$X^{(1)} = [6, -4, 10, -4, 8, -4, 12, -4]^T$$

  #### Stage 2: Twiddle and Butterfly along rank $b_1$ ($W_4 = [1, -j]$)
  Modulating by $T^{(2)} = [1, -j]$ on the $b_1=1$ elements and contracting along $b_1$:
  $$X^{(2)} = [16, -4+4j, 6-10j, -4-4j, -4, -4+4j, 6+10j, -4-4j]^T$$

  #### Stage 3: Twiddle and Butterfly along rank $b_0$ ($W_8 = [1, \frac{\sqrt{2}}{2}(1-j), -j, \frac{\sqrt{2}}{2}(-1-j)]$)
  Modulating by twiddles $W_8^k$ on the $b_0=1$ half and contracting along $b_0$ yields the exact spectrum:
  $$\begin{aligned}
    X_0 &= 36.0000 + 0.0000 j \\
    X_1 &= -4.0000 + 9.6569 j \\
    X_2 &= -4.0000 + 4.0000 j \\
    X_3 &= -4.0000 + 1.6569 j \\
    X_4 &= -4.0000 + 0.0000 j \\
    X_5 &= -4.0000 - 1.6569 j \\
    X_6 &= -4.0000 - 4.0000 j \\
    X_7 &= -4.0000 - 9.6569 j
  \end{aligned}$$
edge_expression_walkthrough: |
  - **Hypercube Embedding:** The 1D signal of length $N = 2^L$ is represented as an $L$-dimensional tensor $x \in \mathbb{C}^{2 \times 2 \times \cdots \times 2}$ over binary ranks $(b_{L-1}, \ldots, b_0)$.
  - **Bit-Reversal Permutation:** In Decimation-in-Time (Formulation A), reversing the rank indices $(b_{L-1}, \ldots, b_0) \to (b_0, \ldots, b_{L-1})$ performs the bit-reversal permutation.
  - **Stage Contraction:** At each stage $s \in \{1, \ldots, L\}$, a 2-point butterfly contraction transforms the active binary time rank into a frequency rank, coupled with stage twiddle factors $T_s$.
  - **Transpose-Free Alternative:** In Decimation-in-Frequency (Formulation B), the cascade contracts descending ranks ($b_{L-1} \to b_0$) directly on the natural-order input without requiring initial permutation.
edge_expression: |
  ### Formulation A: Decimation-in-Time (DIT) with Initial Rank Transpose
  
  $$
  \begin{aligned}
    &\triangleright\ \text{Tensors} \\
    x^{N} &\to \text{Complex},\ \text{input time-domain sequence} \\
    F_2^{2,\, 2} &\to \text{Complex},\ \text{the fundamental } 2\times 2 \text{ butterfly matrix} \\
    X^{S,\, 2,\, \ldots,\, 2} &\to \text{Complex},\ L\text{-dimensional state tensor across stages } s \in \{0, \ldots, L\} \\
    T^{S,\, 2,\, \ldots,\, 2} &\to \text{Complex},\ \text{stage-dependent twiddle factor tensor} \\[6pt]
    &\triangleright\ \text{Initialization (Stage } s = 0\text{)} \\
    X_{0,\, b_0,\, b_1,\, \ldots,\, b_{L-1}} &= x_{b_{L-1},\, \ldots,\, b_1,\, b_0} \quad (\text{Bit-reversal rank transpose}) \\[6pt]
    &\triangleright\ \text{Extended Einsum (Butterfly stage } s \in \{1, \ldots, L\}\text{)} \\
    X_{s,\, k_{s-1},\, \ldots} &= (F_2)_{k_{s-1},\, b_{s-1}} \cdot \left(X_{s-1,\, b_{s-1},\, \ldots} \cdot T_{s,\, b_{s-1},\, \ldots}\right) :: \bigwedge \times(\cap) \bigvee +(\cup) \\[6pt]
    &\diamond : s \equiv L
  \end{aligned}
  $$

  <br/>

  ### Formulation B: Decimation-in-Frequency (DIF) Transpose-Free Input Cascade

  $$
  \begin{aligned}
    &\triangleright\ \text{Tensors} \\
    x^{N} &\to \text{Complex},\ \text{input time-domain sequence} \\
    F_2^{2,\, 2} &\to \text{Complex},\ \text{the fundamental } 2\times 2 \text{ butterfly matrix} \\
    X^{S,\, 2,\, \ldots,\, 2} &\to \text{Complex},\ L\text{-dimensional state tensor across stages } s \in \{0, \ldots, L\} \\
    T^{S,\, 2,\, \ldots,\, 2} &\to \text{Complex},\ \text{stage-dependent twiddle factor tensor} \\[6pt]
    &\triangleright\ \text{Initialization (Stage } s = 0\text{)} \\
    X_{0,\, b_{L-1},\, \ldots,\, b_0} &= x_{b_{L-1},\, \ldots,\, b_0} \quad (\text{Direct natural input, no transpose}) \\[6pt]
    &\triangleright\ \text{Extended Einsum (Butterfly stage } s \in \{1, \ldots, L\}\text{)} \\
    X_{s,\, k_{L-s},\, \ldots} &= \left((F_2)_{k_{L-s},\, b_{L-s}} \cdot X_{s-1,\, b_{L-s},\, \ldots} :: \bigwedge \times(\cap) \bigvee +(\cup)\right) \cdot T_{s,\, k_{L-s},\, \ldots} :: \bigwedge \times(\cap) \\[6pt]
    &\diamond : s \equiv L
  \end{aligned}
  $$
other_notes: |
  ### Inherent Parallelism
  At each stage $s$, contraction reduces only along the single active binary dimension $b_{s-1}$. All other $L-1$ ranks act as batch indices. Consequently, all $N/2$ butterfly operations at every stage are completely independent and can execute concurrently with zero inter-thread communication.

  ### Kronecker Product Equivalence
  The Einsum cascade corresponds directly to the classical factorization of the DFT matrix into sparse Kronecker matrix products:
  $$F_N = \left[\prod_{s=1}^L \left(I_{2^{L-s}} \otimes F_2 \otimes I_{2^{s-1}}\right) \cdot \mathbf{T}^{(s)}\right] \mathbf{P}_{\text{bit-rev}}$$
variants: |
  The cascade extends directly to general mixed-radix factorizations $N = p_1 \cdot p_2 \cdots p_M$:
  - **Radix-4 / Radix-8:** Uses higher-order butterfly kernels $F_4$ or $F_8$, reducing total non-trivial complex multiplications.
  - **General Mixed-Radix:** Decomposes arbitrary composite lengths into an $M$-stage cascade with complexity $O(N \sum_{m=1}^M p_m)$.
implementation_notes: |
  In hardware implementations (GPU / vector processors), tensor contractions along binary dimensions map directly to strided memory loads and SIMD vector butterfly shuffles.
complexity_costs: |
  - **Direct DFT:** $N^2$ complex multiplications $\implies O(N^2)$.
  - **Radix-2 FFT:** $L = \log_2 N$ stages, each computing $N/2$ butterflies $\implies \frac{N}{2} \log_2 N$ complex multiplications $\implies O(N \log N)$.
related_notes: |
  [Sparse Matrix-Vector Multiplication (SpMV)](spmv.html) uses similar sparse tensor contractions over compressed formats.
---
