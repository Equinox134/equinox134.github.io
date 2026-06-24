---
title: Round to Odd + Correct Rounding
author: Equinox134
date: 2026-06-24
categories:
  - Math
  - Research
tags:
  - Computer
  - Science
math: true
mermaid: true
excerpt: Seminar for FPC Lab I did about round to odd
---

Papers 

- \[TC 08] Emulation of a FMA and Correctly Rounded Sums: Proved Algorithms Using Rounding to Odd ([https://ieeexplore.ieee.org/document/4358278](https://ieeexplore.ieee.org/document/4358278 "https://ieeexplore.ieee.org/document/4358278"))
- \[POPL 22] One polynomial approximation to produce correctly rounded results of an elementary function ([https://dl.acm.org/doi/10.1145/3498664](https://dl.acm.org/doi/10.1145/3498664 "https://dl.acm.org/doi/10.1145/3498664"))

Mainly focus on the polynomial approximation part

This is a rough memo I made for myself, don't expect it to be nice

Also here's the presentation slide I made

<a href="/assets/files/round to odd/FPC_Lab__seminar_2026_06_24.pdf" style="display:flex;align-items:center;gap:16px;max-width:600px;padding:16px 20px;border:1px solid #e0e0e0;border-radius:8px;text-decoration:none;color:inherit;"> <span style="font-size:28px;">📁</span> <span style="flex:1;"><span style="font-size:16px;">FPC_Lab__seminar_2026_06_24.pdf.pdf</span><br><span style="font-size:13px;color:#888;"></span></span> <span style="font-size:20px;">↓</span> </a>

# Round to Odd

Round to odd is a nonstandard rounding mode. However, using for some intermediate values instead of round to nearest, we can obtain correctly rounded results at the end of computation.

"Correct rounding principle": the result should be same as if it was first computed with infinite precision, then rounded to the precision of the destination format

Standard format: IEEE-754 floating point numbers
But, there may exist higher precision formats, and some registers can use them (ex, Intel x86 floating point units use 80 bit long registers)
problem:
1. Setting a target precision in the register -> computation matches destination precision, but setting a target precision is costly
2. Double rounding: rounded to the extended precision first, then to a target precision -> leads to unexpected inaccuracy

Double rounding not always a threat -> use a new rounding mode for the first rounding -> when a real is not representable, round to adjacent fp with an odd mantissa, aka round to odd

Rounding to odd can produce more accurate results without extra precision

## Double Rounding

### FP Definition

$(n,e)$ -> $n \times 2^e$ : integral signed mantissa $n$, integral signed mantissa $e$
format denoted by $\mathbb{B}$, consists of lowest exp $-E$ and precision $p$
--> overflow doesn't matter, so no largest exponent
$(n,e)$ is representable if $\lvert n \rvert < 2^p$ and $e \geq -E$
$\mathbb{F}$ is the subset of real numbers represented by $\mathbb{B}$

round to nearest even denoted by $f = \circ(x)$
--> $f$ is the fp number closest to $x$. If $x$ is halfway, choose one with even mantissa
faithful rounding : $f$ is a faithful rounding of $x$ if it is either rounded up or down

### Double Rounding Accuracy

Double rounding : round to a high precision first, then to a lower one

Extended precision $\mathbb{B}_e = (p+k,E_e)$
Working precision $\mathbb{B}_w = (p,E_w)$

If same rounding (nearest), then can lead to less precise results than after one rounding

bound:

$$\lvert f - x \rvert \leq \left(\frac{1}{2} + 2^{-k-1}\right) \text{ulp}(f)$$

Double rounding's are faithful

**Theorem 1 (DblRndStable)** \
Let $R_e$ be a faithful rounding in extended precision $\mathbb{B}_e = (p+k,E_e)$ and let $R_w$ be a faithful rounding in the working precision $\mathbb{B}_w = (p,E_w)$. If $k \leq 0$ and $k \leq E_e - E_w$, then, for all real values of $x$, the floating-point number $R_w(R_e(x))$ is a faithful rounding of $x$ in the working precision

Last requirement -> $e^\text{min}_e \leq e^\text{min}_w$, which is equivalent to saying any normal fp number with respect to $\mathbb{B}_w$ should be normal with respect to $\mathbb{B}_e$

## Rounding to Odd

Rounding to odd does not belong to IEEE 754, should not be mistaken for rounding to nearest odd

Denote by $\bigtriangleup$ rounding toward $+\infty$, $\bigtriangledown$ rounding toward $-\infty$
Rounding to odd is defined by

$$\square_\text{odd}(x) = \begin{cases}
x \quad &\text{if } x \in \mathbb{F} \\
\bigtriangleup(x) & \text{if its mantissa is odd} \\
\bigtriangledown(x) & \text{otherwise}
\end{cases}$$

$\square_\text{odd}(x)$ is not necessarily the nearest fp number with odd mantissa, this is wrong when $x$ is close to a power of 2.

**Theorem 2 (To_Odd)** Rounding to odd has the properties of a rounding mode:
* Each real can be rounded to odd
* Rounding to odd is faithful
* Rounding to odd is monotone
Moreover,
* Rounding to odd can be expressed as a function: A real cannot be rounded to two different floating-point values
* Rounding to odd is symmetric: If $f = \square_\text{odd}(x)$ then $-f = \square_\text{odd}(-x)$

### Implementation

1. Round to zero : $\mathcal{Z}(x)$
2. Perform logical OR between inexact flag (or sticky bit) $\iota$

why this works?

Another way to round to odd with $p + k$ bits:
* round to zero with $p+k-1$ bits
* concatenate the inexact bit at the end of the mantissa

both are aimed at hardware implementation

might not be effective for software implementation

### Correct Double Rounding

**Theorem 3 (To_Odd_Even_Is_Even)** Assuming $p \geq 2$, $k \geq 2$, and $E_e \geq 2 + E_w$,

$$\forall x \in \mathbb{R}, \space \circ^p\left(\square^{p+k}_\text{odd}(x)\right) = \circ^p(x)$$

proof is split into 3 cases:
* $x$ is in the exact middle of two consecutive fp numbers
* $x$ is slightly different from the midpoint
* far from the midpoint

Algorithms like FMA, adding multiple numbers can be done using this

# Polynomial Approximation to Produce Correctly Rounded Results

--> proposes a method to generate a single polynomial approximation that produces correctly rounded results for all inputs, multiple rounding modes and multiple precision configurations. \
Idea: generate a polynomial approximation for a representation with $n+2$ bits using round-to-odd

## Rounding Components

Rounding components consist of $(s,v^-,rb,sticky)$ from the real value $v_\mathbb{R}$.

$s$ : represents the sign and identifies whether $v_\mathbb{R}$ is positive or negative \
$v^-$ : The smaller of $v_{sm}$ and $v_{lg}$ in magnitude where

$$\begin{align}
v_{sm} = \max\{v \in \mathbb{F}_{n,\lvert E \rvert} \space \vert \space v \leq v_\mathbb{R}\} \\
v_{lg} = \min\{v \in \mathbb{F}_{n,\lvert E \rvert} \space \vert \space v \geq v_\mathbb{R}\}
\end{align}$$

the other is $v^+$

$rb$ : the rounding bit, $(n+1)^{th}$-bit of $v_\mathbb{R}$ in an extended precision representation. This determines whether $v_\mathbb{R}$ is closer to $v^-$ or $v^+$ \
$sticky$ : the sticky bit, the bitwise OR of all bits of $v_\mathbb{R}$ starting from the $(n+2)^{th}$-bit. Together with $rb$ we can tell if $v_\mathbb{R}$ is in the exact middle or where it is.

## RLibm Approach & Main Method

RLibm approach for polynomial approximation, $\mathbb{T}$ is the target representation

1. use an oracle to compute correctly rounded results of a function $f(x)$ for each input $x \in \mathbb{T}$
2. identify an interval $[l,h]$ around the correctly rounded result s.t. any value in $[l,h]$ rounds to the correct value --> rounding interval
	1. Polynomial evaluation, range reduction, output compensation happen in a representation with higher precision $\mathbb{H}$, rounding intervals are also in $\mathbb{H}$
3. range reduction to transform $x$ to $x^\prime$. The generated polynomial will approximate result for $x^\prime$, then use output compensation to produce final correctly rounded result for $x$
	1. Both range reduction and output compensation happen in $\mathbb{H}$ and can experience numerical errors, this should not affect the generation of correctly rounded results
	2. Must deduce intervals for reduced domain so that the polynomial evaluation over the reduced input produces correct results for the original inputs
	3. In other words, find an interval $[l^\prime, h^\prime]$ that when $P(x^\prime)$ falls in there, after output compensation the value lives in $[l,h]$
	4. Use the inverse of the output compensation function to find this interval
4. synthesize a polynomial of a degree $d$ using some LP solver that satisfies constraints like $l^\prime \leq P(x^\prime) \leq h^\prime$
	1. Provides more freedom in generating polynomials

So the main idea is similar to the above approach, instead we find the rounding intervals in relation to round-to-odd, and call these intervals odd intervals.

First, the paper defines round-to-odd as follows:

$$v_{ro} = RN_{\mathbb{T},ro} (v_\mathbb{R}) = \begin{cases}
s \times v^- \space &\text{if } IsOdd(v^-) \lor (rb = 0 \land sticky = 0) \\
s \times v^+ &otherwise
\end{cases}$$

where $v_\mathbb{R}$ is a real value, $s$ is the sign bit, $rb$ is the rounding bit, and $sticky$ is the sticky bit. Also, $v^-$ and $v^+$ are the largest and smallest floating point numbers such that $v^- \leq v_\mathbb{R} < v^+$.

The goal is to find a polynomial approximation $A_\mathbb{H}$ such that $RN_{\mathbb{T}_k,rm} (A_\mathbb{H}(x)) = RN_{\mathbb{T}_k,rm} (f(x))$, where $A_\mathbb{H}$ is implemented in representation $\mathbb{H}$. The main insight is that if the polynomial approximation produces correct results in $\mathbb{T}_{n+2}$< then it produces correct results for any representation $\mathbb{T}_k$ for any rounding mode.

More specifically:

**Theorem 1** Let $v_\mathbb{R} = f(x)$ be a real values result of an elementary function and $v_{ro} = RN_{\mathbb{T}_{n+2},ro}(v_\mathbb{R})$. Let $v$ be a value in the odd interval of $v_{ro}$. Consider a rounding mode $rm \in \{rn, ra, rz, ru, rd\}$. Then,

$$RN_{\mathbb{T}_k,rm}(v) = RN_{\mathbb{T}_k,rm}(v_\mathbb{R})$$

proof is provided later.

## The Algorithm

The high-level view of the algorithm to generate the polynomial is as follows:

1. Calculate results of the function $f$ in round-to-odd mode for all possible inputs (via an oracle like MPFR)
2. For each result, calculate the corresponding odd intervals
	1. There are cases when the intervals are a single point (when the rounded value is even), which are called singletons
	2. singletons significantly reduce the freedom of the possible polynomials possible, and so are handled separately using properties of elementary functions
3. Given the intervals, find the polynomial satisfying them using RLibm's polynomial generator (explained above)

More in-depth of each step:

Computing the round-to-odd result from a real value:

find the correctly rounded result $y_{ro}$ for input $x$. Compute each $y = f(x)$ using an oracle like MPFR, then obtain the rounding components $(s, v^-, rb, sticky)$, and use them to compute $y_{ro}$

Deducing the odd interval of an input:

We have to find intervals in $\mathbb{H}$ such that producing any value in that interval rounds to $y_{ro}$, which is called the odd interval. If $y_{ro}$ is even, then it is a singleton. Else, all values in $\mathbb{H}$ strictly greater than the preceding value of $y_{ro}$ and less the succeeding forms the odd interval.

Piecewise polynomial generation using the odd intervals:
Create a piecewise polynomial using LP

Implementation of the polynomial approximation for $\mathbb{T}_k$:

Given $x$, check if it's odd interval is a singleton, if so use a table look up or function-specific properties. Otherwise perform range reduction, use Horner's method for polynomial evaluation to compute round-to-odd result in $\mathbb{T}_{n+2}$, and round that result to $\mathbb{T}_k$.

### Handling Singletons

Singletons only happen if the real value matches the round-to-odd value exactly. Both $\mathbb{T}_n$ and $\mathbb{T}_{n+2}$ are finite precision, so all values are rational. Therefore we have to find rational inputs that produce rational outputs.

This is done by using mathematical properties of functions. For example, $e^x$, by the Lindemann-Weierstrass theorem, if the input $x$ is a non-zero rational, then $e^x$ cannot be rational, so the only possible singleton is $x = 0$. This is done for other functions like $2^x, \log_2(x)$ and so on.

## Proof of Main Theorem

**Lemma 1** The round-to-odd result $v_{ro}$ in $\mathbb{T}\_{n+2}$ preserves the sign of $v_\mathbb{R}$

Pf) The only value that rounds to 0 is 0

**Lemma 2** Let $v_{ro} = RN_{\mathbb{T}\_{n+2},ro}(v_\mathbb{R})$. The first $(n+1)$-bits of $v_{ro}$ and $v_\mathbb{R}$ are identical.

Pf) $v_{ro}$ is created using rounding components $(s_{v_{ro}}, v^-\_{v_{ro}}, rb_{v_{ro}}, sticky_{v_{ro}})$. The round-to-odd mode preserves the sign, so we assume $v_\mathbb{R}$ is positive. Also $v^-_{v_{ro}}$ is a truncated value or $v_\mathbb{R}$, so all the $(n+2)$-bits of $v^-\_{v_{ro}}$ and $v_\mathbb{R}$ are identical. After rounding $v_{ro}$ is either $v^-_{v_{ro}}$ or the successor of $v^-\_{v_{ro}}$. We now show that the $(n+1)$-bits of $v_{ro}$ and $v_\mathbb{R}$ are identical.

If $v^-_{v_{ro}}$ is odd, then $v_{ro} = v^-_{v_{ro}}$. Therefore all $(n+2)$-bits of $v_{ro}$ and $v_\mathbb{R}$ are identical. Else $v^-\_{v_{ro}}$ is even. So the last bit of $v^-\_{v_{ro}}$ is 0. (1) if $rb_{v_{ro}} = 0$ and $sticky_{v_{ro}} = 0$ then $v_{ro} = v^-\_{v_{ro}}$, so the lemma holds. (2) if $rb_{v_{ro}} \ne 0$ and $sticky_{v_{ro}} \ne 0$ then $v_{ro}$ is equal to the successor of $v^-\_{v_{ro}}$. The only bit that changes between $v^-\_{v_{ro}}$ and the successor is the $(n+2)^{th}$-bit. So the first $(n+1)$-bits are the same.

**Lemma 3** The $(n+2)^{th}$-bit of $v_{ro}$ is equal to the bitwise OR of all the bits of $v_\mathbb{R}$ starting from the $(n+2)^{th}$-bit

Pf) Use similar strategy to lemma 2. Intuitively the lemma states that the last bit of $v_{ro}$ is 0 if and only if all bits starting from the $(n+2)^{th}$-bit of $v_\mathbb{R}$ is 0. Again assume $v_\mathbb{R}$ positive, and the rounding components are $(s_{v_{ro}}, v^-\_{v_{ro}}, rb_{v_{ro}}, sticky_{v_{ro}})$. Again $v_{ro}$ is either $v^-\_{v_{ro}}$ or it's successor.

If $v^-\_{v_{ro}}$ is odd, then $v_{ro} = v^-\_{v_{ro}}$. Then the $(n+2)^{th}$-bit of $v^-\_{v_{ro}}$ is 1. Since $v^-\_{v_{ro}}$ is a truncated $v_\mathbb{R}$ the $(n+2)^{th}$-bit of $v_\mathbb{R}$ is 1. So the bitwise OR of bits of $v_\mathbb{R}$ starting from the $(n+2)^{th}$-bit is 1, which is equal to the $(n+2)^{th}$-bit of $v_{ro}$.

Else $v^-\_{v_{ro}}$ is even. There are two subcases. The first is when $rb_{v_{ro}} = 0$ and $sticky_{v_{ro}} = 0$, so $v_{ro} = v^-\_{v_{ro}}$. The $(n+2)^{th}$-bit of $v^-\_{v_{ro}}$ is 0. So is the $(n+2)^{th}$-bit of $v_\mathbb{R}$. By definition, $rb_{v_{ro}}$ is the value of the $(n+3)^{th}$-bit and $sticky_{v_{ro}}$ is the bitwise OR of all bits starting from the $(n+4)^{th}$-bit. So the bitwise OR of all bits starting from the $(n+2)^{th}$-bit is 0, which is the $(n+2)^{th}$ bit of $v_{ro}$.

In the next subcase $rb_{v_{ro}} \ne 0$ and $sticky_{v_{ro}} \ne 0$. In this case $v_{ro}$ is equal to the successor of $v^-\_{v_{ro}}$ which is odd. So the $(n+2)^{th}$-bit of $v_{ro}$ is 1. By similar reasons as above, some bit after the $(n+2)^{th}$-bit in $v_\mathbb{R}$ is 1. So the lemma holds.

**Lemma 4** Let $(s_1, v_1^-, rb_1, sticky_1)$ and $(s_2, v_2^-, rb_2, sticky_2)$ be the rounding components for two real values $v_1$ and $v_2$ in rounding them to a FP representation $\mathbb{T}\_n$. If $s_1 = s_2, v_1^- = v_2^-, rb_1 = rb_2$ and $sticky_1 = sticky_2$, then $RN_{\mathbb{T},rm} (v_1) = RN_{\mathbb{T},rm}(v_2)$ for any rounding mode $rm$.

Pf) Follows directly from the definition of rounding components.

### Proof that Double Rounding the Round-to-Odd Result Produces Correct Results for all $\mathbb{T}_k$

Here's a sketch of the proof.

We prove the following:

**Theorem 2** Given a real number $v_\mathbb{R}$, representations $\mathbb{T}\_k$ and $\mathbb{T}\_{n+2}$ with same number of exponent bits that satisfy $\lvert E \rvert + 1 < k \leq n$, and a rounding mode $rm \in \{rn, ra, rz, ru, rd\}$, then $RN_{\mathbb{T}\_k,rm} (v_\mathbb{R}) = RN_{\mathbb{T}\_k,rm} (RN_{\mathbb{T}\_{n+2},ro}(v_\mathbb{R}))$.

Let $v_{ro} = RN_{\mathbb{T}\_{n+2},ro}(v_\mathbb{R})$. By lemma 4, we just have to check if the rounding components for $v_\mathbb{R}$ and $v_{ro}$ are the same. Again, round-to-odd preserves sign, so we assume positive.

Let $(s_1, v_1^-, rb_1, sticky_1)$ and $(s_2, v_2^-, rb_2, sticky_2)$ be the rounding components for $v_\mathbb{R}$ and $v_{ro}$ respectively in $\mathbb{T}\_k$. Let the representation of $v_\mathbb{R}$ in extended infinite precision representation $B_{v_\mathbb{R}}$ be

$$B_{v_\mathbb{R}} = b_1 b_2 \cdots b_{k-1} b_k b_{k+1} \cdots b_n b_{n+1} b_{n+2} b_{n+3} \cdots$$

By definitions of rounding components the following is true:

$$B_{v_1^-} = b_1 b_2 \cdots b_{k-1} b_k, \quad rb_1 = b_{k+1}, \quad sticky_1 = b_{k+2} \vert b_{k+3} \vert \cdots$$

By lemma 2 and lemma 3:

$$B_{v_{ro}} = b_1 b_2 \cdots b_{k-1} b_k \cdots b_n b_{n+1} t, \quad t = b_{n+2} \vert b_{n+3} \vert \cdots$$

Since $k \leq n$ there are at least one bit between $b_k$ and $t$. The rounding components are:

$$B_{v_2^-} = b_1 b_2 \cdots b_{k-1} b_k, \quad rb_2 = b_{k+1}, \quad sticky_2 = b_{k+2} \vert b_{k+3} \vert \cdots \vert b_{n+1} \vert t$$

Comparing this with $v_\mathbb{R}$ we can see that all are the same, hence the rounded result is the same via lemma 4. Theorem 1 follows directly after theorem 2.

## Conclusion

This is the first work on creating a single polynomial approximation that works for all representations and rounding modes.
