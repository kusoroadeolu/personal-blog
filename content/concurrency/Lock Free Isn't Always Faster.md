# Lock-Free Isn't Always Faster

Lock-free data structures are supposed to be the fast option. No blocking, no threads waiting on each other, no lock contention grinding things to a halt. So when I benchmarked a lock-free linked list against a lock-based structure, I expected the lock-free one to win.

It didn't. It lost by up to 80x.

## Wait, aren't locks the slow ones?

That's the assumption baked into most concurrency advice: locks are slow, avoid them if you can. And it's not entirely wrong  but it's usually a misdiagnosis. The thing that actually kills performance isn't the lock itself, it's `lock contention`: multiple threads piling up to fight over the same locked resource. If your critical section is tiny and the lock acquisition costs more than the work being protected, sure, locks look slow. But that's a symptom of bad lock usage, not proof that locks are inherently slow. Profiling and benchmark data back this up consistently.

Enter `lock-free` data structures. These are structures that guarantee at least one thread always makes progress, no blocking involved. On paper, they sound like the obvious upgrade. In practice, they're notoriously hard to get right. It's common to see a lock-free structure pass rigorous stress tests, ship to production, and then have a race condition surface months later that nobody caught. Locks, by comparison, are just easier to reason about, which is why they're usually the right default early on.

Here's the part that surprised me though: even when a lock-free structure *is* correct, it can still be dramatically slower than a well-designed lock-based one. So I built two structures to find out why.

## The setup

Two concurrent ordered lists, both upholding the set invariant, both supporting `add`, `remove`, and `contains`:

- A **lock-free linked list**
- A **lock-based unrolled list** (still a linked structure, but each node holds a contiguous array of elements instead of a single value)

If lock-free avoids locking overhead entirely, how does a structure that *still uses locks* end up faster by a wide margin? The answer has almost nothing to do with locking at all.

### How your CPU actually fetches memory

The lock-free linked list suffers from pointer chasing. This is a classic problem with linked structures, where finding a value means hopping from node to node, each one potentially scattered somewhere different in memory.

The unrolled list sidesteps this. Because it stores elements in contiguous arrays, it leans on **spatial locality**; the principle that if a CPU core accesses one memory location, it's very likely to access nearby locations next. Modern memory systems fetch data in 64-byte cache lines, and arrays are laid out exactly to exploit that: when the CPU pulls one element from an array, it usually pulls the neighboring elements too, straight into L1 or L2 cache. Cache access is fast. Main memory access is not. (I go deeper on this in my Cache Coherence article)

This is the idea Martin Thompson calls **mechanical sympathy**; designing software that works with the hardware's grain instead of against it.

Back to the benchmark, just to make things concrete

## Benchmark setup

- Mode: Throughput (ops/s)
- Warmup: 10 iterations × 1s
- Measurement: 10 iterations × 1s
- Forks: 3
- Threads: 8
- CPU: Intel(R) Core(TM) i5-10300H @ 2.50GHz, 4 cores, 8 processors

Keyspace: 1_000_000 random integers. Two workloads:
- **Read-heavy**: 90% contains, 9% adds, 1% removes
- **Write-heavy**: 50% adds, 40% removes, 10% contains

## What do the numbers say?

### Write Heavy
![Write Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=1729214476&format=image)

### Read Heavy
![Read Heavy Chart](https://docs.google.com/spreadsheets/d/e/2PACX-1vTvl_TCJug-pfYqm8BALLomB7GZWus5U3q9pGHrqS5V7AXudMLME796garFMhenHvp_iaVYwUtXuyWH/pubchart?oid=2083678510&format=image)

The unrolled list beat the lock-free list by roughly **50x** on the write-heavy workload and **80x** on the read-heavy one. Despite the lock-free list offering non-blocking guarantees on every path, and the unrolled list relying on plain fine-grained locking.

This shows that progress guarantees don't automatically guarantee performace. However, designing with respect to modern hardware does.

## Conclusions

"Lock-free" and "fast" aren't the same thing. A structure can give you perfect non-blocking guarantees and still lose badly to a lock-based structure that simply respects how the CPU fetches memory. If you're chasing performance, designing around modern hardware will often get you further than designing around avoiding locks.

Source code for both structures is on my GitHub: https://github.com/kusoroadeolu/concurrent-lists