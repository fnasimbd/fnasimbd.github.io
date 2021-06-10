---
layout: post
title: "Binary Search Reconsidered"
modified:
categories: blog
excerpt: "How to implement binary search algorithm correctly and modify it for different scenarios avoiding errors."
tags: ["algorithms", "problem-solving", "divide-and-conquer"]
comments: true
share: true
---

Binary search is a simple yet efficient divide and conquer algorithm for searching over sorted data. Beginners find it confusing. Even once mastered, implementing it for slightly different situations often introduces tricky errors mainly due to complicated and unnecessary modifications to it.

As we will see, symmetry allows using the same simple and elegant base implementation for various cases with very little to no modifications. The exploration proceeds in the following order:

- Implement binary search for finding upper bound of a value in a monotonically increasing sequence of integers.
- Using the implementation for lower bound with little modification and explanation of a popular bug (infinite loop).
- Modifying the implementation for monotonically decreasing sequences.

How Binary Search Works
===

We start with the typical binary search problem: given a **monotonically increasing** sequence of integers $$a$$ of length $$n$$ and an integer $$x$$, find the smallest value in the sequence that is $$\geq x$$ (**upper bound of $$x$$ in the sequence**). As we will see later, all other binary search problems are, in fact, slight modifications of this problem.

In binary search, we start assuming the entire sequence ($$[0 \ldots n-1]$$) as the search space and proceed by iteration. On each step of the iteration, we **reduce the search space by half** by checking the middle value $$a_{mid}$$ of the sequence.

- If $$a_{mid} < x$$, then obviously, as the sequence is monotonically increasing, all values in the lower half $$[lb \ldots mid]$$ are so too, therefore **not worth searching in;** in this case, discard the lower half and reduce the search space to the upper half ($$[mid + 1 \ldots ub]$$), which may contain values $$\geq x$$.

- On the other hand, if $$a_{mid} \geq x$$, then we have already found an upper bound; we have to **look for a smaller one,** which the lower half $$[lb \ldots mid]$$ may contain; so reduce the search space to $$[lb \ldots mid]$$ in this case.

On each iteration, the sequence is reduced by half, while maitaining the **invariant: $$[lb \ldots ub]$$ contains the solution.** In fact, to maintain this invariant, we don't exclude $$a_{mid}$$ in the second case. We terminate the algorithm when the sequence is **reduced to only one element**, that is, $$lb = ub$$. This last element is the upper bound we are looking for.

If all the elements in the sequence are $$< x$$, that is, **no element $$\geq x$$ exists,** then $$ub$$ reaches the end of the sequence. As a result, search result $$ub$$ may either mean that $$a_{ub}$$ is the upper bound or no upper bound is found. Line 19 checks for that case and returns an invalid index $$-1$$ in order to avoid ambiguity.

A sample implementation of the binary search algorithm for finding the upper bound of $$x$$ follows.

{% highlight cpp linenos %}
int bsearch(int a[], int n, int x)
{
    int lb = 0, ub = n - 1, mid;

    while (lb < ub)
    {
        mid = (lb + ub) / 2;

        if (a[mid] < x)
        {
            lb = mid + 1; // exclude the lower half (from 0 to mid)
        }
        else
        {
            ub = mid;
        }
    }

    if (a[mid] < x)
    {
        return -1; // no element >= x found
    }

    return ub;
}
{% endhighlight %}

Lines 9--16 performs the reduction by updating $$lb$$ and $$ub$$. At line 9, we check for the half of the sequence that certainly doesn't contain $$x$$ and exclude it from search accordingly at line 11; at line 15, we reduce the search sequence for a smaller upper bound.

What Goes Wrong with the Lower Bound Search?
===

What happens if we want to solve the opposite problem, finding the **lower bound of $$x$$ in the sequence:** the largest value $$\leq x$$? What modification might the preceding algorithm require? In this case, on each iteration, we exclude the upper half of the sequence if $$a_{mid} > x$$ and reduce search to the lower half otherwise. As a result, symmetrically, the reduction logic in the previous implementation (lines 9 to 16) changes to the following:

{% highlight cpp %}
if (a[mid] > x)
{
    ub = mid - 1; // exclude the upper half (mid to n - 1)
}
else
{
    lb = mid;
}
{% endhighlight %}

Moreover, in this case, $$lb$$ reaches the beginning of the sequence **if no value $$\leq x$$ exists;** invalid index $$-1$$ is returned in this case. 

{% highlight cpp %}
if (a[mid] > x)
{
    return -1; // no element <= x found
}
{% endhighlight %}

The resulting implementation, however, **falls into an infinite loop** at a degenerate case when the search space is reduced to 2, that is, $$ub = lb + 1$$, and $$a_{lb}$$ is a lower bound, which is always the case unless the sequence contains no lower bound. Notice that in this case **$$mid = lb$$ due to integer division** and as $$a_{mid}$$ is a lower bound the search space is updated to $$[mid \ldots ub]$$, that is, where it had been as $$mid = lb$$, and remains so forever.

**If $$lb$$ is updated at least once, then $$a_{lb}$$ is a potential lower bound.** We must seek improvement over it, that is, $$a_{ub}$$ in this case. The trick to accomplish this is to compute **ceiling** instead of floor as $$mid$$ in this case. A popular trick for computing ceiling using integer arithmetic during division is to add $$divisor - 1$$ ($$1$$ in this case) to the dividend.

{% highlight cpp %}
mid = (lb + ub + 1) / 2;
{% endhighlight %}

Though the upper bound variant uses the same $$mid$$ formula, it is not prone to the same problem since in the similar case, $$a_{ub}$$ is a potential upper bound and $$a_{lb}$$ is exactly the improvement over it to be checked. It must be obvious, however, that using this new $$mid$$ formula in the upper bound variant will symmetrically result into an infinite loop.

Which Index to Return?
===

During upper bound search it is conventional to return $$ub$$: index of the largest value $$\leq x$$. Symmetrically, the same reason applies to $$lb$$ for the lower bound case. Notice, however, that the algorithm terminates with $$lb = ub$$. So returning either of them in both upper bound and lower bound cases do.

Searching for Exact Values
===

What if we want to search for the exact value $$x$$ instead of its upper or lower bounds? **Any of the two variants will do:** if $$x$$ exists in the sequence, both ends up at that index on termination. Additionally, the check for upper/lower bound value's existence after termination has to be replaced with a check for existence of the exact value $$x$$ ($$a_{ub} = x$$ or $$a_{lb} = x$$) in order to make sure that $$x$$ exists in the sequence.

Multiple Occurrences of Result
===

If the result (upper bound, lower bound, or exact value) has multiple occurrences in the sequence, we might be interested in the first or last occurrence of it. In the upper bound case, we usually want the **first occurrence of the upper bound value** and the **last occurrence of the lower bound value** in the lower bound case. In both cases, the corresponding implementations need no change; they already terminate with the desired result.

In order to find the first or the last occurrence of the result in the exact value search, using the lower bound or the upper bound variant respectively will do.

Monotonically Decreasing Sequences
===

How does the algorithm change if the input sequence is reversed, that is, made **monotonically decreasing?**

In both variants, due to the reversal of the sequence and symmetry, **only the direction of sequence reduction gets reversed** and everything else remains the same. As long as the implementation is concerned, **the lower bound problem becomes the upper bound one** and vice versa; swapping the $$mid$$ calculation and sequence reduction logic does the trick.

Summary
===

With the base binary search implementation in mind, watching for the following three key factors help modifying it for different scenarios and also avoid a lot of errors:

- If the mid element is not a result, exclude the half including it.
- Conversely, if the mid element is a result, narrow down the sequence for smaller/larger results.
- Adjust the mid calculation to avoid infinite loops.
