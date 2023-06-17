---
layout: post
title: "Kotlin coroutines: Part 2"
tags: [ programming, kotlin, concurrency ]
---

In my [previous post](/2023/06/03/kotlin-coroutines-1.html), I introduced Kotlin coroutines and described my concurrent approach to solving the problem of merging `k` sorted lists into a single sorted list.

[//]: # (In this post, I am going to present the results of some performance tests I ran for my code, comparing the concurrent and non-concurrent versions.)

[//]: # (But before that.. let's temper our expectations by asking the question: What's the best we can expect?)

In this post, I am going to talk about what is the best we can expect by leveraging concurrency in the specific problem we chose. Can concurrency help us significantly? If yes, with what caveats?

### Time complexity (with concurrency)

In my previous post, I described that the time complexity is `O(N log(k))` (without concurrency) where `N` is the combined size of all the `k` lists.

Now let's expand on that.

`O(N log(k))` can be written as `O(N + N + .. + N)`, where the number of terms in the summation is `log k`.  
`= c1*(N + N + .. + N) + c2` where c1 and c2 are constants.

Each `N` above represents a single pass through all the lists. Let's say that the first `N` represents going through the `k` input lists and merging them into `k/2` lists. The second `N` represents going through `k/2` lists and merging them into `k/4` lists, and so on.  

Let's make the following assumptions:
1. We have a machine that is capable of as much parallelism as we need. There is no upper bound to the number of processors.
2. Concurrency has no overhead cost (scheduling, extra resource consumption by threads, etc.).
3. All the `k` input lists are the same size (`N/k` each).

The first two sound pretty unreasonable, but as you will see, they will help us see the bigger picture. The third assumption is mainly for simplifying our calculations.

Now with our assumptions, let's think about merging `k` lists into `k/2` lists. We can say that everytime we use an additional processor, we can reduce the time complexity of merging `k` lists by half but that stops at `k/2` processors. Beyond that point, there is no benefit of using additional processors.
We can apply the same reasoning to the idea of merging `k/2` lists into `k/4` lists, and so on.

So, with our version of concurrency, we can write the time complexity as follows:  
{::nomarkdown}
<img src="/assets/kotlin-2-equations-1.svg">
{:/}

Using the geometric series formula:  
{::nomarkdown}
<img src="/assets/kotlin-2-equations-2.svg">
{:/}

In the above equation, if we assume `k` to be sufficiently large then `(k-1)/k` can be ignored, and we can say that the time complexity is `O(N)`.

`O(N)` is an order of magnitude of improvement over `O(N log(k))` and we have now just shown that **O(N) is the best we can do in a very ideal world**.

However, if we think about real world scenarios, the number of processors is usually limited. My machine has `10` processors and that means that the best we can do is merge `20` lists at the same time. So we lose the benefit of concurrency as `k` increases beyond `20`. 

In conclusion, we showed that to truly take advantage of concurrency in this problem, we need the number of processors to scale linearly with `k`. If the number of processors is not comparable to `k` then we may still see some improvement but that may not be significant.