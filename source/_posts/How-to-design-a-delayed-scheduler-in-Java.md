---
title: How to design a delayed scheduler in Java?
date: 2021-03-13 23:13:28
tags:
    - "Java"
    - "concurrency"
categories:
    - "technique"
    - "programming"
    - "design"
---

Designing a delayed scheduler could be very interesting but challenging. Though in Java there are many built-in tools under the concurrency section, studying them on the code level is a really good practice.

The question here is to design a delayed scheduler that can accept a `Runnable` task with a delay. The task should be executed after the delay time.

At this moment, let's forget a large scale scheduler on a distributed system. The scope right now is to implement such a class and an API:

```Java
public class DelayedScheduler {
    // unit of delayTime is milliseconds
    public Future scheduler(Runnable task, long delayTime);
}
```

### Approach 1: `PriorityQueue` + polling

A simple and intuitive solution is to maintain a priority queue which contains all tasks to be executed. The most urgent task will be at the peek of this queue. We then will have a thread which iteratively and periodically checking the peek task in the priority queue; then pop and run the task if needed. This is called ***polling***.

A big issue with this approach is, how can we decide the period of polling?
- If the interval is too small, it will be a huge workload for the CPU;
- If the interval is too large, the real execution time of a task might not be accurate (it will be delayed).

### Approach 2: `PriorityQueue` + timer

Instead of polling periodically, what we can do is: get the delay time of the current peek task, set a timer with the time, check again when the timer is up.

However, **this is even worse compared with the approach 1**. Why? Consider such a situation: now the peek element in the queue will be executed one hour later, so the timer will be up after 1 hour. However, after that a new task comes into the queue, and this task's delay time is 30 minutes. Apparently, we are not able to execute this task after 30 minutes later as the thread blocks due to the timer.

### Approach 3: Let's look into `DelayQueue` now!

In Java we have a built-in queue called `DelayQueue`. This is exactly what we want! We will look into how it works by analyzing the source code, but firstly, let's implement the delay scheduler with this class.

Let's write a wrapper for the `Runnable` task.

```Java
class DelayTask extends FutureTask implements Delayed {

    private final long startTime;
    private final Runnable task;

    public DelayTask(Runnable task, long delayTime) {
        super(task, null);  // null is the return value for the runnable tasks
        this.task = task;
        this.startTime = System.currentTimeMillis() + delayTime;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long diff = this.startTime - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.getDelay(), ((DelayTask) o).getDelay());
    }
}
```

Note that we need to implement `getDelay` and `compareTo` functions, as `Delayed` is in fact only an interface, which also implements `Comparable` interface.

Now, let's implement the scheduler class.

```Java
public class DelayedScheduler {
    private final DelayQueue<DelayTask> queue;

    public DelayedScheduler() {
        this.queue = new DelayQueue<>();
        this.startExecute();
    }

    public Future schedule(Runnable task, long delayTime) {
        DelayTask newTask = new DelayTask(task, delayTime);
        this.queue.offer(newTask);
        return newTask;
    }

    private void startExecute() {
        Runnable execute = () -> {
            while (true) {
                try {
                    DelayTask task = this.queue.take();
                    task.run();
                } catch (InterruptedException e) {
                    e.printStackTree();
                }
            }
        };
        new Thread(execute, "Executor").start();
    }
}
```

The code is really concise and self-explanatory, thanks to the good capsulation of this class. The source code of `DelayQueue` is not complicated but very delicate!

Below is the implementation of some core functions of this class.

```Java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E> implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader = null;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();

    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
}
```

As you can see, we still use a priority queue to manage all tasks and the peek is the one with the shortest delay time. Instead of using polling or timer, we use the wait/notify(signal) to handle the concurrency problems. If there are many threading that blocks on `take()` functions (i.e., they are all waiting to get an element from the queue), we use a ***leader/follower*** pattern to efficiently minimize unnecessary timed waiting. The leader will only wait for the delay time of the peek element in the queue, while other threads need to wait until signaled.


Here are many design tricks that are deserved to be noticed.

> Q: In `offer()` function, why we need to check the peek with the element we just added?

```Java
q.offer(e);
if (q.peek() == e) {
    leader = null;
    available.signal();
}
```

Once we've added the element, if the peek element becomes the one we just added, it means the element will be earliest one to come out of the queue (this is exactly the issue that approach 2 cannot handle!). If we ignore it, no thread will be aware of it and the leader will still wait for the previous peek element!

To figure out how exactly it works, let's go through one example. Suppose now the peek element's name is A, and its delay is 1 hour. In addition, we have one or more thread blocking on `take()` because element A cannot be out until 1 hour later. The leader thread, say thread-1, blocks on `awaitNanos()` (or say, it is waiting for A to become available) and other threads (if any) are waiting until signal as either of them is not the leader:

```Java
if (leader != null)
    available.await();
```

Now, a new element B comes in, and its delay time is only 10 minutes. According to the code, it will set `leader` to `null` and notify one of the waiting thread.

1. If there is only one waiting thread, which is thread-1, then it will stop waiting (line 65), enter the while loop again, get the peek element again (**now the peek element will be B**), and finally starts waiting for B;
2. If there are more than one thread, which means we have followers, then the thread signaled is not necessarily thread-1!
    - if thread-1 get signaled, then it is same with 1;
    - otherwise (**here comes the tricky part**), another thread, say thread-2, wakes up! Since `offer()` function also sets `leader` to `null`, so thread-2 will re-enter the loop in `take()` and becomes the leader, and it will also start to wait for B. As for thread-1, it will wake up once the timeout happens (1 hour) or signaled by other operations.

> Q: After one thread takes an element, why do we need to signal?

```Java
finally {
    if (leader == null && q.peek() != null)
        available.signal();
    lock.unlock();
}
```

This is in fact apparent: all follower threads are actually waiting infinitely, we must notify one of the threads if there is **no leader** but **the queue has elements inside it**. Generally, you can find that `available.signal()` is in fact to **make sure there is a new when a newer element becomes available at the head of the queue or a new thread may need to become leader** (this is the comment in the source code :P).

> Q: Why we need to do this: `first = null`?

According to the Doug Lea's comment after this line, we know this is to avoid memory leakage. If a thread (thread-1) is not the leader but it is still holding `first` in its thread, then after `first` is taken by the leader, it cannot be collected by Java GC (though it should be) as long as thread-1 keeps waiting.

> Q: When a thread start to waiting in the `take()` method, does it keep holding the lock?

No, it will not. Note, `availability` is associated with the current lock!

The doc says:


> `void await()
>      throws InterruptedException`
> Causes the current thread to wait until it is signalled or interrupted.
>
> The lock associated with this Condition is atomically released and the current thread becomes disabled for thread scheduling purposes and lies dormant until one of four things happens:
> * Some other thread invokes the signal() method for this Condition and the current thread happens to be chosen as the thread to be awakened; or
> * Some other thread invokes the signalAll() method for this Condition; or
> * Some other thread interrupts the current thread, and interruption of thread suspension is supported; or
> * A "spurious wakeup" occurs.
>
> In all cases, before this method can return the current thread must re-acquire the lock associated with this condition. When the thread returns it is guaranteed to hold this lock.

### Approach 4: implement a delay queue by ourselves!

Okay, as we get clear with the mechanism of this class, we can totally implement the scheduler without `DelayQueue`. Some part of the codes will remain the same.

```Java
public class DelayedScheduler {
    private final PriorityBlockingQueue<DelayTask> queue;
    private final Condition available;
    private final ReentrantLock lock;
    private Thread leader;

    public DelayedScheduler() {
        this.queue = new PriorityBlockingQueue();
        this.lock = new ReentrantLock();
        this.available = this.lock.newCondition();
        this.leader = null;
        this.startExecute();
    }

    public Future schedule(Runnable task, long delayTime) {
        DelayTask newTask = new DelayTask(task, delayTime);
        this.offer(newTask);
        return newTask;
    }

    private void startExecute() {
        Runnable execute = () -> {
            while (true) {
                try {
                    DelayTask task = this.take();
                    task.run();
                } catch (InterruptedException e) {
                    e.printStackTree();
                }
            }
        };
        new Thread(execute, "Executor").start();
    }

    private boolean offer(DelayTask task) {
        this.lock.lock();
        try {
            this.queue.offer(task);
            if (this.queue.peek() == task) {
                this.leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    private DelayTask take() throws InterruptedException {
        this.lock.lock();
        try {
            while (true) {
                DelayTask peek = this.queue.peek();
                if (peek == null) {
                    // no element at all, just wait!
                    available.await();
                } else {
                    long delay = peek.getDelay(TimeUnit.NANOSECONDS);
                    if (delay <= 0) return this.queue.poll();
                    if (leader != null) {
                        // not the leader, need to waits
                        this.available.await();
                    } else {
                        Thread curThread = Thread.currentThread();
                        this.leader = curThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (curThread == this.leader) this.leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && this.queue.size() > 0) {
                available.signal();
            }
            this.lock.unlock();
        }
    }
}
```

However, this is still kind of overkilled. Why? Because we will not have multiple thread calling `take()` at the same time! Thus, we might not need to use the leader/follower pattern here, and we can simplify the code further.

```Java
public class DelayedSchedulerSelfDev {

    private final PriorityBlockingQueue<DelayedTask> queue;
    private final ReentrantLock lock;
    private final Condition available;

    // ...

    private boolean put(DelayedTask task) {
        this.lock.lock();
        try {
            this.queue.offer(task);
            if (task == this.queue.peek()) {
                this.available.signal();   // wake up one thread (not all!)
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    private DelayedTask take() throws InterruptedException {
        this.lock.lock();
        try {
            while (true) {
                DelayedTask peekTask = this.queue.peek();
                if (peekTask == null) {
                    // no elemnets; wait!
                    available.await();
                } else {
                    long delay = peekTask.getDelay(TimeUnit.NANOSECONDS);
                    if (delay <= 0) return this.queue.poll();
                    available.awaitNanos(delay);
                }
            }
        } finally {
            lock.unlock();
        }
    }

}
```

As you can see, you don't need to maintain a `leader` thread now. The logic for `take()` is also much simpler: the thread either waits forever if there is no element (until signaled) or waits until the first element becomes available.

### Distributed Scheduler

Let's talk about it in another post if we have time!
