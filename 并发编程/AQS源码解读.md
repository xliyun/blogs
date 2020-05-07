## 一、AbstractQueuedSynchronizer的内部类、成员变量、方法

AQS为各种锁和同步组件的实现提供了一个模板，通过成员变量private volatile int state 表示同步状态(volatile保证多线程之间的可见)，并且通过Node内部类维护一个FIFO同步队列

### 1.继承自AQS需要重写的方法



- tryAcquire(int)  试图在独占模式下获取对象状态，成功返回true，失败返回false。
- tryRelease(int)  试图设置状态来反映独占模式下的一个释放，成功返回true，失败返回false。
- tryAcquireShared(int) 试图在共享模式下获取对象状态，负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int) 试图设置状态来反映共享模式下的一个释放，如果释放后允许唤醒后续等待结点返回true，否则返回false。
- isHeldExclusively()

### 2. 内部类Node

- CANCELLED，值为1 。场景：当该线程等待超时或者被中断，需要从同步队列中取消等待，则该线程被置1，即被取消（这里该线程在取消之前是等待状态）。节点进入了取消状态则不再变化；
-  SIGNAL，值为-1。场景：后继的节点处于等待状态，当前节点的线程如果释放了同步状态或者被取消（当前节点状态置为-1），将会通知后继节点，使后继节点的线程得以运行；
-  CONDITION，值为-2。场景：节点处于等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()方法后，该节点从等待队列中转移到同步队列中，加入到对同步状态的获取中；
- PROPAGATE，值为-3。场景：表示下一次的共享状态会被无条件的传播下去；
- INITIAL，值为0，初始状态。

```java
static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
    
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

### 3. ConditionObject

implements Condition, java.io.Serializable



## 二、从ReentrantLock获取公平锁入手看AbstractQueuedSynchronizer

- ReentrantLock含有三个内部类Sync、FairSync、NonfairSync。
- ReentrantLock的构造方法会创建一个FairSync或者NonfairSync内部类，它们都继承另一个内部类Sync，而Sync继承自AQS，FairSync和NonfairSync都需要重写tryAcquire(int arg)和tryRelease(int arg)来重载AQS的代码。
- ReentrantLock的lock()和unlock()都是调用了FairSync或者NonfairSync的lock和unlock()方法。
- 而FairSync和NonfairSync则是实现公平和非公平锁的细节。


### 1.构造方法

默认是公平锁，还可以传递构造参数fair实现非公平锁，这两个都是排他锁

**下面先用非公平锁来看ReentrantLock怎么使用AQS的。**

```java
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### 2.Sync继承AQS

Sync继承自AQS

```java
abstract static class Sync extends AbstractQueuedSynchronizer{
    //省略...
}
```



### 3. Sync子类NonfairSync重载lock()

ReentrantLock初始化时，会初始化成员变量为NonfairSync(继承自Sync)或者FairSync(继承自Sync)，所以当ReentrantLock调用lock()和unlock()时，调用的是NonfairSync的lock和unclock方法。

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        
        //非公平锁的获取锁
        final void lock() {
            //第一次尝试获取AQS锁
            if (compareAndSetState(0, 1))
                //AQS锁的拥有者赋值为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //调用AQS的acqurie将当前线程加入同步队列
                acquire(1);
        }

        //非公平锁的释放锁
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

### 4.AQS的compareAndSetState方法

- AQS的compareAndSetState(0,1)就是CAS操作，是JDK中有一个类Unsafe实现，它提供了硬件级别的**原子操作**l来修改AQS的state状态。
- CAS有三个操作数：内存值V、旧的预期值0、要修改的值1，当且仅当预期值0和内存值V相同时，将内存里AQS的state修改为1并返回true，否则什么都不做并返回false。
- 这段代码可能会有多个线程调用，多个线程尝试获取锁(AQS就是那个锁，获取到锁的就把this线程传入，并修改原来的状态0为1)

```
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

compareAndSwapInt是通过反射根据字段偏移去修改对象的.
可以看到int是4个字节的偏移量,long是4个字节的偏移量,string是4个字节的偏移量
注意 Unsafe的对象不能直接new,要通过反射去获取.

```java
	//类state成员变量的偏移值
    private static final long stateOffset;

    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```



### 5.AQS的acquire方法

acquire是AQS的方法，

```java
    public final void acquire(int arg) {
        //如果获取资源失败(!tryAcquired(arg))
        // 那么就把这个线程放到等待队列里面去addWaiter(Node.EXCLUSIVE)
        //Node.EXCLUSIVE就是独占的意思，是null
        //第二次尝试获取AQS锁
        if (!tryAcquire(arg) &&
                //addWaiter将线程加入队列后，就
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

  设置线程中断不影响线程的继续执行，但是线程设置中断后，线程内调用了wait、jion、sleep方法中的一种， 立马抛出一个 InterruptedException，且中断标志被清除，重新设置为false。

```java
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```



### 6.NonfairSync重写tryAcquire(int acquires)

非公平锁重写tryAcquire调用的是Sync的

```java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

### 7.Sync的nonfairTryAcquire方法

这是第二次获取AQS锁，如果获取到锁就把当前线程赋值给AQS的exclusiveOwnerThread，如果没有获取到锁，并且是当前线程已经拿到了锁，就将状态加1，表示当前线程在AQS上加了两把锁。因为锁重入的时候只有当前线程，所以代码赋值state的时候不需要考虑多线程问题。

```java
        //非公平锁获取AQS锁
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            //获取状态，如果是0
            if (c == 0) {
                //把当前状态设置为1
                if (compareAndSetState(0, acquires)) {
                    //当前线程设置为独占的线程 就是说当前线程挣到了资源，获取到了锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果不为零，并且当前线程和锁住资源的线程是同一个线程，那么
            else if (current == getExclusiveOwnerThread()) {
                //资源被当前线程锁住多少次加1
                int nextc = c + acquires;
                if (nextc < 0) // overflow小于0就是溢出了
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);//更新我们的状态
                return true;
            }
            //获取锁失败
            return false;
        }
```

### 8.AQS的addWaiter(Node mode)方法

在上面的b标题中，如果tryAcquire(1)失败，就会去执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
                //addWaiter将线程加入队列后，就
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

将当前线程制作成头节点为null的同步队列的节点。

然后整理同步队列，如果尾节点不为空，当前线程前一个节点就是尾节点。如果尾节点为空，

```java
    private Node addWaiter(Node mode) {
        //addWaiter(EXCLUSIVE)独占专递过来的是null 就是新建一个节点 把这个Node加入到尾部去
        Node node = new Node(Thread.currentThread(), mode);
        //拿到尾节点作为当前节点的前一个
        Node pred = tail;
        //如果尾节点不为空
        if (pred != null) {
            //当前节点的前一个就是尾节点
            node.prev = pred;
            //compareAndSetTail把我们当前的节点设置为尾节点
            if (compareAndSetTail(pred, node)) {
                //让原来尾节点的下一个节点指向当前节点
                pred.next = node;
                return node;
            }
        }
        //如果尾节点为空，就说明一个节点都没有(如果有一个头结点，那么他也是尾节点)
        //enq就是新建一个头结点，然后把当前节点作为尾节点
        enq(node);
        return node;
    }
```

### 9.AQS的enq方法(不用同步synchronized写的典范)

- 新建头节点enq可能会有多个线程进来，在for循环中执行，先获取尾节点，在头节点还没有设置时，头节点和尾节点都是null。
- 多个线程中肯定会有某个线程通过compareAndSetHead(new Node())创建值为空的头节点，创建了头节点的那个线程，同时也是尾节点，所以tail=head。
- 这个时候t即指向头节点，也指向尾节点。
- 头节点创建之后，接下来这些线程会去执行else ，多个线程中肯定会有某个线程compareAndSetTail(t, node)竞争成功，将自己设置为尾节点这时候再将头节点的下一个节点指向当前线程的节点t.next = node;
- enq故意不用同步方法，反正哪个线程创建头节点都一样，然后这些线程又会在for循环中竞争创建尾节点，添加到尾节点上，保证了效率(for循环比while效率高)。

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { 
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

### 10.AQS的acquireQueued(final Node node, int arg)方法

- 进入for死循环，边界是p == head && tryAcquire(arg)，而p是node.predecessor();当前线程的前一个节点，也就是说p是同步队列中第一个节点才能去尝试获取AQS的锁。当执行完shouldParkAfterFailedAcquire(p, node)清理掉前面的失效节点后，就会将当前线程阻塞。
- 如果线程执行过阻塞，interrupted就会被赋值为true
- setHead(node)把当前节点的值和前指针赋值为null，变成没有值的head节点
- 然后让原来的头节点的后指针指向null，等待jvm GC掉

```java
    //传递进来当前线程创建的node
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {//!!!!!!!!!!!!!!线程就是阻塞在了这里
                //获取当前线程的前一个节点 ！！！！！！！！！
                Node p = node.predecessor();
                //如果前一个节点是头节点(也就是当前节点是第一个节点，AQS链表是一个空的head指向头结点)，并且当前节点tryAcquire获取源锁成功
                //第三次获取AQS的锁
                if (p == head && tryAcquire(arg)) {
                    //当前节点tryAcquire获取到锁了就把他设置为头结点,并且线程Thread设置为空，因为已经获取到线程锁了，
                    setHead(node);
                    p.next = null; // help GC 释放掉前一个头结点
                    failed = false;
                    return interrupted;
                }
                //p前一个节点
                //shouldParkAfterFailedAcquire如果前一个节点为SINGNAL等待唤醒状态
                //进行park挂起当前线程为wait状态 并且返回该线程是否被中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())//挂起后线程的代码就停在了这里，被唤醒后从这里继续循环执行，被唤醒也能再被别的线程抢了，所以for(;;)循环
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### 11.AQS的shouldParkAfterFailedAcquire(Node pred, Node node)方法

如果没有通过(p == head && tryAcquire(arg))的线程会执行此函数

如果当前节点的前一个节点状态是SINGLE(等待唤醒状态)，就返回true。

如果前一个节点状态为CANCLE状态，就从当前节点往前找把所有CANCLE状态的节点都从队列中剔除。

如果前一个节点状态是大于0的，就通过CAS将状态更改为SINGLE。

如果最终返回了 false，但是外面的acquireQueued(final Node node, int arg) 也是for循环，所以整理完了还是会再次进来，然后返回true。

```java
										          //pred前一个节点 node当前节点   
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //如果>0 就把cancle的节点取消掉 返回false 如果是0或者-3 设置为SIGNAL返回false，也就是说总有一次是SIGNAL返回true
        int ws = pred.waitStatus;//拿到前一个节点的等待状态
        if (ws == Node.SIGNAL)//-1 如果前一个节点是等待唤醒的状态
            
            return true;
        if (ws > 0) {//就是1 CANCLE状态

            do {
                //移除属于CANCLE状态的节点，因为他们不需要去竞争CPU的资源了
                node.prev = pred = pred.prev;//连等赋值从右往左执行
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else { //其他就是0或者-3 不会是-2CONDITION
            /*PROPAGATE，值为-3。场景：表示下一次的共享状态会被无条件的传播下去；
            INITIAL，值为0，初始状态。*/

            //判断前驱节点是否是SINGNAL，
            //如果前一个节点的状态不是大于0的，则通过CAS尝试将状态修改为“Node.SIGNAL”，自然的如果下一轮循环的时候会返回值应该会返回true。
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

将当前线程挂起到WATING状态，等待唤醒。也就是阻塞状态

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

```java
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
```

## 三 、ReentrantLock释放锁

### 1.ReentrantLock的unlock()方法

```java
    public void unlock() {
        //sync的release其实调用的是继承自AQS的方法
        sync.release(1);
    }
```

### 2.AQS的release(int arg)方法

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
             //上文说道acquireQueued操作完成后（拿到了锁），会将当前持有锁的节点设为头结点
            //如果头节点不为空，说明队列里面有node，没有Node的时候head和tail都是null
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```



### 3.Sync的tryRelease(int releases)方法

- 计算当前AQS锁释放后的状态。
- 如果不是AQS锁的拥有者来操作，就抛出异常。
- 如果AQS上的锁已经为0了，将AQS锁拥有者置空。
- 设置AQS锁状态。

```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

4.AQS的unparkSuccessor(Node node)方法

```java
							//传入的是head节点    
    private void unparkSuccessor(Node node) {
        //获取当前节点等待状态
        int ws = node.waitStatus;
        if (ws < 0)// 状态值小于0，为SIGNAL(-1)、CONDITION(-2)、PROPAGATE(-3)
            //比较设置当前的节点为0
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        //如果下一个节点为空(那就是空队列了)或者waitStatus>0是CNCELLED状态
        if (s == null || s.waitStatus > 0) {
            s = null; //把下一个节点置为空 ，然后从结尾向前遍历，获取到下一个不为空的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                /*可能会存在当前线程的后继节点为null，超时、被中断的情况，如果遇到这种情况了，则需要跳过该节点，但是为何是从tail尾节点开始，而不是从node.next开始呢？
                是因为在addWaiter()时，里面添加尾节点的代码
                if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
            执行完if (compareAndSetTail(pred, node)还没有执行pred.next=node;这个空档，你从前往后遍历，就会找不到这个尾节点，死循环了*/
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //方法来释放对应的被挂起的线程
            //头结点的后继结点被唤醒，那么后继被挂起的结点会从acquireQueued()唤醒，尝试获取锁，如果获取成功就将头结点设置为自身，并将头结点的前任节点清空。
            LockSupport.unpark(s.thread);
    }
```

