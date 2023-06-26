---
layout: post
title: "Kotlin coroutines: Part 1"
tags: [ programming, kotlin, concurrency ]
---

I have been learning Kotlin recently with a special focus on concurrency with coroutines. In this post, I will describe how I took advantage of concurrency to solve a specific problem.

### What are coroutines (from Kotlin documentation<sup>[1]</sup>)?  
A coroutine is an instance of suspendable computation. It is conceptually similar to a thread, in the sense that it takes a block of code to run that works concurrently with the rest of the code. However, a coroutine is not bound to any particular thread. It may suspend its execution in one thread and resume in another one.

### Why use coroutines instead of doing things the usual way (thread pools)
Thread pools are the usual and recommended way of writing concurrent programs in Java. Coroutines have the following advantages over thread pools:
1. Coroutines are a light-weight abstraction over threads and don't map directly to OS threads, unlike Java `Thread` instances. This means that you can launch way more coroutines than threads in a thread pool. We can do this because coroutines are 'suspendable' (they don't block threads) and a lot more coroutines can be run on a single thread.
2. Coroutines support the model of [structured concurrency](https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency) which had advantages like fewer memory leaks and built-in cancellation support. Structured concurrency also ensures that any errors in the code are properly reported and are never lost.

### Problem: How to merge k sorted lists into a single sorted list?  
Let's solve the problem of merging k sorted lists such that the resulting list is sorted. For example:  
When given these 3 lists as input:  
`[2, 3, 5]`  
`[-1, 1, 2, 30, 239]`  
`[-5, -3, -1, 8, 22, 39]`

We can expect an output as follows:  
`[-5, -3, -1, -1, 1, 2, 2, 3, 5, 8, 22, 30, 39, 239]`

Let's first look at a simple solution to this problem (without concurrency):
{% highlight kotlin linenos %}
fun mergeKSortedLists(lists: List<List<Int>>): List<Int> {
    val queue = ArrayDeque<List<Int>>()
    lists.forEach { queue.add(it) }
    while(queue.size > 1) {
        queue.addLast(
          mergeTwoSortedLists(
            queue.removeFirst(),
            queue.removeFirst()
          )
        )
    }
    return queue.removeFirst()
}
{% endhighlight %}

What we did above is as follows:
1. Put all `k` input lists in a queue
2. Pop two lists from the start of the queue and call the `mergeTwoSortedLists(list1, list2)` function on them. We will look at the `mergeTwoSortedLists` function in a little bit.
3. Push the merged list to the end of the queue.
4. Repeat steps 2 and 3 until only 1 list remains in the queue. This should happen in `k-1` iterations.
5. Return the single list remaining in the queue as the answer.

Now let's look at the `mergeTwoSortedLists()` function:
{% highlight kotlin linenos %}
fun mergeTwoSortedLists(list1: List<Int>, list2: List<Int>): List<Int> {
    var index1 = 0
    var index2 = 0
    val result = mutableListOf<Int>()
    while(index1 < list1.size && index2 < list2.size) {
        if(list1[index1] < list2[index2]) {
            result.add(list1[index1])
            index1 += 1
        } else {
            result.add(list2[index2])
            index2 += 1
        }
    }
    result.addAll(list1.slice(index1 until list1.size))
    result.addAll(list2.slice(index2 until list2.size))
    return result.toList()
}
{% endhighlight %}

#### Time complexity:  
If the `k` lists combined are of size `N`, the time complexity of the above solution is `O(N log(k))`.  
*Reason*: Every time we pass through all `N` items we reduce the number of lists `k` by half. We stop when we have a single list.

### Solution with coroutines

{% highlight kotlin linenos %}
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.joinAll
import kotlinx.coroutines.Job
import kotlinx.coroutines.launch

suspend fun concurrentlyMergeKSortedLists(lists: List<List<Int>>): List<Int> = coroutineScope {
    val channel = Channel<List<Int>>(1)
    val jobs = mutableListOf<Job>()
    repeat(lists.size) {
        jobs.add(
            launch {
                channel.send(lists[it])
            }
        )
    }
    repeat(lists.size - 1) {
        jobs.add(
            launch {
                channel.send(
                    mergeTwoSortedLists(
                        channel.receive(), channel.receive()
                    )
                )
            }
        )
    }
    jobs.joinAll()
    channel.receive()
}
{% endhighlight %}

### Explanation
Let's try to understand a few keywords in the above code:
1. `suspend`: The suspend keyword means that this function is suspendable and can be run as a coroutine.
2. `coroutineScope`: Creates a `CoroutineScope` and calls the suspend function with this scope. You can see that we are using `launch` (which is a coroutine builder) inside the function which launches a child coroutine. In Kotlin, every child coroutine must be defined within a `CoroutineScope`, which is why we need to call `coroutineScope`.
3. `Channel`: A `Channel` is conceptually very similar to the Java `BlockingQueue`. One key difference is that channels use suspending operations (`send` and `receive`) instead of blocking operations (`put` and `take`). In this problem, the usage of a regular `Queue` (used in my non-concurrent solution above) maps very nicely to `Channel` / `BlockingQueue`. `Channels` and its different types are documented in detail [here](https://kotlinlang.org/docs/coroutines-and-channels.html#channels) and [here](https://kotlinlang.org/docs/channels.html).
4. `launch` and `Job`: `launch` launches a new coroutine without blocking the current thread and returns a reference to the coroutine as a `Job`.

What we are doing in the concurrent code is analogous to what we did in the non-concurrent implementation earlier. Here are the equivalent steps:
1. The first `repeat` block sends all lists to `channel`, similar to how we put all lists in a `queue` earlier. Each `send` operation, however, can be run asynchronously because it's defined within `launch`. Also notice that we defined the channel's size to be `1`. What that means is that the channel will only actually store 1 item in the queue, and the remaining send operations will be suspended.
2. The second `repeat` block launches `k-1` coroutines. Each of these coroutines is suspended until two lists are ready to be received from the `channel`. When two lists are received, they are merged using `mergeTwoSortedLists` and sent to the channel.
3. All the coroutines launched above can be referenced from the `jobs` list. `jobs.joinAll()` suspends the coroutine corresponding to the `concurrentlyMergeKSortedLists` function until all the child coroutines (referenced in `jobs`) finish execution. It's important to note here that we are talking in terms of suspending rather than blocking which is a major advantage.
4. In the end, we know that there will be exactly `1` list left in the channel after all coroutines launched above have completed execution. The result of the last expression inside `CoroutineScope` (`channel.receive()`) is automatically returned. 

### Closing thoughts and next steps
I really enjoyed learning about the powerful concurrency primitives in Kotlin! Learning about suspendable functions and coroutines forced me to think about concurrency in a completely different way. As a next step, I am planning to run some performance tests to see how the two implementations compare in practice.

All the code that I described in this blog post can be [accessed here](https://github.com/ankushb92/LearningKotlin/blob/main/src/main/kotlin/mergeKSortedLists.kt).

[1]: https://kotlinlang.org/docs/coroutines-basics.html
