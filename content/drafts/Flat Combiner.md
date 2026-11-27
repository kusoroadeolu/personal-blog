# Flat Combining

The idea of flat combining is to allow threads publish their operation to modify a critical section as a request. One thread becomes the combiner and applies requests on behalf of the others. 

The idea of a single thread applying operations for is quite similar to a concurrency pattern called  **delegation**

### Delegation
For delegation, instead of protecting some critical section with some mutex or lock, we funnel a critical section to one `owning` thread. This thread can be chosen at construction or assigned dynamically at runtime. 

Threads who want their operation applied to that critical section submit their requests through a message passing queue, which the delegated thread repeatedly polls from.

However, since the delegated thread can't obviously be allowed to spin forever, especially in userspace as it wastes CPU, the delegated thread would need to park at some point. 
This does introduce the issue of a scheduler dependency, meaning there is a window between which the delegated thread being scheduled onto the cpu and it running requests for other threads. 

This issue is worsened if a waiting thread needs to wait synchronously for its request to be applied.


## Why Flat Combining?
TODO Write something here

---

A basic implementation of **flat combining** might look something like this.

```java
final Queue<Request> waitQueue = new ConcurrentLinkedQueue<>();
static final int FREE = 0;
static final int HELD = 1;
static final int MAX_SERVES = 16;
final AtomicInteger state = new AtomicInteger(FREE);

static class Request {
  final AtomicReference<Runnable> work;
  Thread waiter;

  Request(Runnable work, Thread thread) {
      this.waiter = thread;
      this.work = new AtomicReference<>(work);
  }
}

public void combine(Runnable work) {
  Request r = new Request(work, Thread.currentThread());
  waitQueue.add(r);

  while(r.work.getAcquire() != null) {

      if (state.compareAndSet(FREE, HELD)) {
        
        if(r.work.getPlain() != null) {
          r.work.setPlain(null);
          work.run();
        }
        
        serveWaitingThreads();
        return;
      }

      spinThenPark();
  }
}

public void serveWaitingThreads() {
    Request req;
    List<Thread> waiters = new ArrayList<>();
    int serves = 0;
    while(serves < MAX_SERVES && (req = waitQueue.poll()) != null) {
        Runnable work = req.work.getPlain();
        if (work != null) {
          work.run();
          req.work.setRelease(null);
          waiters.add(req.waiter);
          ++serves;
        }
    }

    state.set(FREE);
    wakeNextThread();
    wakeServedThreads(waiters);
}

// Note that the park/unpark protocol is deliberately left abstract.

```

Flat combining doesn't get rid of that scheduler dependency though. This is mainly cause combiner, after serving waiters, might need to wake a successor to pick up from where it left, which still leaves that window between getting scheduled and running requests for other threads.

This scenario mainly occurs when the wait queue has more waiters than the number of waiters the combiner can serve at once. So, how often it happens depends on the size of the critical section the combiner is protecting and the number of waiters the combiner can serve in one pass.


### An extra benefit!
Flat combining does have a nice extra quirk that alleviates certain memory bottlenecks. 

> The presence of a critical section usually indicates the presence of an in memory shared object which is modified in that critical section

With conventional locking, claiming ownership of some shared object in a critical section by threads on different cores can cause memory traffic due to the coherence protocol. This memory traffic is pretty expensive as it requires cross-core communication. 

Flat combining can improve locality here since a single combiner can execute multiple operations against the shared object in one pass rather than repeatedly transferring ownership to other threads.

