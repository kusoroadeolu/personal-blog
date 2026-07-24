# Elimination Backoff Stack

## Contention

Concurrency is hard. Making concurrent data structures performant is even harder. Though many things can cause a concurrent data structure to be unscalable, contention by multiple threads on a shared resource is usually the main perf killer. Hence why there multiple mechanisms to reduce contention on shared resources in concurrent data structures e.g. Striping, Flat combining, Bit reversal etc. Elimination is just one of these many contention reduction mechanisms.

### A Treiber Stack

A Treiber stack is one of the simplest concurrent data structures. However under high concurrency, its head pointer is a single point of contention due to multiple threads performing CAS operations and failing at the head of the stack.

**A Sample Implementation**

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

public class TreiberStack<T> {
    private volatile Node<T> head;

    public boolean push(T t) {
        var node = new Node<>(t);
        Node<T> h;
        do {
            h = loHead();
            node.spNext(h); //Backed by acquire write
        }while (!HEAD.compareAndSet(this, h, node));

         return true;
    }

    public T pop() {
        Node<T> h;
        Node<T> next;
        do {
            h = loHead();
            if (h == null) return null;
            next = h.lpNext();
        } while (!HEAD.compareAndSet(this, h, next));
        return h.value;
    }

    public String toString() {
        return head.toString();
    }

    private Node<T> loHead() {
        return (Node<T>) HEAD.getAcquire(this);
    }

    private static final VarHandle HEAD;
    private static final VarHandle NEXT;

    static {
        try {
            var l = MethodHandles.lookup();
            HEAD = l.findVarHandle(TreiberStack.class, "head", Node.class);
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
        }catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    static class Node<T> {
        final T item;
        volatile Node<T> next;

        Node(T item) {
           this.item = item;
        }

      Node<T> lpNext() {
         return NEXT.get(this);
      }

      void spNext(Node<T> next) {
        NEXT.set(this, next);
      }
    }
}
```

Usually, to solve this issue of contention, some treiber stacks implement exponential backoff on CAS failures, to increase the chances on that on the next CAS, we will succeed. During that backoff threads waste resources by idling, and success on the next CAS is not guaranteed.

## What is Elimination

Elimination is a contention reduction mechanism, which exploits the fact that when two inverse operations are applied to a shared resource, they cancel out each other.

Let's use a stack as an example. At its core, a stack is a **LIFO** based data structure which has two fundamental operations: push and pop which are the inverse of each other.

- A push operation replaces the head of stack with a new node and sets its next pointer to old head

- A pop operation replaces the head of the stack with its next pointer

Given a stack with an initial state:

A -> B -> C

where A is the head of the stack. If we pop the head of the stack then push a new A onto the stack, the stack remains unchanged. This also applies if we push a new value D onto the stack then pop the head, the stack also remains unchanged.

Now that we've understood the core idea of elimination, let's move to the high level pseudocode.

### Elimination Backoff Stack

The core idea is quite simple, if a pop / push operation fails on the stack, we then try to eliminate inverse operations on the array, if that fails, we back off then retry and so on

**High Level Pseudocode**

A thread tries to apply our operations on the given concurrent stack, if the operations on the stack fail after one attempt, it proceeds to elimination array. The thread then makes itself visible to other threads looking for operations to eliminate. It then looks for threads with inverse operations on the array to cancel out. Here, there are two main outcomes: successful elimination (either we find them or they find us), or we encounter a similar op and backoff

### It's not all sunshine and rainbows

The collision array helps in the sense when there's contention on the head of the stack and rather than just backing off and doing nothing useful, we shift that contention to a different structure where threads can possibly make progress rather than waiting idly

The collision array is highly dependent on an even ratio of inverse operations being applied. If the workload is highly asymmetric (higher push/pop operations than its counterpart), the structure basically decays to treiber stack with backoff.

### Adaptive Backoff

Unlike most stacks which use exponential backoff in time. The elimination array uses adaptive backoff which backs off both in space and time. Instead of just backing off in time which means a thread decreases its idle time after a failed collision, a thread backs off in space by increasing or reducing the range in the array in which it can operate after a collision failure or success.

The main idea behind this is, if a thread successfully collides or gets collided with by another thread multiple times, that thread can safely assume that it can increase its range in the collision array and that while it's waiting, it will get collided with, leading to no wasted work while waiting. Meanwhile, if a thread does not collide or get collided with frequently, it decreases its range in the collision array so as to collide or get collided with more frequently. It also decreases its time idle, to prevent wasting resources while getting no meaningful work done.

## Performance

I decided to run some benchmarks using JMH on this implementation against a treiber stack and a simple lock based stack. I tested 3 access patterns:

1. 50-50 Push Pop ratio
2. 75-25 Push Pop ratio
3. 100-0 Push Pop ratio

### Benchmark Setup

- JMH version: 1.37
- JVM: Java 25, HotSpot 64-Bit Server VM (25+37-LTS-3491)
- Benchmark mode: Throughput (ops/s)
- Warmup: 10 iterations × 1s each
- Measurement: 10 iterations × 1s each
- Forks: 3
- Thread configuration: 2 , 4, 8
- CPU Specs: Intel(R) Core(TM) i5-10300H CPU @ 2.50GHz (2.50 GHz), 4 cores, 8 processors

### Results

#### 50 - 50 Push Pop ratio

![50 Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vQJuAFuD8b4fx3S2cjq2Z0Cn7f0TRBEkDrRnCveGDeZe_SxTihNqXgOCRdMVjp_NYwcKJ_wXHASjjjh/pubchart?oid=24000677&format=image)

The elimination stack's thrpt is 2x that of the lock based stack and treiber stack under medium - low concurrency. This makes sense as when threads have near equal ratio of inverse operations to collide with, the collision probability in the elimination array increases significantly leading to less wasted work

**A brief follow up**

I did decide instrument the collision count across all threads to see what variables directly affect it, tweaking things like min/max range, min/max wait times, collision array size and wait strategies(PARK/SPIN). Surprisingly, across all threads, the only variables proportional to the collision count and thrpt were the number of threads, the push/pop ratio and the wait strategy (which was a short park in this scenario)

#### 75 - 25 Push Pop ratio

![75 Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vQJuAFuD8b4fx3S2cjq2Z0Cn7f0TRBEkDrRnCveGDeZe_SxTihNqXgOCRdMVjp_NYwcKJ_wXHASjjjh/pubchart?oid=759556672&format=image)

The elimination stack's thrpt decreases significantly due to an unequal access pattern except at 4 threads due to how my benchmark was run, at 4 threads we have an equal ratio of push and pop operations hence the uneven thrpt.

The treiber stack degrades after 2 threads,showing severe perf degradation under this access pattern.

#### 100 - 0 Push Pop ratio

![100 Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vQJuAFuD8b4fx3S2cjq2Z0Cn7f0TRBEkDrRnCveGDeZe_SxTihNqXgOCRdMVjp_NYwcKJ_wXHASjjjh/pubchart?oid=2137721075&format=image)

Under this access pattern, the elimination stack degrades into a treiber stack with a fancy backoff mechanism. Meanwhile, the lock based stack proves to be the most stable as its throughput barely moves from 2 - 8 threads




## Further Thoughts
Elimination is a pretty interesting contention control mechansims which could possibly also be used for other concurrent data structures like concurrent queues or linked lists. Also, we could possibly combine it with other control mechanisims like Flat Combining.

I would also love to benchmark my implementation on laptops which higher core counts e.g 16 cores or 32 cores as they can sustain true parallelism at 16/32 thread counts


## References
- Main Paper: http://www.inf.ufsc.br/~dovicchi/pos-ed/pos/artigos/p206-hendler.pdf
- GitHub: https://github.com/kusoroadeolu/elimination-backoff-stack

