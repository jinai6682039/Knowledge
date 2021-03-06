# Deep into ReentrantLock

Java提供了ReentrantLock来作为可扩展的synchronized的同步锁，提供了与synchronized相同的语义，但又提供了时间锁等候，可中断锁等候等功能 。
ReentrantLock实现了Lock接口，其中的**全局抽象类Sync**是**ReentrantLock实现的关键**。**Sync**为**锁功能的实现**提供了一些列方法。而ReentrantLock也使用**Sync**类来实现了两种锁：**NonfairSync**（非公平锁）与**FairSync**（公平锁）两种。ReentrantLock默认使用的是NonfairLock。

全局类Sync提供了一个**同步互斥量state**以及一个**等待线程链表的表头head**。
state：为0时，代表还没有线程获取了锁；为>= 1时，代表有某个线程获取了锁，并且可能重入state - 1（state > 1 )次

## ReentrantLock的使用
```java
    ReentrantLock takeLock = new ReentrantLock();    
    // 获取锁  
    takeLock.lock();  
    try {  
      // 业务逻辑  
    } finally {  
      // 释放锁  
      takeLock.unlock();  
    }  
```
一般ReentrantLock的使用可以参照上面的实例。为什么需要在finally中来释放ReentrantLock？这样做是为了在业务逻辑处理时抛出了异常，导致没有执行锁的释放。

## ReentrantLock的实现
在ReentrantLock.lock()中会调用sync.lock()
```java
public void lock() {
        sync.lock();
    }
```
这里的sync可能时NonfairSync或者FairSync，这取决于ReentrantLock的构造函数。
对于NonfairSync和FairSync来说，其lock()方法是不同的。
```java
		// NonfairSync.lock()
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

		// FairSync.lock()
        final void lock() {
            acquire(1);
        }
```

对于NonfairSync来说，其可以看出为什么叫做非公平锁。首先其在某个线程调用lock()方法的时候，会先调用CAS操作去尝试去修改Sync中的同步信号量state的值。若修改成功，则说明此线程已经拿到了锁，这就对已经在Sync中的等待线程链表中的线程不公平。若CAS修改失败，此时会和FairSync处理一样，调用Sync父类的acquire(1)方法来尝试获取锁。
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这里调用的tryAcquier()方法在Sync父类中是一个空实现，需要在NonfairSync和FairSync中来实现。
```java
		// NonfairSync
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        // FairSync
                protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
可以看到NonfairSync与FairSync对tryAcquire()有不同的实现。NonfairSync中是调用了Sycn.nonfairTryAcquire()方法。这里传递进来的参数int acquire都为1。

对于NonfairSync.tryAcquire()，调用线程会先获取当前Sync的 同步信号量state，当state为0时，再次越过Sync的等待队列，使用CAS操作尝试修改state值来获取锁，若修改成功，则当前线程获取了锁。当state不为0，则说明已经有线程获取了锁，此时判断Sycn中获取锁的线程是否为当前线程，若是，则相当于锁重入，state+1。若不是，则会跳出tryAcquire()方法，接下来将当前线程加入到Sync的等待队列中并阻塞。

对于FairSync.tryAcquire()，调用线程也会先获取Sync的同步信号量state。当state为1时，处理与NonfairSync一致。但当state为0时，会先调用Sync父类的hasQueuedPredecessors()方法来判断Sync的等待队列中是否还有前序线程，若没有则尝试使用CAS操作去修改state。若存在，则将线程加入到Sync的等待队列中，并阻塞。

## tryLock() & tryLock(long, TimeUnit)
tryLock() 与 tryLock(long, TimeUnit)提供了可以非阻塞的锁的获取方式。这两个方法都会尝试使用NonfairSync的tryAcquire()方法去获取锁。当获取不成功，前者则会立即返回false，而后者在参数指定的时间内若一直没有获取到，就会返回false。
这就比synchronized提供了非阻塞的锁的获取方式以及时间等待锁。

## lockInterruptibly()
lockInterruptibly()提供了可中断的锁获取方式。
当两个线程同时使用lockInterruptibly()方法去获取锁，此时若线程A已经获取锁了，那么线程b将继续等待。此时若调用了B.interrupt()，就能中断B线程的等待，让线程b去执行其他工作。