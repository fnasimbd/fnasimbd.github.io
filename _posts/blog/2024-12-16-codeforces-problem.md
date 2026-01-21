---
layout: post
title: "Codeforces 992 (Div. 2) Problem C"
modified:
categories: blog
excerpt: "An interesting problem."
tags: ["algorithms", "problem-solving", "codeforces"]
comments: true
share: true
---

Codeforces Round 992 (Div. 2) [C. Ordered Permutations](https://codeforces.com/contest/2040/problem/C).

## The problem statement

The problem starts by defining the following function for any permutation of integers $$1$$ to $$n$$:

$$S(p) = \sum_{1 \leq l \leq r \leq n} \min(p_l, p_{l+1}, \dots , p_{r})$$

The function is actually the _sum of the minimum elements of all possible subarrays of a permutation $$p$$._

For a given $$n$$, the problem asks to _find the $$k$$-th permutation in lexicographical order among the permutations giving the maximum or report that there is none._

## Permutations yielding maximum $$S(p)$$

The first task is to identify the permutations that yield the maximum value of $$S(p)$$. Let's observe the case for $$n = 3$$.

$$
[1], [1, 2], [1, 2, 3] = 1 + 1 + 1\\
[2], [2, 3] = 2 + 2
[3] = 3
$$

```
[1], [1, 2], [1, 2, 3] = 1 + 1 + 1
[2], [2, 3] = 2 + 2
[3] = 3

[1], [1, 3], [1, 3, 2] = 1 + 1 + 1
[3], [3, 2] = 3 + 2
[2] = 2

[2], [2, 1], [2, 1, 3] = 2 + 1 + 1
[1], [1, 3] = 1 + 1
[3] = 3

[2], [2, 3], [2, 3, 1] = 2 + 2 + 1
[3], [3, 1] = 3 + 1
[1] = 1
```

Here, the maximum sum is $$10$$, achieved only by the permutations $$[1, 2, 3]$$, $$[1, 3, 2]$$, and $$[2, 3, 1]$$. Now, what is the characteristic of these permutations?

The first clue comes from observing the position of $$1$$ in the permutations. Notice that in the maximum sum permutations, $$1$$ is always either at the beginning or at the end. It is because _if $$1$$ is in the middle, it occurs as the minimum value in more subarrays, reducing the overall sum._

If this is the case for the smallest element $$1$$ in the overall permutation, the same must be the case for the next smallest element $$2$$ in the remaining subarray. Observing the permutations above confirms the fact.

It generalizes to this: **each element either at the beginning or at the end of the subarray.**

```
[2, 4, 3, 1]

[2], [2, 4], [2, 4, 3], [2, 4, 3, 1] = 2 + 2 + 2 + 1 = 7
[4], [4, 3], [4, 3, 1] = 4 + 3 + 1 = 8
[3], [3, 1] = 3 + 1 = 4
[1] = 1

[1, 2, 3, 4]

[1], [1, 2], [1, 2, 3], [1, 2, 3, 4] = 1 + 1 + 1 + 1
[2], [2, 3], [2, 3, 4] = 2 + 2 + 2
[3], [3, 4] = 3 + 3
[4] = 4

[1, 3, 2, 4]

[1], [1, 3], [1, 3, 2], [1, 3, 2, 4] = 1 + 1 + 1 + 1
[3], [3, 2], [3, 2, 4] = 3 + 2 + 2
[2], [2, 4] = 2 + 2
[4] = 4

[1, 3, 4, 2]

[1], [1, 3], [1, 3, 4], [1, 3, 4, 2] = 1 + 1 + 1 + 1
[3], [3, 4], [3, 4, 2] = 3 + 3 + 2
[4], [4, 2] = 4 + 2
[4] = 4

[2, 3, 1, 4]

[2], [2, 3], [2, 3, 1], [2, 3, 1, 4] = 2 + 2 + 1 + 1
[3], [3, 1], [3, 1, 4] = 3 + 1 + 1
[1], [1, 4] = 1 + 1
[4] = 4
```

Given that each element can be either at the beginning or at the end, there are total $$2^{n-1}$$ permutations that gives the maximum value of $$S(p)$$.

## Finding the $$k$$-th permutation

Then comes the question of how to sort them lexicographically.

| Permutation | Binary Encoding | Explanation
|-------------|-----------------|-------------
| `[1, 2, 3]` | `0 0 0` | All elements remain in their positions.
| `[1, 3, 2]` | `0 1 0` | The second element in the subarrary `[2, 3]` moved to the end.
| `[2, 3, 1]` | `1 0 0` | Only the first element moved to the end; remaining subarray `[2, 3]` remains the same.
| `[3, 2, 1]` | `1 1 0` | Both the first and second elements moved to the end.

The ordering can be encoded as a binary string of length $$n-1$$. Each set bit indicates that the element at that position is to be moved to the end.

```
[1, 2, 3, 4]
[1, 2, 4, 3]
[1, 3, 4, 2]
[1, 4, 3, 2]
[2, 3, 4, 1]
[2, 4, 3, 1]
[3, 4, 2, 1]
[4, 3, 2, 1]
```

## Caveats

* There may not be as many as $$k$$ permutations with maximum sum. In those cases, the problem asks to return $$-1$$.
* Checking for $$k$$ needs bit additional care, because $$2^{n-1}$$ will overflow as the bound of $$n$$ $$2 × 10^{5}$$ is very large. For $$n > 40$$ the number of desired permutations ($$1,099,511,627,776$$ close to $$10^{12}$$) exceeds the bound of $$k$$ ($$10^{12}$$). Therefore, the check $$k < 2^{n-1}$$ is necessary only for $$n < 40$$.
