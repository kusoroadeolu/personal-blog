# Lock Free Isn't always faster
There's a popular misconception that lock free data structures are always faster. However, lock free data structures (even if correct) can be handle less operations per microsecond compared to a lock-based structure that sympathises with modern computer hardware or more commonly known as **mechanical sympathy** by Martin Thompson.

To demonstrate this fact, I'll use 2 concurrent lists I made. A lock free linked list and an unrolled concurrent list.

The unrolled list provides a significant advantage over the lock free linked list. While the lock free linked list suffers from pointer chasing (a common issue with linked lists/queues).The unrolled list, which still uses a linked structure, however:

1. Stores data in batches (arrays) to reduce the number of nodes a thread needs to hop to find a value 

2. More importantly, adheres to the principle of spatial locality; a principle in computer architecture which if a core accesses a memory location, it is highly likely to access nearby memory locations. 

If you recall, from my Cache Coherence article, memory systems access data in 64 byte cache lines and arrays are contingously arranged, so when data from an array is fetched, the cpu assumes surrounding data will be needed, hence all the needed data in the array is already in the CPU's L1 or L2 cache, which is much faster than accessing main memory.

Anyways, enough talking, let's get into the graphs lol

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
![Write Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=2083525584&format=image)

### Read Heavy
![Read Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=1477816088&format=image)

From these results, we can see the unrolled list's thrpt surpasses that of the lock free list by almost 50x for the write heavy workload and 80x for the read heavy workload, even though the lock free list explicitly avoids locks and provides lock free guarantees on the write and read path while the unrolled list uses a fine grained blocking approach. This shows that designing data structures with hardware in mind offers better performance than just designing data structures with progress guarantees in mind.

So yeah this was just a little experiment on my part and I am excited to build more concurrent data structures with hardware architecture in mind

As always the source code for these structures can be found on my github.

**Github**: https://github.com/kusoroadeolu/concurrent-lists