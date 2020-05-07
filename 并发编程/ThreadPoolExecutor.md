## 一、ThreadPoolExecutor概述

当我们新添加一个任务到线程池时:

- 如果线程池中的线程数量少于corePoolSize(核心线程数)，就创建新的线程来运行。
- 如果线程池中线程的数量大于等于corePoolSize，并且阻塞队列workQueue未满，就将任务添加到workQueue 中。
- 如果线程池中线程的数量介于corePoolSize和maximumPoolSize 之间，并且阻塞队列已满，创建新线程去执行。
- 如果阻塞队列已满，并且线程数量等于maximumPoolSize，则执行拒绝策略。

## 二、ThreadPoolExecutor成员变量



```java
    // 线程池的控制状态:用来表示线程池的运行状态（整型的高3位）和运行的worker数量（低29位）
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 29位偏移量
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 最大容量（2^29 - 1）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // 线程运行状态，总共有5个状态，需要3位来表示（所以偏移量的29 = 32 - 3）
    /**
     * RUNNING    :    接受新任务并且处理已经进入阻塞队列的任务
     * SHUTDOWN    ：    不接受新任务，但是处理已经进入阻塞队列的任务
     * STOP        :    不接受新任务，不处理已经进入阻塞队列的任务并且中断正在运行的任务
     * TIDYING    :    所有的任务都已经终止，workerCount为0， 线程转化为TIDYING状态并且调用terminated钩子函数
     * TERMINATED:    terminated钩子函数已经运行完成
     **/
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }// 获取线程池运行状态 CAPACITY是最大容量，取前三位
    private static int workerCountOf(int c)  { return c & CAPACITY; }// 获取线程数量 用&去掉前三位就是当前运行的线程数
    private static int ctlOf(int rs, int wc) { return rs | wc; }//rs表示runState，wc表示workState，两个合成ctl

    // 阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    // 可重入锁
    private final ReentrantLock mainLock = new ReentrantLock();
    // 存放工作线程集合
    private final HashSet<Worker> workers = new HashSet<Worker>();
    // 终止条件
    private final Condition termination = mainLock.newCondition();
    // 最大线程池容量
    private int largestPoolSize;
    // 已完成任务数量
    private long completedTaskCount;
    // 线程工厂
    private volatile ThreadFactory threadFactory;
    // 拒绝执行处理器
    private volatile RejectedExecutionHandler handler;
    // 线程等待运行时间
    private volatile long keepAliveTime;
    // 是否运行核心线程超时
    private volatile boolean allowCoreThreadTimeOut;
    // 核心池的大小
    private volatile int corePoolSize;
    // 最大线程池大小
    private volatile int maximumPoolSize;
    // 默认拒绝执行处理器
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

```

## 三、构造方法

设置核心线程数，线程池最大容量，保持存活时间(0不进行销毁)，时间单位，采用的阻塞队列，ThreadFactory(接口，返回一个Thread)，拒绝策略handler(默认AbortPolicy)

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }   

public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

为了简化创建线程池的步骤

```
ExecutorService pool = Executors.newFixedThreadPool(10);
//核心线程为0，线程池最大为Integer.MAX_VALUE，60秒后销毁，是一个用的时候创建，不用的时候销毁的线程池。
ExecutorService pool2 = Executors.newCachedThreadPool();
```

### 线程工厂ThreadFactory

返回一个包装有Runnable的Thread，线程池在执行时其实就是调用的返回的那个Thread的start方法。当线程池要启动线程时，就会去回调它。我们可以在实现时指定一些线程的优先级什么的。

```java
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```

### ThreadPoolExecutor默认ThreadFacotry

```java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            //指定线程组，指定线程的Runnable，指定线程名，栈的大小
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);//设置线程优先级为普通优先级
            return t;
        }
    }
```

### 四种拒绝策略

拒绝策略需要实现RejectedExecutionHandler，也可以自定义

#### AbortPolicy 

当有任务来时直接抛异常

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

#### DiscardPolicy 

直接抛弃该任务，不处理

```java
    public static class DiscardPolicy implements RejectedExecutionHandler {
        public DiscardPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

#### CallerRunsPolicy

 直接由执行该方法的线程继续执行该任务，除非线程池已经shutdown，这个任务才会被丢弃，否则继续执行该任务会发生阻塞。 

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {

    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

#### DiscardOldestPolicy

抛弃queue中的第一个任务 ,执行该任务

```java
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public DiscardOldestPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
}
```

## 四、execute提交任务

1. 如果线程池当前线程数量小于corePoolSize，就addWorker(command,true)创建新线程，创建成功就返回，失败则继续执行后续步骤，addWorker失败的原因是:

     - 线程池已经shutdown，shutdown的线程不再接收新任务。

        -  判断完workerCountOf(c)<corePoolSize后，由于并发，别的线程先创建了worker线程，导致workerCount>=corePoolSize

2. 如果线程池还在running状态，将task加入workQueue阻塞队列中，如果加入成功，进行double-check，如果加入失败(可能是队列已满)，则执行后续步骤。double-check主要目的是判断刚加入的workQueue阻塞队列的task是否能被执行

   - 如果线程池已经不是running状态了，应该拒绝添加新任务，从workQueue中删除任务

   - 如果线程池是运行状态，或者从workQueue中删除任务失败(线程池里的线程已经把这个任务消耗了)，判断当前是否还有活动的worker，如果没有就添加一个(只要还有一个worker就会消耗阻塞队列中的任务)

3. 如果线程池不是running状态或者无法入队列，尝试开启新线程，扩容至maxPoolSize，如果addWork(command,false)失败了，拒绝当前command。//不是running状态了，虽然addWork里面有校验，但是应该和addWork(command,false)有关，应该逻辑就是非running状态然后就addWork(command,false)

    

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();//获取ctl状态，前三位
        //1.当前线程池运行线程数量如果小于核心线程数，新建线程(addWork有对线程池状态的判断，所以这里没加)
        if (workerCountOf(c) < corePoolSize) {
            //addWorker成功返回
            if (addWorker(command, true))
                return;
            //addWorker可能会失败:
            //线程池shutdown,不再接收线程的addWorker
            //别的线程抢到了核心线程，所以当前线程addWorker失败
            c = ctl.get();
        }
        //2.当前线程数量大于核心线程数，并且任务入队列成功
        //当前线程池是RUNNING状态并且
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //双重验证，线程池是非RUNNING状态就移除任务
            //保证在execute期间，如果外部调用shutdown
		/**
         * 再次校验放入workerQueue中的任务是否能被执行
         * 1、如果线程池不是运行状态了，应该拒绝添加新任务，从workQueue中删除任务
         * 2、如果线程池是运行状态，或者从workQueue中删除任务失败（刚好有一个线程执行完毕，并消耗了这个任务），确保还有线程执行任务（只要有一个就够了）
         */
            //如果再次校验过程中，线程池不是RUNNING状态，并且remove(command)--workQueue.remove()成功，拒绝当前command
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //这里只比较worker数是否为0，而不是worker数和corePoolSize比较的原因是
            //这个比较是上一步负责的，这里只负责增加一个在阻塞队列中等待的线程，确保会有worker去消费它
            //因为只要有一个worker就会消耗workQueue中等待的线程。
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //3.到达这里的条件是线程池不是RUNING状态或者无法入队列
	    //尝试开启新线程，扩容至maxPoolSize，如果addWork(command, false)失败了，拒绝当前command
        else if (!addWorker(command, false))
            reject(command);
    }

```

reject代码

```java
    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```

## 五、addWorker(Runnable firstTask, boolean core)

addWorker就是把firstTask用ThreadFactory包装成Thread并赋值给Woker然后将Worker放到workers里执行任务。

### addWorker的两个参数

firstTask woker线程的初始任务，可以为空woker继承AQS、Runnable

core	为true时将corePoolSize作为判断上限，为false时将maxinumPoolSize作为上限

### addWorker方法有四种构造方法

| 构造方法                 | 线程池上限                  |
| ------------------------ | --------------------------- |
| addWorker(command,true)  | 线程池上限是corePoolSize    |
| addWorker(command,false) | 线程池上限是maximumPoolSize |
| addWorker(null,false)    | 线程池上限是maximumPoolSize |
| addWorker(null,true)     | maximumPoolSize             |

第一个：当线程数小于corePoolSize时，放一个需要处理的task进workers。如果wokers的长度超过corePoolSize，就返回false。

第二个：当队列被放满时，就尝试将这个新来的task直接放入workers，这时候传递false，wokers的长度上限就是maximumPoolSize，如果线程池也满了就返回false。

第三个：放一个空的task进workers，长度限制是maximumPoolSize。这样一个task为空的worker在线程执行的时候会去阻塞队列消耗任务。

第四个：放一个空的task进workers，长度限制是corePoolSize，如果超过corePoolSize就返回false，什么都不干。是用在prestartAllCoreThreads ()方法为ThreadPoolExecutor预先启动corePoolSize个worker等待从阻塞队列workeQueue中获取任务执行。

### 执行流程

1. 判断线程池当前是否为可以添加worker线程的状态，不是则返回false:

   - 线程池状态>shutdown，可能为SHUTDOWN,STOP,TIDING，TERMINATED 不能添加woker线程。

   - 线程状态==shutdown，firstTask不为空，不能添加worker线程，因为shutdown状态的线程池不接收新任务。

   - 线程状态==shutdown，firstTask==null，workerQueue为空，不能添加woker线程，因为firstTast为空是为了添加一个没有任务的线程再从workQueue获取task，而workerQueue为空，说明添加无任务线程已经没有意义。	

      ​	

2. 线程池当前线程数量是否超过上限(根据corePoolSize或maximumPoolSize)，超过了return false，没超过则wokerCount加1，继续下一步。

3. 在线程池的ReentrantLock保证下，向workers中添加woker实例，添加后完成解锁，并启动woker线程，如果这一切都成功了，return true，如果添加wokers失败或启动，调用addWorkFailed()逻辑。

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry://外层循环
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);//获取线程池前三位的运行状态

            /*
             * rs 状态RUNNABLE=-1，SHUTDOWN=0,STOP=1,TIDING=2，TERMINATED=3
             * 在rs已经是SHUTDOWN状态的基础上判断是否是以下三个条件之一
             * 线程池状态是STOP=1,TIDING=2，TERMINATED=3 之一
             * 或者 firstTask有任务，即在线程池已经SHUTDOWN的情况下还想要添加新的任务
             * 或者 阻塞队列workQueue是空的，那么就不需要worker来消耗阻塞队列的任务
             *
             */
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            //循环以完成增加
            for (;;) {
                //获取当前运行的线程数量
                int wc = workerCountOf(c);
                //当前运行的线程数量大于总容量或者大于 (core是true 核心线程数 线程池最大容量)
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //CAS将当前线程数增加1(低29位woker数量) 成功就跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //运行到这CAS将woker数量加1失败，再次获取线程池的当前状态
                c = ctl.get();  // Re-read ctl
                //如果当前状态和进入retry循环时获取到的不一样 从外层开始继续循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //1.将firstTask(Runnable类型)传递给Worker
            //2.Worker继承AQS，设置setState(-1);就是未上锁
            //3.用ThreadFactory将传递进去的firstTast包装成一个Thread类并赋值给Worker的成员变量thread
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //如果线程池还在运行或者 线程停止了但是阻塞队列里还有任务(firstTask==null,创建没有初始任务的Worker去消耗阻塞队列)
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable 线程已经启动，抛非法线程状态异常
                            throw new IllegalThreadStateException();
                        workers.add(w);//workers是HashSet
                        int s = workers.size();
                        //设置最大池大小
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果往HashSet中添加woker成功，启动线程。
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //如果启动线程失败
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

## 六、内部类Worker

Woker类本身既实现了Ruunable，又继承了AbstractQueueSynchronized，所以即是一个可以执行的任务，又可以达到锁的效果。

### new Worer()

1. 将AQS的state设置为-1，在runWorker()前不允许中断
2. 待执行的Runnable任务以参数的形式赋值给firstTask
3. ThreadFacotry用Worker这个R unnable创建Thread
4. 之所以Worker将要运行的任务包装成Thread，运行时通过runWorker执行thread.start()，是要通过Worker控制中断，firstTask只负责需要执行的任务。

### Worker控制中断

1. 设置AQS初始state设置为-1，此时不允许中断interrupt()，只有woker线程启动了，执行了runWorker()，将state设置为0，才允许中断，避免你刚初始化Worker()还没有运行任务就被ThreadPoolExecutor的shutdown()之类的函数影响。

   实现细节为：

   - shutdown()线程池时，会对每个worker  tryLock()上锁，而Worker类的tryAcquire()方法是固定的将state从0变成1，所以刚初始化完的woker状态为-1，tryAcqurie()失败，不能运行interrupt()。
   - shutdownNow()线程池时，不用tryLock()上锁，会去调用worker.interruptIfStarted()终止worker， interruptIfStarted()必须在state>0时才能执行。

2. 为了防止某些情况下，运行中的worker被中断，runWorker()每次运行任务都会lock()上锁，而shutdown()这类可能会终止worker的操作需要先获取worker的锁，这样就防止了中断正在运行的线程。

3. Worker实现的AQS为不可重入锁(从0变成1)，防止在运行时的setCorePoolSize()之类的其他方法会将锁的数量增加(比如setCorePoolSize里调用interruptIdleWorkers()方法就是给worker加锁，中断线程之类)。

```java
    /**
     * Worker类投机取巧的继承了AbstractQueueSynchronized来简化在执行任务时的获取、释放锁
     * 这样防止了中断在运行中的任务，只会唤醒(或中断)在等待从workQueue中获取任务的线程
     * 解释:
     *    为什么不直接执行execute(command)提交的command，而要在外面包一层Worker呢
     *    主要是为了控制中断
     *    用什么控制？
     *    用AQS锁，当运行时上锁，就不能中断，ThreadPoolExecutor的shutdown()方法中断前都要获取worker锁
     *    只有在等待从workQueue中获取任务getTask()时才能中断
     * worker实现了一个简单的不可重入的互斥锁，而不是使用ReentrantLock可重入锁
     * 因为我们不想让调用比如setCorePoolSize()这种线程控制方法时可以再次获取锁(重入)，也就是你刚new Worker()完，还没有执行，就被这些函数影响了，不能执行runWorker
     * 解释:
     *    setCorePoolSize()时可能会interruptIdleWorkers()，在对一个线程interrupt时会要w.tryLock()
     *    如果可以重入，就可能会在对线程池操作的方法中中断线程，类似的方法还有
     *    setMaximunPoolSize()
     *    setKeepAliveTime()
     *    allowCoreThreadTimeOut()
     *    shutdown()
     * 此外，为了让线程真正开始后开可以中断，worker初始化时设置AQS状态为-1，在开始runnWorker时将state设置为0，而state>=0时才可以中断。
     *
     */
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {

        private static final long serialVersionUID = 6138294804551838833L;
        
        final Thread thread;//通过ThreadFactory将Worker(Runnable对象)封装成Thread

        Runnable firstTask;

        volatile long completedTasks;
        Worker(Runnable firstTask) {
            //Worker继承了AQS设置状态为-1，就是未上锁
            setState(-1); // inhibit interrupts until runWorker
                          //在调用runWorker()前，禁止interrupt中断，在，在interruptIfStarted()方法中会判断 getState()>=0
            //成员变量赋值成要运行的 Runnavle
            this.firstTask = firstTask;
            //通过线程工厂，创建一个包含firstTask的线程
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        //就是调用firstTask.run()
        public void run() {
            runWorker(this);
        }
        //0表示没被锁定，1表示被锁定了
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
        /*
         * 尝试获取锁
         * 重写AQS的tryAcquire()，
         */
        protected boolean tryAcquire(int unused) {
            //通过CAS将AQS状态从0变成1，就是原来状态是0，尝试同步状态变成1，每次来获取锁都是从0变成1，所以不存在重入的情况
            if (compareAndSetState(0, 1)) {
                //设置AQS锁拥有者为当前线程。
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        //尝试释放锁
        //AQS锁当前拥有者变为空
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        /*
         * 中断(如果是运行状态)
         * shutdownNow时会循环对worker线程执行
         * 不需要获取worker的锁，即使在worker运行时也能中断
         */
        void interruptIfStarted() {
            Thread t;
            //如果state>=0 && woker当前运行的任务不为空 && woker当前运行的任务没有被中断
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

## 七、getTask()获取任务

1. 首先判断是否可以满足从workQueue中获取任务的条件，不满足return null
   - 线程池状态是否满足：
     - shutdown状态+workQueue为空或stop状态，都不满足，因为被shutdown后还是要执行workQueue也为空，就可以退出了
     - stop状态，shutdownNow()操作会使线程池进入stop状态，此时不接收新任务，中断正在执行的任务，workQueue中的任务也不执行了，故return null返回
   - 线程数量是否超过maximunPoolSize或获取任务是否超时
     - 线程数量超过maximunPoolSize可能是线程池在运行时调用了setMaximumPoolSize()被改变了大小，否则已经addWorker()成功不会超过maximumPoolSize
     - 如果当前线程数量>corePoolSize，才会检查是否获取任务超时，这也体现了当前线程数量达到maximumPoolSize后，如果一直没有新任务，会逐渐终止worker线程直到corePoolSize
2. 如果满足获取任务条件，根据是否需要定时获取调用不同的方法
   - workQueue.poll()：如果keepAliveTime时间内，阻塞队列还是没有用任务，返回null
   - workQueue.take()：如果阻塞队列为空，当前线程会被挂起等待；当队列中有任务加入时，线程被唤醒，take方法返回任务。
3. 在阻塞从workQueue中获取任务时，可以被interrupt()中断，代码中捕获了InterruptedException，重置timedOut为初始值false，再次执行循环，进行第一步的判断满足就继续获取任务，不满足return null，会进入worker退出的流程

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            /*
            线程池状态的判断，两种情况下会将workerCount-1，并返回null
            1.线程池状态为SHUTDOWN，且workQueue为空
            2.线程池状态为STOP(shutdownNow()后会变成STOP，不需要考虑workQueue的情况）
            */
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            //如果allowCoreThreadTimeOut为true，说明corePoolSize和maxinum都需要定时。
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            /*如果(当前运行的worker数大于线程池最大线程数 || (需要定时 && 然后超时了))
               &&
               线程池里运行的
            */
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                //减1成功就返回null，失败循环继续减1
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            /*
                blockingQueue的take()阻塞使用LockSupport.part(this)进入wait状态的，对LockSupport.part(this)进行interrrupt不会抛异常，但还是会有中断响应
                但AQS的ConditionObject的await()对中断状态做了判断，会报告中断状态reportInterruptAfterWait(interruptMode)
                就会上抛InterruptedException，在此捕获，重新开始循环
                如果由于shutdown()等操作导致的空闲worker中断响应，外层循环判断状态时，可能return null
            */
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : //大于corePoolSize
                    workQueue.take();                                     //小于等于corePoolSize
                //获取任务返回
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;//响应中断，重新开始循环，清除中断状态
            }
        }
    }
```

## 八、processWorkerExit()  --  worker线程退出



1. worker数量减1

   - 如果是突然终止，说明是runWorker执行task时异常情况导致，completedAbruptly为true
   - 如果不是突然终止，说明是worker线程没有task可执行了，不用-1，因为已经在getTask()方法里减1了

2. 从Workers Set中移除worker，删除时需要上锁mainlock

3. tryTerminate()：在对线程池有负效益的操作时，都需要“尝试终止”线程池，大概逻辑：

   - 如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其进入退出流程
   - 没有线程了，更新状态tiding->terminated

4. 是否需要增加worker线程，如果线程池还没有完全终止，仍需要保持一定数量的线程线程池状态是running或shutdown

   - 如果当前线程是突然终止的，addWorker()

   - 如果当前线程不是突然终止的，但当前线程数量<要维护的线程数量，addWorker()，如果调用线程池shutdown()，直到workQueue为空前，线程池都会维持corePoolSize个线程，然后再逐渐销毁这corePoolSize个线程

     

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        /*
         1、worker数量-1
         如果突然中止，说明task执行时异常情况导致，即run()方法执行时发生了异常，那么工作的worker线程数量需要-1
         如果不是突然中止，说明worker线程没有task可执行了，不用-1，因为getTask()方法中已经-1了
         */
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        /*
         2、从Workers Set中移除worker
         */
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //把worker的完成任务数加到线程池的完成任务数
            completedTaskCount += w.completedTasks;
            //从HaasSet<Worker>中移除
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        /*
          3.在对线程池有负效益的操作时，都需要“尝试终止”线程池
          主要是判断线程池是否满足终止的状态
          如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其能进入退出流程
          没有线程了，更新状态为tiding->terminated
         */
        tryTerminate();

        /*
          4.是否需要增加worker线程
          线程状态是running 或 shutdown
          如果当前线程是突然终止的，addWorker()
          如果当前线程不是突然终止的，但当前线程数量 < 要维护的线程数量，然后逐渐销毁这coolPoolSize个线程
         */
        int c = ctl.get();
        //如果状态是running、shutdown，即tryTerminate()没有成功终止线程池，尝试添加一个worker
        if (runStateLessThan(c, STOP)) {
            //不是突然完成的，既没有task任务可以获取而完成的，计算min，并根据当前worker数量判断是否需要addWorker()
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                //min为0，即不需要维护核心数量，且workQueue不为空，至少保持一个线程
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                //如果线程数量大于最少数量，直接返回，否则下面至少要addWorker一个
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //添加一个没有firstTask的worker
            //只要worker是completedAbruptly突然终止的，或者线程数量小于要维护的数量，就新添一个worker线程，即使是shutdown状态
            addWorker(null, false);
        }
    }
```

