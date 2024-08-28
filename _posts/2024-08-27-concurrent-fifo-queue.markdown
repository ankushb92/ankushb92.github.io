---
layout: post
title: "Concurrent FIFO Queue implementations in Java and C#"
tags: [ programming, C#, java, concurrency, data structures ]
full-width: true
---

I recently got a chance to dive into the implementation details of concurrent (thread-safe) data structures and also techniques for implementing them lock-free. In this post, I want to specifically discuss the FIFO (First In, First Out) Queue.

### Concurrent FIFO Queue with locking

_An implementation with locking is as follows:_ <br/>
* Use two locks: `enqueueLock` and `dequeueLock`.
* Any `Enqueue` operation/thread needs to acquire the `enqueueLock` and similarly any `Dequeue` operation needs to acquire the `dequeueLock`.
* If a `Dequeue` operation is invoked and the Queue is empty, it will block other `Dequeue` operations until an `Enqueue` operation is performed.
* We can have a max-size for the Queue and `Enqueue` operations would block other `Enqueue` operations until the the size reduces below max-size.

_Note 1_: The above is how Java `BlockingQueue` is implemented (particularly `LinkedBlockingQueue`).
<br/>
_Note 2_: `BlockingCollection` is the C# equivalent for Java `BlockingQueue`. It's implemented as a wrapper over the lock-free `ConcurrentQueue` and uses a semaphore for controlling access when the collection is full or empty.

As a concrete example, below is a snippet of the implementation of `LinkedBlockingQueue` in Java ([source](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/java/util/concurrent/LinkedBlockingQueue.java)):

(`put`/`take` are the same as `Enqueue`/`Dequeue` operations described above)

{% highlight java linenos %}
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    
    private final int capacity;
    private final AtomicInteger count = new AtomicInteger(0);
    private transient Node<E> head;  // head of linked list
    private transient Node<E> last; // tail of linked list
    private final ReentrantLock putLock = new ReentrantLock();
    private final ReentrantLock takeLock = new ReentrantLock();
    private final Condition notEmpty = takeLock.newCondition();
    private final Condition notFull = putLock.newCondition();

    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                notFull.await();
            }
            last = last.next = node;
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
    .
    .
}
{% endhighlight %}

The upsides of the locking approach is:
* It is thread-safe.
* It works well in producer-consumer scenarios. Blocking in the empty queue scenario creates backpressure from consumers to producers.

The downsides:
* Acquiring and releasing of locks has a performance overhead, which introduces additional latency to operations.
* Locking and blocking prevent multiple threads from performing operations concurrently, resulting in reduced throughput.

### Lock-free

#### Why lock-free?

While we saw above that a locking based implementation is possible, lock-free (and non-blocking) implementations exist to avoid the downsides of locking and blocking: increased latency and reduced throughput. `ConcurrentQueue` in C# and `ConcurrentLinkedQueue` in Java are lock-free implementations.

#### Implementation

The lock-free implementations in both Java and C# are inspired by a 1996 paper by Michael-Scott titled "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms":<br/>
[https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf](https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf)

A very important component of the lock-free implementation is the atomic [CAS](https://en.wikipedia.org/wiki/Compare-and-swap) (compare-and-swap) operation. Java provides `AtomicReference.compareAndSet` for this and C# provides us `Interlocked.CompareExchange`.

A brief description of the algorithm:
* The underlying data structure we use is a linked list where each node contains an item and a pointer to the next node.
* We use compare-and-swap (CAS) operations to update the head and tail pointers, ensuring that concurrent operations on the queue are thread-safe without locking.
    * Enqueue: a new node is created and linked as the new tail using CAS.
    * Dequeue: the head pointer is updated to the next node using CAS.

If you'd like to see concrete implementations:
* Java implementation can be [found here](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/java/util/concurrent/ConcurrentLinkedQueue.java).
* C# implementation can be [found here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/collections/Concurrent/ConcurrentQueue.cs).


### Async implementation

Another interesting way to implement a Concurrent Queue is by providing asynchronous programming capabilities to users (async/await in C#).<br/>
C# provides us with the `Channels` [library](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels) for this. This is an async (non-blocking) alternative to `BlockingCollection` we discussed above for producer-consumer scenarios.

Without going into the implementation details of `Channels`, let's look at an example of the producer/consumer pattern from the [official documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels#producer-patterns).

#### Producer example
{% highlight csharp linenos %}
static async ValueTask ProduceWithWhileWriteAsync(
    ChannelWriter<Coordinates> writer, Coordinates coordinates)
{
    while (coordinates is { Latitude: < 90, Longitude: < 180 })
    {
        await writer.WriteAsync(
            item: coordinates = coordinates with
            {
                Latitude = coordinates.Latitude + .5,
                Longitude = coordinates.Longitude + 1
            });
    }

    writer.Complete();
}
{% endhighlight %}

#### Consumer example
{% highlight csharp linenos %}
static async ValueTask ConsumeWithAwaitForeachAsync(
    ChannelReader<Coordinates> reader)
{
    await foreach (Coordinates coordinates in reader.ReadAllAsync())
    {
        Console.WriteLine(coordinates);
    }
}
{% endhighlight %}

### Closing thoughts

Both Java and C# provide a variety of implementations of Concurrent FIFO Queue depending on various use cases like producer-consumer, high throughput, asynchronous, etc.<br/>
Building a good understanding of these may be helpful while navigating codebases that use one or more of the different types of these data structures and concurency primitives; or in choosing the right implementation for the task at hand while building new software or refactoring existing code for scalability.
