# Lock Free Isn't always faster
There's a popular misconception that locks are slow. This misconception is usually caused by people using locks in ways where the cost of acquiring the lock is usually much more expensive than the actual work being done in the critical section. The major factor however which contributes to this misconception is `lock contention`; a performance bottleneck which occurs when multiple threads compete to access a locked resource. Unapollogetically, this conclusions are supported by profile data and benchmarks

Alas, there's a group of concurrent constructs which are usually termed `lock free` which provide non blocking progress guarantees allowing at least one thread to make progress at a time. They are usually coined to be faster compared to lock based structures. However, using locks to protect datastructures is much easier and usually the right choice earlier on as lock free data structures are notoriously harder to get right. There are multiple cases in which these lock free datastructures look and work correctly at the moment under rigorous stress tests, only for a race condition to show up months later in production. 

However, lock free data structures (even if correct) can be magnitudes of seconds slower compared to a lock-based structure; usually one that sympathises with modern computer hardware or more commonly known as **mechanical sympathy** by Martin Thompson.

To demonstrate this fact, I'll use 2 concurrent ordered lists I made which uphold the set invariant. A lock free linked list and a lock based unrolled concurrent list. These lists support three main functions `add`, `remove` and `contains`

The unrolled list provides a significant advantage over the lock free linked list. While the lock free linked list suffers from pointer chasing (a common issue with linked datastructures).The unrolled list, which still uses a linked structure, however it stores data in memory in continguous blocks (arrays) to reduce the number of nodes a thread needs to hop to find a value. This adheres to the principle of spatial locality; a principle in computer architecture which if a core accesses a memory location, it is highly likely to access nearby memory locations. 

If you recall, from my Cache Coherence article, most modern memory systems access data in contiguous 64 byte cache lines. Arrays arrange their elements are in such manner, so when data from an array is fetched, the CPU assumes surrounding data will be needed, hence all or most of the data in the array is already in the CPU's L1 or L2 cache, which can be accessed faster by the CPU compared to main memory

To put things into context, I've created a benchmark which measures the throughput of these structures under two workloads: a write heavy workload and a read heavy workload

## Benchmark Setup
- JMH version: 1.37
- JVM: Java 25, HotSpot 64-Bit Server VM (25+37-LTS-3491)
- Benchmark mode: Throughput (ops/s)
- Warmup: 10 iterations × 1s each
- Measurement: 10 iterations × 1s each
- Forks: 3
- Thread configuration: 8
- CPU Specs: Intel(R) Core(TM) i5-10300H CPU @ 2.50GHz (2.50 GHz), 8 cores

These experiments were performed on a keyspace of 10_000 integers generated at random with read workloads of ratio 90% contains, 9% adds and 1% removes and write workloads of 50% adds, 40% removes and 10% contain ops.

## Benchmark results
### Write Heavy
![Write Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=1729214476&format=image)

### Read Heavy
![Read Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=2083678510&format=image)

From these results, we can see the unrolled list's thrpt surpasses that of the lock free list by almost 50x for the write heavy workload and 80x for the read heavy workload, even though the lock free list explicitly avoids locks and provides lock free guarantees on the write and read path while the unrolled list uses a fine grained blocking approach. This shows that designing data structures with hardware in mind offers better performance than just designing data structures with progress guarantees in mind.

So yeah this was just a little experiment on my part. However 
As always the source code for these structures can be found on my github.

**Github**: https://github.com/kusoroadeolu/concurrent-lists