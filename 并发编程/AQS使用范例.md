一、ReentrantLock()实现AQS的过程

ReentrantLock()的lock()方法调用的是内部类NonfairSync的lock()方法，NonfairSync的lock方法调用了AQS的acquire()方法，AQS的acquire()方法调用NonfairSync的tryAcquire()，里面调用Sync的nonfairTryAcquire方法

我们要做的就是在tryAcquire里面写拿锁的代码并把AQS的线程拥有者设置为当前线程。

```java
//实现了Lock接口，MyLock2就是一把锁了，但是我们就不实现了，委托给helper来实现
public class MyLock2 implements Lock {

    private Helper helper=new Helper();

    @Override
    public void lock() {
        helper.acquire(1);
    }

    @Override   //可中断
    public void lockInterruptibly() throws InterruptedException {
        helper.acquireInterruptibly(1);
    }

    @Override      //是tryAcquire实现的一个手段
    public boolean tryLock() {
        return helper.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return helper.tryAcquireNanos(1,unit.toNanos(time));
    }

    @Override
    public void unlock() {
        //释放一个锁
        helper.release(1);
    }

    @Override
    public Condition newCondition() {
        //ConditionObject是AQS的子类所以让helper来返回
        return helper.newCondition();
    }

    //就是ReentrantLock里的Sync
    private class Helper extends AbstractQueuedSynchronizer{
        //选择性实现tryAcquire和tryRelease
        @Override         //独占锁
        protected boolean tryAcquire(int arg) {
            //如果第一个线程进来，可以拿到锁，因此我们可以返回true

            //如果第二个线程进来，拿不到锁，返回false。特例：如果当前进来的线程和当前保存的线程是同一个线程，则可以拿到锁，但是有代价，更新状态值

            //如何判断是第一个线程进来，还是其他线程进来

            //官方jdk文档说用getState setState 和 compareAndSetState方法操作以原子方式更新的int值
            int state=getState();
            Thread current=Thread.currentThread();

            if(state==0){//初始化状态为0，可以获取到锁，我们应该把状态设置为1
                //不能用setState(arg)多个线程进来会产生线程问题
                if(compareAndSetState(0,arg)){
                    //设置状态成功，当前线程是第一个进入的线程
                    setExclusiveOwnerThread(Thread.currentThread());
                    return true;
                }
            }else if(getExclusiveOwnerThread()==current){//重入线程
                setState(state+1);
                return true;
            }
            return false;



            //return super.tryAcquire(arg);
        }

        //释放资源时会判断是否是当前线程，所以不会有其他线程进来，不用设置同步操作
        @Override
        protected boolean tryRelease(int arg) {
            //锁的获取和释放肯定是一一对应的，那么调用此方法的线程一定是当前线程

            //因为tryAcquire用setExclusiveOwnerThread设置了当前线程
            //所以我们先判断一下当前线程是否是独占锁的线程
            if(Thread.currentThread()!=getExclusiveOwnerThread() ){
                throw new RuntimeException();
            }

            int state=getState()-arg;//也就是减去1

            boolean flag=false;
            //
            if(state == 0){
                //把当前线程设置为空
                setExclusiveOwnerThread(null);
                flag=true;
            }


            setState(state);
            return flag;
           // return super.tryRelease(arg);
        }

        Condition newCondition(){
            return new ConditionObject();
        }
    }

}
```

