---
layout: post
title: "Reactive programming with Kotlin"
tags: [ programming, kotlin ]
full-width: true
---

I've recently been thinking about [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming) and found a [practice problem on Exercism](https://exercism.org/tracks/kotlin/exercises/react) that was fun to implement.


### Task description
 
Implement a system such that:  
* The system has two types of cells: Input cells and Compute cells. Input cell values are settable by the user. Compute cell values are not settable but are computed by functions that takes other cell values as input.  
* Implement updates so that when an input value is changed, values propagate to reach a new stable system state.
* Compute cells should allow for registering change notification callbacks. Call a cell’s callbacks when the cell’s value in a new stable state has changed from the previous stable state.

Exercism provides a set of tests using which we can determine the correctness of the solution. [Here are the tests](https://github.com/exercism/kotlin/blob/main/exercises/practice/react/src/test/kotlin/ReactTest.kt)  for this problem.
(I removed the `@Ignore` annotations while testing).

Let's look at a test for understanding the problem better:
```
@Test
fun computeCellsCanDependOnOtherComputeCells() {
    val reactor = Reactor<Int>()
    val input = reactor.InputCell(1)
    val timesTwo = reactor.ComputeCell(input) { it[0] * 2 }
    val timesThirty = reactor.ComputeCell(input) { it[0] * 30 }
    val output = reactor.ComputeCell(timesTwo, timesThirty) { (x, y) -> x + y }

    assertEquals(32, output.value)
    input.value = 3
    assertEquals(96, output.value)
}
```

The above test creates a single input cell, two compute cells that depend on the input cell, and an output (compute) cell that depends on the other compute cells. The test expects the output to reactively get updated when the input cell is updated.

<br/>
The test below tests for change notification callbacks:

```
@Test
fun callbacksOnlyFireOnChange() {
    val reactor = Reactor<Int>()
    val input = reactor.InputCell(1)
    val output = reactor.ComputeCell(input) { if (it[0] < 3) 111 else 222 }

    val vals = mutableListOf<Int>()
    output.addCallback { vals.add(it) }

    input.value = 2
    assertEquals(listOf<Int>(), vals)

    input.value = 4
    assertEquals(listOf(222), vals)
    }
```

<br/>
Another test that I thought was really cool was the implementation of the Adder digital circuit [here](https://github.com/exercism/kotlin/blob/main/exercises/practice/react/src/test/kotlin/ReactTest.kt#L190).
<br/>
<br/>

### Implementation (attempt \#1)

The problem came with [this](https://github.com/exercism/kotlin/blob/main/exercises/practice/react/src/main/kotlin/React.kt) skeleton implementation:

```
class Reactor<T>() {
    // Your compute cell's addCallback method must return an object
    // that implements the Subscription interface.
    interface Subscription {
        fun cancel()
    }
}
```

<br/>
**Here is my complete implementation:**  

```
interface Subscription<T> {
    val callback: (T) -> Unit
    fun cancel()
}

class SubscriptionImpl<T>(inputCallback: (T) -> Unit): Subscription<T> {
    private var canceled = false
    override val callback = { it: T -> if (!this.canceled) inputCallback(it) }
    override fun cancel() {
        this.canceled = true
    }
}

interface Cell<T> {
    var value: T
    val setValue: () -> Unit
}

class Reactor<T> {
    val cellToCellsMap = mutableMapOf<Cell<T>, MutableSet<Cell<T>>>()

    inner class InputCell(value: T): Cell<T> {
        override var value: T = value
            set(value) {
                field = value
                cellToCellsMap[this@InputCell]?.map {
                    it.setValue()
                }
            }
        override val setValue: () -> Unit = {}
    }

    inner class ComputeCell(
        vararg inputs: Cell<T>,
        private val computeLogic: (List<T>) -> T
    ) : Cell<T> {
        override var value = computeLogic(inputs.map {it.value})
        override val setValue: () -> Unit
        private val subscriptions = mutableListOf<Subscription<T>>()

        init {
            inputs.map {
                this@Reactor.cellToCellsMap.getOrPut(it) {mutableSetOf(this@ComputeCell)}
                    .add(this@ComputeCell)
            }
            setValue = {
                value = computeLogic(inputs.map {cell -> cell.value}).also { newValue ->
                    if (newValue != value) {
                        subscriptions.map {sub -> sub.callback(newValue) }
                    }
                }
                cellToCellsMap[this@ComputeCell]?.map {
                    it.setValue()
                }
            }
        }
        fun addCallback(callback: (T) -> Unit): Subscription<T> {
            return SubscriptionImpl(callback).also { subscriptions.add(it) }
        }
    }
}
```

<br/>

### Testing
I ran the tests and found that my code couldn't pass two (out of twenty) tests. The tests that failed are:
* [callbacksShouldOnlyBeCalledOnceEvenIfMultipleDependenciesChange](https://github.com/exercism/kotlin/blob/main/exercises/practice/react/src/test/kotlin/ReactTest.kt#L147) 
* [callbacksShouldNotBeCalledIfDependenciesChangeButOutputValueDoesntChange](https://github.com/exercism/kotlin/blob/main/exercises/practice/react/src/test/kotlin/ReactTest.kt#L164)  

Let's look at one of them (as we will see, the other one failed for the same reason):

```
@Test
fun callbacksShouldNotBeCalledIfDependenciesChangeButOutputValueDoesntChange() {
    val reactor = Reactor<Int>()
    val input = reactor.InputCell(1)
    val plusOne = reactor.ComputeCell(input) { it[0] + 1 }
    val minusOne = reactor.ComputeCell(input) { it[0] - 1 }
    val alwaysTwo = reactor.ComputeCell(plusOne, minusOne) { (x, y) -> x - y }

    val vals = mutableListOf<Int>()
    alwaysTwo.addCallback { vals.add(it) }

    for (i in 2..5) {
        input.value = i
    }

    assertEquals(listOf<Int>(), vals)
}
```

The above test failed because in my implementation, update events flowing through our system are not coordinated with each other. `plusOne` and `minusOne` cell values depend on the same `input` cell. However, updates to `plusOne` and `minusOne` independently trigger updates to the `alwaysTwo` cell.

<br/>

### Improved Implementation

I came up with the implementation below that passes all the tests including the two that failed before. What I did differently is as follows:  
* Defer firing all callbacks to *after* all descendants of an updated input cell are updated.
* Take snapshots of values of all descendants of an input cell before and after updating them.
* After all descendants are updated, take a diff of the snapshots and fire callbacks for only the compute cells that have been updated.

<br/>

```
interface Subscription<T> {
    val callback: (T) -> Unit
    fun cancel()
}

class SubscriptionImpl<T>(inputCallback: (T) -> Unit): Subscription<T> {
    private var canceled = false
    override val callback = { it: T -> if (!this.canceled) inputCallback(it) }
    override fun cancel() {
        this.canceled = true
    }
}

interface Cell<T> {
    var value: T
    val setValue: () -> Unit
}

class Reactor<T> {
    private val cellToCellsMap = mutableMapOf<Cell<T>, MutableSet<ComputeCell>>()

    private fun getAllDescendants(cell: Cell<T>): List<ComputeCell> {
        if (!cellToCellsMap.containsKey(cell) && (cell is ComputeCell)) return listOf(cell)
        return cellToCellsMap[cell]?.map { child -> getAllDescendants(child) }?.flatten() ?: listOf()
    }

    private fun takeSnapshot(inputCell: InputCell): Map<ComputeCell, T> {
        return getAllDescendants(inputCell).associateWith { it.value }
    }

    inner class InputCell(value: T): Cell<T> {
        override var value: T = value
            set(value) {
                field = value

                val beforeSnapshot = this@Reactor.takeSnapshot(this@InputCell)
                cellToCellsMap[this@InputCell]?.map { it.setValue() }
                val afterSnapshot = this@Reactor.takeSnapshot(this@InputCell)

                val updatedComputeCells = beforeSnapshot.keys.filter { key -> beforeSnapshot[key] != afterSnapshot[key] }
                updatedComputeCells.map { cell ->
                    cell.subscriptions.map {sub -> sub.callback(cell.value) }
                }
            }
        override val setValue: () -> Unit = {}
    }

    inner class ComputeCell(
        vararg inputs: Cell<T>,
        private val computeLogic: (List<T>) -> T
    ) : Cell<T> {
        override var value = computeLogic(inputs.map {it.value})
        override val setValue: () -> Unit
        val subscriptions = mutableListOf<Subscription<T>>()

        init {
            inputs.map {
                this@Reactor.cellToCellsMap.getOrPut(it) {mutableSetOf(this@ComputeCell)}
                    .add(this@ComputeCell)
            }
            setValue = {
                value = computeLogic(inputs.map {cell -> cell.value})
                cellToCellsMap[this@ComputeCell]?.map { it.setValue() }
            }
        }
        fun addCallback(callback: (T) -> Unit): Subscription<T> {
            return SubscriptionImpl(callback).also { subscriptions.add(it) }
        }
    }
}
```

