# C4 
A term used to coin four C's when it comes to memory ordering in concurrent based systems. 
1. Causality - a principle that one event, process or state(the cause) directly influences and produces another (the effect)
2. Coherence - a principle that implies given multiple actors writing/reading to shared state, they always agree on a shared order of writes
3. Commutativity - Denotes that the result of an operation always remains the same regardless of what order it is performed in. Reminds me of _Sequential Consistency_ of multi core systems
4. Consensus - a principle where multiple nodes agree on a single data value/state, even if some nodes fail


# Java memory modes cheat sheet
## Opaque.
### Guarantees
Per variable access with opaque mode are guaranteed certain traits. Given a variable C

1. Threads interacting through a specific variable(marked as opaque) cannot form cyclic accesses with themselves. i.e. a thread appears to read a value written in the future. A happens before B happens before A
---
>  Thread B reads C = 2 (reads a value from the future) -> Thread A writes C = 2 (Hasn't happened yet but B has read it)
---

2. Coherence: Threads must agree on a given linear ordering of accesses to that variable, basically you can't have a situation where it is ambiguous which write came first. This builds upon the previous point of acyclicity. Basically communication between threads using opaque mode access cannot form a cyclic dependence since a linear ordering is always agreed upon.

    This is upheld by two invariants:
    - Variable overwrite order is consistent with the read-by relation: In the linear ordering, a read of a write must come before an overwrite of that write 
    - Variable overwrite order is consistent with the from-read relation: An overwrite of a write must appear after a read that observed that write
      i.e. Given a read **R1** observes a write **W1**, a write **W2** cannot be ordered before **W1** even though it might have occurred before **W1**.
      So even if W2 occurred before W1 in time, as long as in the agreed upon ordering, **W1** comes before **W2**

3. Bitwise Atomicity: Writes to long/double fields are written atomically. No word tearing can occur 

4. Progress: Writes to a variable are eventually visible across threads. Though no further guarantees are made for other variables

Note that these guarantees can only be upheld if both accesses are opaque(or stronger) and not necessarily a weaker mode for both reads/writes

### Caveats
1. Opaque mode does not coordinate ordering constraints with respect to other variables 
2. Coherence does not guarantee the specific ordering even though ordering is guaranteed

## Release/Acquire
### Guarantees
1. If a write A comes before interthread **Release** mode write W in source program order, then write A comes before write W in local program order
2. If interthread **Acquire** mode read R comes before read B in source program order, then read R comes before read B in local program order

____
Usages can be thought of in terms of 
1. Ownership i.e. a thread making a constructed object available/visible to other threads to use
2. Ownership transfer i.e. a thread transfers ownership of an object it will not use again to another thread

### Fences
A `VarHandle#releaseFence()` ensures all non-local writes/reads complete before the fence
A `VarHandle#acquireFence()` ensures all non-local reads complete before the fence and invalidates reads after the fence i.e. if an acquireFence separates two reads, the second read cannot reuse an old value it saw before the fence 

### Caveats
- A release write does not guarantee ordering of any writes after it


## Volatile
The default ordering mode for `Varhandle`. When all accesses in local program order use volatile memory ordering, then all accesses are sequentially consistent i.e. Local program order must respect memory ordering

| Thread A   | Thread B   |
|------------|------------|
| L1: r1 = x | S2: x = 1  |
| S1: y = 1  | L2: r2 = y |
 Across all possible sequential reorderings, x or y can be 1 but both cannot be 1


### Fences
A `VarHandle#FullFence` ensures access before and after the fences cannot be reordered with each other


Trailing fence convention ensures full fences between methods who do not know anything about each other are not redundant
```
releaseFence()    // drain earlier accesses before write
x = v            // write
fullFence()       // make write visible + serve as leading fence for next access

v = x            // read (sees latest value because of the above fullFence)
fullFence()       // serve as leading fence for whatever comes after this read
```

