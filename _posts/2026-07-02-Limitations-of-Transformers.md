---
title: Theoretical Limitations of multi-layer Transformer
author: Equinox134
date: 2026-07-02
categories:
  - Research
  - AI
tags:
  - Computer
  - Science
math: true
mermaid: true
excerpt: Some notes on the paper
---
Paper of interest:

\[FOCS22] Theoretical Limitations of multi-layer Transformer (https://arxiv.org/abs/2412.02975)

## Introduction

This paper tries to answer the question "Does the architecture of transformers have any potential limitations or weaknesses?". The paper investigates this question from a computational perspective.

**Small vs Large Transformers** : We look at the prompt length $n$ as a growing parameter. The total number of parameters for a transformer is roughly $L \cdot \text{poly}(dHp)$, where $L$ is the number of layers, $d$ is the head embedding dimension, $H$ the number of attention heads, and $p$ the bit precision for each entry in the embedding. One can think of $p$ as $\Theta (\log n)$. We consider a transformer small if $dHp \leq n^{o(1)}$ and large if $dHp \geq n^\epsilon$ for some constant $\epsilon > 0$.

The following is the main result, which is the first unconditional lower bound for any constant layer transformer, unconditional meaning it doesn't rely on unproven conjectures like $\text{TC}_0 \neq \text{NC}_1$.

**Theorem 1.1** (Lower bound for multi-layer Transformer) \
Let $H$ be the number attention heads, $d$ be the head dimension, $p$ the precision, $L$ be the number of layers, $n$ be the prompt length. For any $L \leq \tilde{O} (\log \log (n))$, an $L$-layer decoder-only Transformer could not solve $L$-sequential function composition whenever $Hdp \leq n^{2^{-4L}}$.

$L$-sequential function composition in a nutshell is computing $f_L(w_L,i_{L-1})$ given functions $f_1 \cdots f_L$ and queries $w_1 \cdots w_L$ where $i_k = f_k(w_k,i_{k-1})$.

The paper gives some applications of this, but I won't go over them as their not my main focus right now.

### Very Brief Overview

The paper creates an autoregressive communication model to model the action of transformers, then show that a lower bound for the autoregressive communication model is a lower bound for the transformer as well. Then they prove that $L$-sequential function composition is impossible on a small transformer by showing something called an indistinguishable decomposition exists via induction.

## Preliminaries

### Transformer

We start by formally describing what a transformer is.

Some notation:
* $[n] = \{1, 2, \cdots, n\}$
* $[n_1, n_2] = \{n_1, n_1 + 1, \cdots, n_2\}$
* $[0] = \emptyset$

$L, H, d, p$ is the same as used above.

An $L$-layer transformer can be thought of as a sequence-to-sequence network alternating between MLP and attention layers.

$$f_\text{tran} = f_\text{mlp}^{(L)} \circ f_\text{attn}^{(L)} \circ \cdots \circ f_\text{mlp}^{(1)} \circ f_\text{attn}^{(1)}$$

Given an input sequence $x^{(0)} = (x_1^{(0)}, \cdots, x_n^{(0)}) \in (\mathbb{R}^{dH})^n$, the Transformer computes the output of the $l$-th attention layer $y^{(l)} = (y_1^{(l)}, \cdots, y_n^{(l)})$ and the output of the $l$-th MLP layer $x^{(l)} = (x_1^{(l)}, \cdots, x_n^{(l)})$. For each layer

**Attention** $f_\text{attn}^l$ : for $h \in [H]$ and $i \in [n]$

$$y_i^{(l,h)} = \sum_{j < i} \alpha_{i,j}^{(l,h)}V^{(l,h)}x_j^{(l-1)} \in \mathbb{R}^d$$

$\alpha$ is that thing that transformer computes using the query and key values. The final output is the concatenation of each head.

**MLP** $f_\text{mlp}^l$ : output of the $l$-th layer is a function $g^{(l)} : \mathbb{R}^{dH} \rightarrow \mathbb{R}^{dH}$ applied to each position:

$$x_i^{(l)} = g^{(l)}(y_i^{(l)}) \in \mathbb{R}^{dH}$$

### Sequential Function Composition

**Definition 2.1** ($L$-sequential function composition) \
A $L$-sequential function composition task $L-\text{FuncComp}(w,z_0,z_1,\cdots,z_L)$ is described a sequence of functions $z_0, z_1, \cdots, z_L$, where $z_0 \in [m]$ and $z_l : [N_{l-1}] \rightarrow [N_{l-1}]$ for $l \in [L]$ and a query $w = (w_1, \cdots , w_{L-1}) \in [n_1] \times \cdots [n_{L-1}]$, one computes

$$i_0 = z_0 \in [m], \quad i_1 = z_1(i_0) \in [N_0]$$

and one inductively computes for each $l = 1, 2, \cdots, L-1$

$$i_2 = z_2(w_1, i_1) \in [N_1], \cdots, i_{l+1} = z_{l+1} (w_l, i_l) \in [N_l]$$

The final output is taken as $L-\text{FuncComp}(w, z_0, z_1, \cdots, z_L) = i_L$

Parameters are chosen as:

$$K = (HdpL)^8 \cdot 8^{2L^2}, \quad m = K^{\sum_{l \in [0 : L-1]} 8^l +1}, \quad n_l = K^{4 \cdot 8^{L - l -1}} \quad \forall l \in [L-1]$$

and

$$N_l = m \cdot \prod_{l^\prime \in [l]} n_l \quad \forall l \in [0 : L-1]$$

which makes absolutely no sense right now but hopefully they will later in the proof. According to GPT they come from reverse engineering these:

$$x_{l-1} = x_l^8, \quad, n_l = x_l^4, \quad m = (x_0x_1 \cdots x_{L-1})^8, \quad N_l = N_{l-1}n_l$$

## Autoregressive Communication Model

The main thing of this paper.

**The Autoregressive Communication Model**

**Setting**: $L$ be the number of epochs. There are $L+2$ players called $[-1,L]$. They communicate for $L$ epochs. $B$ is the message bits of a single token.

**Input**: Player $i \in [-1:L]$ receives $m_{(i)}$ tokens $z_i$ as input.

**Communication**: The communication proceeds in $L$ epochs. For $l \in [0 : L]$, let $X_i^{(l)}$ be the message collected by player $i$ after the $l$-th epoch of communication.

When $l = 0$, $X_i^{(0)}$ is the input of player $i$.

For $l = 1, 2, \cdots, L$, the $l$-th epoch of communication proceeds as follows. For player $i$:
* $i$ sends its information $X_i^{(l-1)}$ to all players $[i+1 : L]$
* $j \in [i+1 : L]$ based on its own info and $i$'s info, sends a message $\Pi_{j,i}^{(l)}$ to $i$. The length of the message satisfies $\lvert \Pi_{j,i}^{(l)} \rvert = 2B \cdot m_{(i)}$.
* $i$ updates its collection of info as $X_i^{(l)} := X_i^{(l-1)} \cup \bigcup_{j>i} \Pi_{j,i}^{(l)}$.

**Output**: At the end of the $L$-th round, player -1 outputs a message based on its information $X_{-1}^{(L)}$.

$L-\text{FuncComp}$ in the autoregressive model can be represented like the following, where $B = Hdp$.

* Player $i \in [L]$ receives $z_i$ from the definition, described in $N_{i-1}$ tokens. Player 0 and -1 receives $z_0$ and $w$ as input, each described in 1 token.
* At the end of the $L$-th round, player -1 must output  $L-\text{FuncComp}(w, z_0, z_1, \cdots, z_L) = i_L$

### Autoregressive communication lower bounds imply Transformer lower bounds

The following lemma shows that proving a lower bound for the above communication model immediately implies the desired lower bound against decoder-only Transformers.

**Lemma 3.1** (Reduction from Transformers to autoregressive communication) If there is an $L$-layer decoder-only Transformer that solves the $L$-sequential function composition task, then there is a deterministic autoregressive communication protocol that solves $L-\text{FuncComp}$ with $L$ epochs and $B = Hdp$ message bits.

I'll skip the proof as it's not that important for now. It's mostly just constructing the input tokens and communication protocol from a transformer to a communication model.

This is basically saying that proving bounds for the autoregressive model (which apparently is more tractable, I guess?) automatically shows bounds for the Transformer. The more interesting part is showing that bound.

## Autoregressive Communication Lower Bound

The main thing we want to prove is the following:

**Theorem 4.1** There is no deterministic autoregressive communication protocol solving $L-\text{FuncComp}$ with $L$ epochs and $B = Hdp$ bits.

The proof is long and convoluted and honestly kind of impossible to understand immediately as there are just too many weird notations and definitions that just seem to come out of nowhere. I'll try to give an overall layout first, and hopefully that will make understanding the proof easier.

### Sketch of Proof

We define two sets $Z_{<l}$ and $R_{\geq l}$ where $R_{\geq l}$ is the set of input assignments to players $[l : L]$ and $Z_{< l}$ is the set of input assignments to players $[-1 : l-1]$. We call this an **indistinguishable decomposition** if for every possible inputs $z_{< l} \in Z_{< l}$, all assignments to $R_{\geq l}$ are indistinguishable to players $[-1 : l-1]$ on inputs $z_{< l}$ after $l$ epochs. So basically, for all inputs in $R_{\geq l}$, any assignment in $Z_{<l}$ gives the same output.

A very very high level explanation of the proof is that we are going to create an indistinguishable decomposition $Z_{< L}$ and $R_{\geq L}$, such that the size of $R_{\geq L}$ is large and the possible outputs we can get from $Z_{< L}$ is also large.

By definition of indistinguishable decomposition, all assignments of $R_{\geq L}$, i.e. the inputs for player $L$ does not matter at all for the output. However, because the size of $R_{\geq L}$ is large, there must exist two inputs for player $L$ that gives different outputs for the $L$-sequential function decomposition problem, which will lead to a contradiction.

The more detailed proof is just defining all the ambiguous terms more rigorously and actually constructing the indistinguishable set via induction. I don't think I will go into it super detailed (mostly because I just can't understand it for gods sake), but just slightly dip my toe in it.

The notations in the paper are very confusing (at least for an idiot like me), so lets try to debunk that first.

### Notations

| Symbol                                                       | Meaning                                                                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| $\tilde{z}$                                                  | a specific fixed value of an input                                                                           |
| $A_l$                                                        | the set of all possible inputs player $l$ can hold                                                           |
| $A_l = [N_{l-1}]^{N_{l-1}}$                                  | for $l \in [L]$ all functions $[N_{l-1}] \rightarrow [N_{l-1}]$                                              |
| $A_0 = [m]$, $A_{-1} = [n_1] \times \cdots \times [n_{L-1}]$ | seed domain & query domain                                                                                   |
| $n$                                                          | prompt length                                                                                                |
| $H$, $d$, $p$                                                | attention heads, head dimension, precision                                                                   |
| $B = Hdp$                                                    | message bits per token                                                                                       |
| $K = (HdpL)^8 \cdot 8^{2L^2}$                                | master constant everything is sized against                                                                  |
| $m$                                                          | seed domain size                                                                                             |
| $n_l$                                                        | query-coordinate size at level $l$                                                                           |
| $N_l = m \prod_{l^\prime \leq l} n_{l^\prime}$               | domain size at level $l$                                                                                     |
| $x_l = K^{8^{L-l-1}}$                                        | how many inputs are kept per player $l$ (i.e. $\lvert Z_l \rvert$)                                           |
| $\Delta_l$                                                   | shrink budget, how much $R$ may lose                                                                         |
| $\Theta_l$                                                   | cover floor, minimum number of reachable chain values                                                        |
| $\mathcal{I} (Z_{<l})$                                       | the set of all reachable $i_{l-1}$ values using kept inputs                                                  |
| $\Pi_{j,i}^{l^\prime}$                                       | the message sent from $j$ to $i$ n epoch $l^\prime$, depends on inputs $z_i, \cdots, z_L$ (not sure why tho) |
| $\Lambda^l$, $\Lambda^{l,l^\prime}_{j,i}$                    | the frozen transcript, depends only on early inputs $z_{l-1}, \cdots, z_i$                                   |
| $\Phi$, $\Psi$                                               | candidate frozen transcript                                                                                  |

A formal definition of indistinguishable decomposition is as follows:

Let $l \in [2 : L]$,

$$R_{\geq l} \subseteq A_L \times A_{L-1} \times \cdots \times A_l$$

and

$$Z_{< l} = Z_{-1} \times \cdots Z_{l-1} \subseteq A_{-1} \times \cdots \times A_{l-1}, \quad \text{where } Z_{- 1} = A_{-1}, Z_i \subseteq A_i$$

We say $R_{\geq l}$ and $Z_{< l}$ is an indistinguishable decomposition if for every $\tilde{z}\_{<l} \in Z_{<l}$ and for every $\tilde{\alpha}\_{\geq l}, \tilde{\beta}\_{\geq l} \in R_{\geq l}$ satisfies:

$$\Pi_{j, i}^{l^\prime} (\tilde{z}\_{< l}, \tilde{\alpha}\_{\geq l}) = \Pi_{j,i}^{l^\prime} (\tilde{z}\_{< l}, \tilde{\beta}_{\geq l})$$

i.e. the input for the players above $l$ doesn't matter.

The main idea of the proof is the following lemma:

**Lemma 4.3** Let $\Pi$ be an $L$-epoch $Hdp$ message bits autoregressive communication protocol. If there is an indistinguishable decomposition $R_{\geq L}$ and $Z_{<L}$ such that:

1. (Large remaining entropy) $\lvert R_{\geq L} \rvert \geq \lvert A_L \rvert / \Delta_L$
2. (Large cover) $\lvert \mathcal{I}\_{L-1} (Z_{<L}) \rvert \geq \Theta_{L-1}$

Then $\Pi$ does not solve $L-\text{FuncComp}$

This leads to the next lemma

**Lemma 4.4** For every $L$-epoch $Hdp$ message bits autoregressive communication protocol $\Pi$, there is an indistinguishable decomposition $R_{\geq L}$ and $Z_{<L}$ satisfying the requirements of Lemma 4.3

Therefore  just have to show that such a decomposition always exists, then by Lemma 4.3 no autoregressive communication protocol can solve $L$-sequential function composition, proving the lower bound.

The proof of Lemma 4.3 uses the pigeonhole principle. Making use of large remaining entropy and large cover, one can show that

$$\lvert R_{\geq L} \rvert \geq \frac{\lvert A_L \rvert}{(N_{L-1})^{n_{L-1}\lvert \mathcal{I}\_{L-1} (Z_{< L})\rvert}}$$

Because of that, there exists $z_L^\prime, z_L^{\prime\prime} \in R_{\geq L}$ such that $z_L^\prime(\tilde{w}\_{L-1}, \tilde{i}\_{L-1}) \ne z_L^{\prime\prime}(\tilde{w}\_{L-1}, \tilde{i}_{L-1})$ by the pigeonhole principle. However, by the definition of indistinguishable decomposition, player -1 cannot distinguish between the two inputs, so the output is the same regardless of what input is given, which leads to a contradiction.

What remains is actually constructing such a decomposition, which is done by induction, and is arguably the hardest part of the entire paper. Induction is done on the epoch number $l$.
