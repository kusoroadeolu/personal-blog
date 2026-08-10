# Extending Causal Chains with Release/Acquire
There's been a question that's been on my mind for a while now, but let me rewind a bit. Around 5 months ago, I built my first lock free datastructure a `Treiber Stack`; building one is basically a textbook intro to lock free concurrency. 

```java
//A simple treiber stack
public class TreiberStack<T> {
    private final AtomicReference<Node<T>> head = new AtomicReference<>();

    public boolean push(T t) {
        var node = new Node<>(t);
        Node<T> h;
        do {
            h = head.getAcquire();
            node.next = h;
        } while (!head.compareAndSet(h, node));
        return true;
    }

    public T pop() {
        Node<T> h;
        Node<T> next;
        do {
            h = head.getAcquire();
            if (h == null) return null;
            next = h.next;
        } while (!head.compareAndSet(h, next));
        return h.item;
    }

    public String toString() {
        return String.valueOf(head.get());
    }

    static class Node<T> {
        final T item;
        Node<T> next;

        Node(T item) {
            this.item = item;
        }
    }
}
```

## A quick detour
I want to talk a bit about causality before I get to the question


### What is a causal relationship
A causal relationship means that one event (the cause) makes another effect happen (the effect). 
For example, turning on a light switch turns on a light bulb. 

In this example, turning on the light switch is the **cause** while turning on the light bulb is the **effect**. 

Causal relationships can also be extended to form causal chains; a scenario where an event A leads to an event B which leads to an event C `(A -> B -> C)` 

For example turning on a light switch turns on a light bulb which in illuminates the room. The effect extends the previous causal relationship to produce a new effect and also create a causal chain

### A bit more on causality
Causality is the underlying principle of causal relationships. It is a an ordering between two events in which one event precedes another event, if for every observer it will always appear that event preceded the other

However, some pairs of events cannot be compared with each other, hence have no  causality between themselves (we cannot determine which event occurred before the other) 

### Why should we care?
Causal relations also extend to in-memory concurrency. In Java, from the VarHandle JMM cookbook, the release/acquire accesses and stronger accesses provide causal consistency memory guarantees plus the other guarantees provided by weaker memory accesses. These accesses, of course, just preserve the orderings, they have no understanding of causal relationships. 


### Happens-before
Relating to causality, we can look at **happens-before** as a principle simply defines a causal order before two events in a system; of course though this isn't a formal definition. It's more natural to talk in terms of **happens-before** relationships in Java, as that's what the JMM uses to define legal executions of shared memory accesses across threads.

## So what is the question?
The initial question I asked was: **What is the weakest memory accesses we needed to traverse a treiber stack after an acquire/volatile read of the head of the stack**

After some time, I realized the question could be generalized as is: **Can we create a causal chain using release/acquire which would allow for a thread Z to read a write from a thread X without ever directly observing the write from thread X.**

To answer this question, I decided to come up with a stress test that's pretty easy to reason about.

### The test 
```java
@JCStressTest
@Outcome(id = "2", expect = Expect.ACCEPTABLE, desc = "Causal chain preserved")
@Outcome(id = "1", expect = Expect.ACCEPTABLE, desc = "Boring...")
@Outcome(id = "0", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Hmm interesting")
@State 

/*
* Test to see if the read of dinner before the write to desert can extend the causal chain
* started by the write to dinner allowing us to read both desert and dinner with plain writes
*
* This is similar to how treiber stack push/pop implementations
* as treiber stacks threads read "head" or ready in this scenario
* before trying to cas head to a new head, and rereading head if their cas fails
* extending the causal chain when they eventually succeed
*
* */
public class ExtendingCausalityStress {

    private int dinner;
    private int dessert;
    private AtomicInteger ready;

    public ExtendingCausalityStress() {
         dinner = 0;
         dessert = 0;
        ready = new AtomicInteger(0);
    }

    /*
    * 1 - dinner 
    * 2 - dinner and dessert
    * */
    @Actor
    public void actorX() {
        dinner = 1;
        ready.setRelease(dinner); 
    }

    @Actor
    public void actorY() {
        while (ready.getAcquire() == 0) Thread.onSpinWait(); //creates a causal relationship
        dessert = 2;
        ready.setRelease(dessert);
    }

    @Actor
    public void actorZ(I_Result i) {
        int r = ready.getAcquire(); //creates and observes a causal chain (X -> Y -> Z), we should see the write to both dinner and dessert, more importantly though, we should see the write to dinner (simply put transitivity)
        if (r == 2) { 
            if (dinner == 1 && dessert == 2) i.r1 = 2;
            else i.r1 = 0; //causal chain wasn't created

        } else i.r1 = 1;
    }

}
```
This tests succeeds on both x86_64 and ARM systems. Actor Y extends this causal chain and allows actor Z to transitively observe the write from Actor X. The extension of this chain is caused by a combination of things:

1. Program order establishes **happens-before** between actions within a single thread. So the plain write in actor X happens-before its later release access. When another thread performs an acquire that *synchronizes-with* that release, this extends into a cross-thread **happens-before** edge, making the write visible to the acquiring thread.

2. The acquire from actor Y *synchronizes-with* actor X's release to create a **happens-before** relationship between both accesses, so (X happens-before Y) 

3. The release access in actor Y ensures that all writes that precede the release in global order are propagated to other threads who acquire on that release. 

4. If the acquire from actor Z *synchronizes-with* actor Y's release, it will create a **happens-before** relationship between both accesses, so (Y happens-before Z). Since **happens-before** relationships are transitive, X **happens-before** Y and Y **happens-before** Z, therefore X **happens-before** Z, so Z will be able to observe the write to `dinner` from actor X with a plain memory access.

 The boring outcomes, in this stress test, Z can *synchronize-with* X's release before it ever has a chance to *synchronize-with* Y's release **OR** Z could neither *synchronize-with* X or Y


This kind of thinking also extends to the treiber stack's push/pop methods **OR** structures where multiple threads create a *synchronizes-with* relation with the last release/volatile write on a shared object from a thread which performed the same action leading to a transitive chain of **happens-before** relationships. This allows a later thread who *synchronizes-with* a release/volatile write on that shared object to see all writes which **happened-before** the release/volatile write that the thread observed, using only plain read accesses (similar to stress test I wrote)

`A --release/acquire--> B --release/acquire--> C -- acquire`

which gives us

`A -------------happens-before---------------> C`

even though C never directly *synchronizes-with* A

## Thoughts?


Yeah this was just a mini experiment I decided to write up as it isn't quite intuitive at first. 
