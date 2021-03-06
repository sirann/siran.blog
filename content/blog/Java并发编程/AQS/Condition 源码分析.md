---
title: "Condition 源码分析"
date: 2020-03-01T19:12:42+08:00
draft: false
banner: "/img/blog/banners/6ce41a46gy1g31wxsf5ibj20rs0ku10r.jpg"
author: "Siran"
summary: "Condition可以替代传统的Object中的wait()、notify()和notifyAll()方法来实现线程间的通信，使线程间协作更加安全和高效。"
tags: ["AQS"]
categories: ["并发编程"]
keywords: ["AQS","Jdk源码","基础"]
---

### 简述
**Condition可以替代传统的Object中的wait()、notify()和notifyAll()方法来实现线程间的通信，使线程间协作更加安全和高效**
```c
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```
**常用的方法是await()、signal()和signalAll()，Condition与Object类中的方法对应wait()、notify() 和 notifyAll()
简单来说，Condition需要和Lock一起使用，在不使用Lock时，使用关键字synchronized时的代码如下：**
```c
synchronized(obj){ 
    obj.wait();
}
synchronized(obj){ 
    obj.notify();
}
```
**使用Lock：**
```c
lock.lock(); 
condition.await(); 
lock.unlock();
lock.lock(); 
condition.signal(); 
lock.unlock();
```
**从代码上可以看出，使用synchronized关键字时，所有没有获取锁的线程都会等待，这时相当于只有1个等待队列；而在实际应用中可能有时需要多个等待队列，比如ReentrantReadWriteLock中的ReadLock和WriteLock。
Lock中的等待队列和Condition中的等待队列是分开的，例如在独占模式下，Lock的独占保证了在同一时刻只会有一个线程访问临界区，也就是lock()方法返回后，Condition中的等待队列保存着被阻塞的线程，也就是调用await()方法后阻塞的线程。
所以使用lock比使用synchronized关键字更加灵活。**

****
### Condition的使用
**在Condition接口的javadoc中的例子，代码如下：**
```c
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notFull  = lock.newCondition(); 
    final Condition notEmpty = lock.newCondition(); 
     
    final Object[] items = new Object[100];
    int putptr, takeptr, count;
     
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
     
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```
**代码很简单，定义了一个数组items，put用于向items中添加数据，take用于从items中取出数据，count代表当前items中存放了多少个对象，putptr表示下一个需要添加的索引，takeptr表示下一个需要取出的索引，这样就实现了数组的循环添加和取出数据的功能。put和take的具体功能如下：**

* `put`
     * 当count与items的长度相同时，表示数组已满，则调用notFull.await()来等待同时释放了当前线程的锁；
     * 当线程被唤醒时，将x添加到putptr索引的位置；
     * 如果当前putptr的位置是最后一个，则下一个索引的位置从0开始；
     * 调用notEmpty.signal();通知其他线程可以从数组中取出数据了。
* `take`
     * 当count为0时，表示数组是空的，则调用notEmpty.await()来等待同时释放了当前线程的锁；
     * 当线程被唤醒时，将x添加到takeptr索引的位置；
     * 如果当前takeptr的位置是最后一个，则下一个索引的位置从0开始；
     * 调用notFull.signal();通知其他线程可以向数组中添加数据了。
**** 
### 源码分析
**Condition必须被绑定到一个独占锁上使用，在ReentrantLock中，有一个newCondition方法，该方法调用了Sync中的newCondition方法，看下Sync中newCondition的实现：**
```c
final ConditionObject newCondition() {
    return new ConditionObject();
}
```
**ConditionObject是在AQS中定义的，它实现了Condition接口，自然也就实现了上述的Condition接口中的方法。该类有两个重要的变量：**
```c
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```
**这里的firstWaiter和lastWaiter是不是和之前说过的head和tail有些类似，而且都是Node类型的。
对于Condition来说，它是不与独占模式或共享模式使用相同的队列的，它有自己的队列，所以这两个变量表示了队列的头节点和尾节点。**
****
#### await 方法
```c
public final void await() throws InterruptedException {
            //<1> 判断线程是否已经中断并且归位，如果线程已经中断了那就抛出中断异常
            if (Thread.interrupted())
                throw new InterruptedException();
            //<2> 根据当前线程创建一个Node添加到Condition队列中，等待唤醒
            Node node = addConditionWaiter();
            //<3> 释放当前线程lock， 从AQS的队列中移出
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //<4> 循环判断当前线程的Node是否在Sync队列中，如果不在，则park
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                //<5> checkInterruptWhileWaiting方法根据中断发生的时机返回后续需要处理这次中断的方式，如果发生中断，退出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //<5> acquireQueued获取锁并返回线程是否中断
            //    如果线程被中断，并且中断的方式不是抛出异常，则设置中断后续的处理方式设置为REINTERRUPT
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            //<6> 从头到尾遍历Condition队列，移除被cancel的节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            //<7> 如果线程已经被中断，则根据之前获取的interruptMode的值来判断是继续中断还是抛出异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
>await方法首先根据当前线程创建了一个Node，然后释放当前线程的独占锁。这里的savedState表示当前线程已经加锁的次数（ReentrantLock为重入锁）。while循环其实就是一直判断，当前的线程是否又被添加到了Sync队列中，如果已经在Sync队列中，则退出循环。
>
>什么时候会把当前线程又加入到Sync队列中呢？当然是调用signal方法的时候，因为这里需要唤醒之前调用await方法的线程。
>
>这里还需要注意的是，如果在park的状态下，这时线程中断了，park方法会返回，然后判断后续对此次中断的处理方式：抛出InterruptedException或者继续中断。

****
#### addConditionWaiter 方法
**创建waiter节点到队列中**
```c
private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            //<1> 如果最后一个节点被取消了，那么从头到尾遍历Condition队列，移除被cancel的节点
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //<2> 创建节点加入队列
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```
****
#### fullyRelease 方法
```c
final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            //<1> 获取ReentrantLock中的state的值 
            int savedState = getState();
            //<2> 调用ReentrantLock中的release方法
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```
**因为是可重入的原因，只有在state为0的时候才会真的释放锁，所以在fullyRelease方法中，需要将之前加入的锁的次数全部释放，目的是将该线程从Sync队列中移出。**
****
#### isOnSyncQueue 方法
**该方法判断当前线程的node是否在Sync队列中。**
```c
final boolean isOnSyncQueue(Node node) {
        //<1> 如果当前线程node的状态是CONDITION或者node.prev为null时说明已经在Condition队列中了，所以返回false；
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //<2> 如果node.next不为null，说明在Sync队列中，返回true；
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        //<3> 如果两个if都未返回时，可以断定node的prev一定不为null，next一定为null，这个时候可以认为node正处于放入Sync队列的执行CAS操作执行过程中。
        //而这个CAS操作有可能失败，所以通过findNodeFromTail再尝试一次判断。
        return findNodeFromTail(node);
    }
```
****
#### findNodeFromTail 方法
```c
private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```
**该方法就是从Sync队列尾部开始判断，因为在isOnSyncQueue方法调用该方法时，node.prev一定不为null。但这时的node可能还没有完全添加到Sync队列中（因为node.next是null），这时可能是在自旋中。记得之前说过的enq方法吗，signal的时候会调用这个方法：**
```c
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // 执行findNodeFromTail方法时可能一直在此自旋
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
**所以，这时如果CAS还未成功，那只好返回false了**
****
#### checkInterruptWhileWaiting 方法
**如果当前线程被中断，则调用transferAfterCancelledWait方法判断后续的处理应该是抛出InterruptedException还是重新中断。**
```c
private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```
****
#### transferAfterCancelledWait 方法
**该方法是判断，在线程中断的时候，是否这时有signal方法的调用。**
```c
final boolean transferAfterCancelledWait(Node node) {
        //<1> 如果执行成功，则说明中断发生时，没有signal的调用，因为signal方法会将状态设置为0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            //<2> 如果第<1>步执行成功，则将node添加到Sync队列中，并返回true，表示中断在signal之前
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        //<3> 如果第<1>步失败，则检查当前线程的node是否已经在Sync队列中了，如果不在Sync队列中，则让步给其他线程执行，直到当前的node已经被signal方法添加到Sync队列中
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```
>如果第一次CAS失败了，则不能判断当前线程是先进行了中断还是先进行了signal方法的调用，可能是先执行了signal然后中断，也可能是先执行了中断，后执行了signal，当然，这两个操作肯定是发生在CAS之前。这时需要做的就是等待当前线程的node被添加到Sync队列后，也就是enq方法返回后，返回false告诉checkInterruptWhileWaiting方法返回REINTERRUPT，后续进行重新中断。
>
>简单来说，该方法的返回值代表当前线程是否在park的时候被中断唤醒，如果为true表示中断在signal调用之前，signal还未执行，否则表示signal已经执行过了。
>
>根据await的语义，在await时遇到中断要抛出InterruptedException，返回true则使上层方法checkInterruptWhileWaiting返回THROW_IE，否则返回REINTERRUPT。
****
#### unlinkCancelledWaiters 方法
**该方法就是从头到尾遍历Condition队列，移除状态为非CONDITION的节点。因为在执行该方法时已经获取了独占锁，所以不需考虑多线程问题。**
```c
private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```
****
#### reportInterruptAfterWait 方法
**如果当前线程被中断，则在await方法中调用次方法该方法根据interruptMode来确定是应该抛出InterruptedException还是继续中断。**
```c
private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```
****
#### awaitNanos 方法
**与#await方法类似，提供超时**
```c
public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }
```
****
#### signal 方法
**在该方法中判断当前线程是否占有独占锁，然后通过firstWaiter依次唤醒Condition队列中的node，并把node添加到Sync队列中。**
```c
public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```
**在await方法中可以知道添加到Condition队列中的node每次都是添加到队列的尾部，在signal方法中是从头开始唤醒的，所以Condition是公平的，signal是按顺序来进行唤醒的。**
****
#### doSignal 方法
```c
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
    }
```
**doSignal方法先将队列前面节点依次从队列中取出，然后调用transferForSignal方法去唤醒节点，这个方法有可能失败，因为等待线程可能已经到时或者被中断，因此while循环这个操作直到成功唤醒或队列为空。**
****
#### transferForSignal 方法
```c
final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        //<1> 首先尝试设置node的状态为0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            //<2> 如果设置失败，说明已经被取消，没必要再进入Sync队列了，doSignal中的循环会找到一个node再次执行
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        //<3> 如果设置成功，但之后又被取消了呢？虽然会进入到Sync队列，但在获取锁的时候会调用shouldParkAfterFailedAcquire方法，该方法中会移除此节点。
        Node p = enq(node);
        int ws = p.waitStatus;
        //<4> 如果执行成功，则将node加入到Sync队列中，enq会返回node的前继节点p。这里的if判断只有在p节点是取消状态或者设置p节点的状态为SIGNAL失败的时候才会执行unpark。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
**什么时候compareAndSetWaitStatus(p, ws, Node.SIGNAL)会执行失败呢？如果p节点的线程在这时执行了unlock方法，就会调用unparkSuccessor方法，unparkSuccessor方法可能就将p的状态改为了0，那么执行就会失败。
到这里，signal方法已经完成了所有的工作，唤醒的线程已经成功加入Sync队列并已经参与锁的竞争了，返回true。**
****
#### signalAll 方法
>判断当前线程是否占有独占锁，然后执行doSignalAll方法。
```c
public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
```
****
#### doSignalAll 方法
**可以与doSignal方法比较一下，doSignal方法只是唤醒了一个node并加入到Sync队列中，而doSignalAll方法唤醒了所有的Condition节点，并加入到Sync队列中。**
```c
private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```
****
### 总结
1. **ConditionObject是AQS的内部类，它实现了Condition接口，提供了类似wait、notify和notifyAll类似的功能。**
2. **Condition必须与一个独占锁绑定使用，在await或signal之前必须现持有独占锁。Condition队列是一个单向链表，他是公平的，按照先进先出的顺序从队列中被唤醒并添加到Sync队列中，这时便恢复了参与竞争锁的资格。**
3. **Condition队列与Sync队列是不同的，Condition队列是单向的，队列的第一个节点firstWaiter中是可以绑定线程的；而Sync队列是双向的，队列的第一个节点head是不与线程进行绑定的。**
4. **Condition在设计时就充分考虑了Object中的监视器方法的缺陷，设计为一个lock可以对应多个Condition，从而可以使线程分散到多个等待队列中，使得应用更为灵活，并且在实现上使用了FIFO队列来保存等待线程，确保了可以做到使用signal按FIFO方式唤醒等待线程，避免每次唤醒所有线程导致数据竞争。**
5. **但这样也会导致Condition在使用上要比Object中的提供的监视器方法更为复杂，这时考虑使用Condition的数量、何时使用Condition以及使用哪个condition等等。由于Condition是结合Lock一起使用的，所以是否使用Condition需要和Lock一起进行综合的考虑。**