

## ReentrantReadWriteLock

##### 用32位int来记录读写锁状态

[源码，补码](https://blog.csdn.net/king_msky/article/details/17221973)

```java
        //写锁用低16位
 		static final int SHARED_SHIFT   = 16;
		//1左移16位
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        //无符号右移16位就是读锁的状态
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        //与操作取后16位，传递进来是啥就是啥，就是独占锁的数量
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

#### 非公平锁NofairSync(继承自Sync)

##### 写锁(独占锁)的tryAcquire Sync.tryAcquire()

```java
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            //获取当前状态
            int c = getState();
            //获取当前写锁状态
            int w = exclusiveCount(c);
            //不等于0就不是第一次进入
            if (c != 0) {
                // 如果写锁为0说明当期被读锁锁住(当前进来的是写锁)    或者不是当前线程 都是获取锁失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //当前写锁重入个数再加1 大于最大值2的16次方-1=65535
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            //第一次进入writerShouldBlock在NofairSync里面返回false走compareAndSetState(0，1)
            //就是给锁加1
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;//没有设置加1成功就是获取锁失败，返回false
            //成功设置状态加1，把线程设置为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
```

NofairSync.writerShouldBlock()

```java
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
```

##### 写锁的tryAcquire Sync.tryRelease()

```java
        protected final boolean tryRelease(int releases) {
            //判断是否是写锁
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            //写锁的重入数量减1
            int nextc = getState() - releases;
            //是否还有写锁
            boolean free = exclusiveCount(nextc) == 0;
            if (free)//没有写锁了，设置当前线程为空
                setExclusiveOwnerThread(null);
            setState(nextc);
            //如果没有写锁了就返回true，还有锁就返回false
            return free;
        }
```

##### 读锁(共享锁)

```java
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            //如果被写锁锁住 并且来请求读锁的是其他线程 
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;//??
            int r = sharedCount(c);//获取读锁状态
            //readerShouldBlock保证公平不阻塞 && 小于最大的写锁 && 获取写锁成功
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {//前16位的1粒度就是一个1的十六次方
                if (r == 0) {//如果读锁为零
                    firstReader = current;//第一个读锁就是当前线程
                    firstReaderHoldCount = 1;//当期
                } else if (firstReader == current) {//r!=0 并且是当前线程，就是当前线程重入了
                    firstReaderHoldCount++;//第一个读锁线程个数加1 
                } else {//另外一个线程进来了
                    HoldCounter rh = cachedHoldCounter;//获取记录当前其他线程和重入次数的Threadlocal
                    //如果cachedHoldCounter里没有值或者没有记录当前线程
                    if (rh == null || rh.tid != getThreadId(current))
                        //新建记录当前线程重入次数的HoldCounter
                        cachedHoldCounter = rh = readHolds.get();
                    //如果没有获取到
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);//循环直到拿到返回值
        }
```

记录其他线程重入次数的其他线程的ThreadLocal

```java
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }

        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         */
		//继承ThreadLocal是为了可以直接initialValue()返回一个HoldCounter
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```

读锁的tryReleaseShare

```java
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)//如果是最后一个读锁 
                    firstReader = null;
                else
                    firstReaderHoldCount--;//？？是否有线程安全问题，可能都是自己给自己加锁释放的时候都是一个一个来的
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();//获取到记录当前线程重入数的HoldCounter
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {//自旋将读锁设置减去一个
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0; //所有的读锁都释放了，才是真正的释放锁了
            }
        }
```

