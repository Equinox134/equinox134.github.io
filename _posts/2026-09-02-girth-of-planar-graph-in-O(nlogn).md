---
title: Computing the Girth of a Planar Graph in O(nlogn) Time
author: Equinox134
date: 2026-09-02
categories:
  - Algorithm
tags:
  - Computer
  - Science
math: true
mermaid: true
excerpt: About a paper I read
---
Paper of interest:

\[ICALP09\] Computing the Girth of a Planar Graph in O(nlogn) Time

## Introduction

This paper does exactly what its title suggests: it presents an algorithm for finding the girth (least number of edges in a cycle) of a planar graph in $O(n\log n)$ time. In addition, the proposed algorithm also extends to graphs of bounded genus.

Also the algorithm itself isn't that complicated which is nice I guess. In addition, the algorithm does not require finding any embedding, unlike previous algorithms

## Preliminaries

Some terminology:

* A *planar embedding* of a graph assigns each vertex to a distinct point on a sphere and assigns each edge as a simple curve between the points such that no two cross each other. A graph is planar if it has a planar embedding.
* Consider the set of points on the sphere not assigned to a vertex or edge. The connected components of this set are the *faces* if the planar graph.
* If all vertices lie on a single face, then the graph is *outerplanar*.
* A graph is *$k$-outerplanar* if removing the vertices on the outer face results in a $(k - 1)$-outerplanar graph.
* A *genus* of a graph is the minimum number of handles that need to be added to a sphere in order for the graph to have an embedding.
* A *separator* is a set of vertices whose removal leaves no connected component of more than $2n/3$ vertices.

## The Algorithm

The general idea of the algorithm is to cover the given graph $G$ with $O(n/k)$ overlapping $k$-outerplanar graphs with $k = 2h$, where $h$ is an upper bound for the size of the faces of $G$. The cover is constructed so that the smallest cycle is contained entirely in an outerplanar graph, so we just have to find the smallest cycle in each of the $O(n/k)$ $k$-outerplanar graphs.

We'll start with an algorithm for finding the minimum length cycle in a $k$-outerplanar graph with nonnegative edge lengths in $O(kn\log n)$ time.

### $k$-outerplanar graphs with nonnegative edge lengths

The general idea is to use the fact that $k$-outerplanar graphs have separator of size $O(k)$. The algorithm starts by finding such a separator, which can be done in $O(n)$ time. Then it will make use of divide and conquer, where the subcases are finding the minimum cycle in each piece, and what we are left to do is to check cycles that might pass through multiple pieces. Such cycles will always pass through a separator vertex, so we'll have to find the minimum cycle that passes through a separator vertex.

The algorithm makes use of a single-source shortest path algorithm by Henzinger et al. that constructs the shortest path tree of a planar graph with nonnegative edge lengths in $O(n)$ time. By doing this for each of the vertices in the separator, we obtain the shortest path tree for each separator vertex in $O(kn)$ time.

To find the shortest cycle, we make use of the following lemma:

**Lemma 3.2** Let $G$ be a connected graph with positive edge lengths. If a vertex $v$ lies on a shortest cycle, and if $T$ is a shortest path tree from $v$, then there is a shortest cycle that passes through $v$ and has exactly one edge not in $T$.

The proof isn't that difficult.

**Proof**: Suppose the graph $G$ has a shortest cycle of length $s$ that passes through $v$ and let $C$ be such a cycle of length $s$ with the minimum number of edges that are not in $T$. Assume this number $k \geq 2$. Take a vertex $u \ne v$, then $u$ and $v$ divide $C$ into two parts. Choose $u$ such that each part contains an edge of $C$ not in $T$. Let $C_1$ and $C_2$ be the two split paths of $C$ from $v$ to $u$.

Let $P$ be a path in $T$ from $v$ to $u$. If $v$ and $u$ are the only vertices where $P$ intersects $C_1$ (or $C_2$), then we can create a new cycle with $P$ and $C_1$ (or $C_2$), and since this cycle has less than $k$ edges that are not in $T$, this is a contradiction.

Therefore there exists a vertex $x_1 \notin \lbrace u, v \rbrace$ where $P$ intersects $C_1$ and some vertex $x_2 \notin \lbrace u, v \rbrace$ where $P$ intersects $C_2$. Assume $x_1$ appears before $x_2$ in $P$. Then we can take the part of $P$ from $v$ to $x_2$ and the part of $C_2$ from $v$ to $x_2$ to create a new cycle, and again this cycle has less than $k$ edges not in $T$ so we get a contradiction. $\blacksquare$

With the above lemma we can use the following procedure to find the minimum cycle passing through $v$: for each edge $(x, y) \notin T$ whose length is $\ell(x, y)$ take the minimum over $dist_v (x) + dist_v (y) + \ell (x,y)$ where $dist_v(x)$ is the distance from $v$ to $x$. This procedure takes $O(n)$ time and if $v$ is part of a minimum cycle, we would find it, and if it is not, the value will always be larger.

Combining everything together we can find in $O(kn)$ time the shortest cycle that passes through a separator vertex. If removing the separator vertices results in $t \geq 2$ components, then the total time complexity of the recursive calls are

$$T(n) = T(n_1) + \cdots T(n_t) + O(kn)$$

where $\sum_{i=1}^t n_i \leq n$ and evert $n_i \leq 2n/3$. The solution to this recurrence is $T(n) = O(kn \log n)$ and thus we are done for this part.

Finally, we can apply this method to planar graphs, and since planar graphs have separators of size $O(\sqrt{n})$, the solving a similar recurrence gives us a time complexity of $O(n^{3/2})$ for finding the minimum cycle.

### Covering with $k$-outerplanar graphs

First we assume the graph $G$ is 2-connected. If not, we can decompose it into 2-connected components and try for each component. Also we can assume $G$ is not a cycle since that makes the problem trivial. Next, we are going to modify $G$ such that every edge is incident to a vertex with degree at least 3.

Since the graph is 2-connected there are no vertices with degree 1 or 0. Apply the following process repeatedly: for each vertex $u$ of degree 2 with neighbors $v_1$, $v_2$, if there is no edge $(v_1, v_2)$, remove $u$ and add an edge $(v_1, v_2)$ whose length is the sum of the lengths $(u, v_1)$ and $(u, v_2)$. Let the new graph be called $G^\prime$. The following lemma states two important properties of $G^\prime$.

**Lemma 3.3** In order to compute the girth of an $n$-vertex planar graph $G$, for which some embedding has minimum face length $h$, it suffices to compute the shortest cycle of the planar graph $G^\prime$, which has nonnegative edge lengths and $O(n/h)$ vertices.

**Proof**: Fix an embedding of $G$ which has a minimum face length $h$. Let $n$, $m$, $f$ be the number of vertices edges and faces of $G$, and $n^\prime$, $m^\prime$, $f^\prime$ be the same for $G^\prime$. Since the transformation doesn't change the number of faces $f = f^\prime$. We first show that $f = O(n/h)$. Let $F$ be the set of faces of $G$, and $\lvert x \rvert$ be the size of a face $x \in F$. Then

$$2m = \sum_{x \in F} \lvert x \rvert \geq \sum_{x \in F} h = fh$$

In any planar graph $m \leq 3n - 6$ so we get $f^\prime = f \leq 2m/h \leq 6n/h = O(n/h)$. Now it is left to show $n^\prime = O(n/h)$ or equivalently $m^\prime = O(n/h)$.

Let $T$ be the set of vertices of $G$ with degree at least 3 and $t = \lvert T \rvert$. Note that the vertices of $T$ appear in both $G$ and $G^\prime$. Also let $deg(v)$ be the degree of $v$ in $G$. The vertices of degree 2 in $G^\prime$ form an independent set, because otherwise it would have been deleted. So

$$\sum_{v \in T} deg(v) \geq m^\prime$$

Also

$$2(n^\prime - t) + \sum_{v \in T} deg(v) = 2m^\prime$$

By Euler's formula

$$m^\prime = n^\prime + f - 2 \leq n^\prime + 6n/h$$

Since $deg(v) \geq 3$ for all $v \in T$

$$m^\prime \leq \sum_{v \in T} deg(v) \leq 3\sum_{v \in T} (deg(v) - 2) = 6(m^\prime - n^\prime) \leq 36n/h$$

$\blacksquare$

Using the above lemma we can find an upper bound $h$ for the minimum face length of any embedding of $G$ by setting $h = \min \lbrace n, \lfloor 36n/n^\prime \rfloor \rbrace$.

To actually cover the graph with $k$-outerplanar graphs, do a BFS of $G^\prime$ from an arbitrary vertex $r$. Let $G^\prime_i$ be the graph induced by the vertices whose distance from $r$ is between $ki/2$ and $k+ki/2$ for $k = 2h$ and $i = 0, 1, \cdots \frac{2(n-k)}{k}$. This way each $G^\prime_i$ overlaps with at most 2 other graphs $G^\prime_{i-1}$ and $G^\prime_{i+1}$. One can check that each $G^\prime_i$ is $(k+1)$-outerplanar. Also, since the shortest cycle in $G^\prime$ has at most $h$ edges, the shortest cycle must be contained entirely in a single $G^\prime_i$.

Now we can run our $k$-outerplanar graph algorithm from before for each $G^\prime_i$. Each run takes $Ck\lvert G^\prime_i \rvert \log \lvert G^\prime_i \rvert$ time for some constant $C$. Therefore the total time complexity is

$$\sum_i Ck\lvert G^\prime_i \rvert \log \lvert G^\prime_i \rvert \leq 2Ch \log n \cdot \sum_i \lvert G^\prime_i \rvert$$

And since every vertex in $G^\prime$ appears in at most three $G^\prime_i$'s, $\sum_i \lvert G^\prime_i \rvert = O(\lvert G^\prime \rvert) = O(n/h)$ by Lemma 3.3. Therefore the total time complexity is $O(n\log n)$.

## Extension for Bounded Genus Graphs

There are a few things that need to be extended for the algorithm to work with graphs of bounded genus. First, the $O(n)$ single source shortest path algorithm can work if we can find a separator of size $O(n^{1- \varepsilon})$ in linear time. Therefore the SSSP algorithm works for graphs of bounded genus.

In the algorithm we used the fact that $k$-outerplanar graphs have separators of size $O(k)$. In fact, we can think of the arguments using $k$-outerplanar graphs as arguments that use the fact that the graph has treewidth $O(k)$, as the separator and subgraph arguments all follow. Now lets think we did BFS on the bounded genus graph and take some $G^\prime_i$ to be the graph induced by layers $s$ to $s + k$. We can contract all vertices in layers above $s$ into a single vertex, and since the resulting graph is a minor, it still has the same genus. Also, there is a result saying that the bounded genus graphs have separators and treewidth of the same order as their diameter. Therefore $G^\prime_i$ in this case also has treewidth $O(k)$ and thus all previous arguments hold.

For Lemma 3.3, we can use the fact that in graphs with genus $g$ $m \leq 3n - 6 + 6g$ hold. As $g$ is bounded we still obtain $f = O(n/h)$. Similarly we can use $m^\prime = n^\prime + f - 2 + 2g$, and since $g$ is bounded this still gives us $m^\prime = O(n/h)$.

Combining everything together we can see that the algorithm works for graphs of bounded genus as well.

## Conclusion and Extra Questions

So this was an algorithm to find the girth of a planar (and bounded genus) graphs in $O(n \log n)$ time. The current task I was given was to try and expand this to minor free graphs. I know that there is an algorithm for SSSP in $O(n)$ time for minor closed graph classes, but the other parts need some more thought.
