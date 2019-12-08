---
layout: post
title: 'jetty QueuedThreadPool 源码分析'
---

## 前言
目前手里维护的一个http服务采用的容器是jetty，并且不是spring-boot，连spring都没有，就是手写的jetty server和handler等等。最近在做压测时发现一个奇怪的现象：jetty的线程池在达到满载（最大2000个线程）之后，即使降低了压力，线程池的线程数仍然没有及时的下降到正常水平，而是非常缓慢地下降，大约每两个小时下降1%左右。

这让我感到很奇怪，明明线程的idle时间设置的是5min，为什么线程数过了很久都没有恢复正常，本以为jetty里的线程池应该和jdk自带的线程池没什么区别，看来事实并不是我想象的那样，于是决定了解下jetty提供的QueuedThreadPool的具体实现（基于9.4.8.v20171121版本）。

## 源码部分

我们在项目中使用的是QueuedThreadPool，它继承了AbstractLifeCycle类，同时实现了SizedThreadPool以及Dumpable两个接口。AbstractLifeCycle是jetty中用于管理实例生命周期相关的逻辑，SizedThreadPool则继承自jdk的ThreadPool，在其基础上增加了一些用于获取ThreadPool状态的方法，而Dumpable则声明了将对象dump为string的方法。

QueuedThreadPool采用了jdk中的ThreadPoolExecutor不同的实现方式，看起来逻辑似乎更简单。

### 成员变量

QueuedThreadPool的成员变量并不多，具体列列举如下：

```java
    private final AtomicInteger _threadsStarted; //线程池中一共有多少线程
    private final AtomicInteger _threadsIdle;//线程池中一共有多少空闲的线程
    private final AtomicLong _lastShrink;//上一次“缩小”的时间戳，“缩小”即减少线程数
    private final Set<Thread> _threads;//线程池中的线程
    private final Object _joinLock;//等待所有线程结束用到的lock
    private final BlockingQueue<Runnable> _jobs; //线程池任务队列
    private final ThreadGroup _threadGroup;//线程池任的线程组
    private String _name;//线程池名称
    private int _idleTimeout;//线程空闲的是时间
    private int _maxThreads;//线程池最大size
    private int _minThreads;//线程池最下size
    private int _priority;//线程池中线程的优先级
    private boolean _daemon;//线程池中线程的daemon属性
    private boolean _detailedDump;//dump线程池时是否打印详细的信息
    private int _lowThreadsThreshold;//用于判断线程池是否缺少足够的线程的阈值
    private ThreadPoolBudget _budget;//线程池任务队列
    private Runnable _runnable;//线程池中每个线程的主要运行逻辑
```

### 主要方法

#### doStart

doStart实际上是在AbstractLifeCycle中定义的，表示实例生命周期的开始，QueuedThreadPool中的主要实现就是预先启动_minThreads个线程。

```java
    @Override
    protected void doStart() throws Exception
    {
        super.doStart();
        _threadsStarted.set(0);
        startThreads(_minThreads);
    }
```

#### doStop
doStop也是在AbstractLifeCycle中定义的，表示实例生命周期的结束，QueuedThreadPool中的主要实现就是尝试停止所有线程。在停止过程中，会尝试将所有已提交的任务执行完。
```java
    @Override
    protected void doStop() throws Exception
    {
        super.doStop();
        long timeout = getStopTimeout();
        BlockingQueue<Runnable> jobs = getQueue();
        //如果没有指定stopTimeout，直接清空任务队列
        if (timeout <= 0)
            jobs.clear();
        //用noop把队列填满，目前一共有多少个线程就提交多少个noop任务
        Runnable noop = () -> {};
        for (int i = _threadsStarted.get(); i-- > 0; )
            jobs.offer(noop);
        //先用stopTimeout一半的时间来让所有任务自然执行完
        long stopby = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(timeout) / 2;
        for (Thread thread : _threads)
        {
            long canwait = TimeUnit.NANOSECONDS.toMillis(stopby - System.nanoTime());
            if (canwait > 0)
                thread.join(canwait);
        }
        //如果到这里还有任务没执行完，再尝试激进一点的策略
        //把剩余的线程interrupt一遍
        if (_threadsStarted.get() > 0)
            for (Thread thread : _threads)
                thread.interrupt();
        //再用stopTimeout一半的时间来让所有任务自然执行完
        stopby = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(timeout) / 2;
        for (Thread thread : _threads)
        {
            long canwait = TimeUnit.NANOSECONDS.toMillis(stopby - System.nanoTime());
            if (canwait > 0)
                thread.join(canwait);
        }
        Thread.yield();
        int size = _threads.size();
        if (size > 0) //如果还有任务没执行完，就把剩下的线程都打印一遍
        {
            Thread.yield();
            if (LOG.isDebugEnabled())
            {
                for (Thread unstopped : _threads)
                {
                    StringBuilder dmp = new StringBuilder();
                    for (StackTraceElement element : unstopped.getStackTrace())
                    {
                        dmp.append(System.lineSeparator()).append("\tat ").append(element);
                    }
                    LOG.warn("Couldn't stop {}{}", unstopped, dmp.toString());
                }
            }
            else
            {
                for (Thread unstopped : _threads)
                    LOG.warn("{} Couldn't stop {}",this,unstopped);
            }
        }
        if (_budget!=null)
            _budget.reset();

        synchronized (_joinLock)
        {
            _joinLock.notifyAll();
        }
    }
```

总的来说，结束过程中会在指定的超时时间过去一半的时候把还没执行完的线程都interrupt一下，如果到最后还有任务没执行完，就把它们dump并打印出来。

#### execute

execute即向线程池提交任务的方法，QueuedThreadPool的实现也很简单，就是尝试把任务放到队列里，然后按需创建新的线程。如果入队列失败，则抛出拒绝异常。这里并不向ThreadPoolExecutor一样支持指定reject发生时的处理侧率，而是直接抛出一个拒绝异常。

```java
    @Override
    public void execute(Runnable job)
    {
        if (LOG.isDebugEnabled())
            LOG.debug("queue {}",job);
        if (!isRunning() || !_jobs.offer(job)) //尝试将任务放入队列
        {
            LOG.warn("{} rejected {}", this, job);
            throw new RejectedExecutionException(job.toString());
        }
        else
        {
            //如果入队列成果之后发现一个线程都没了，就重新创建一个线程
            if (getThreads() == 0)
                startThreads(1);
        }
    }
```

#### 线程的主要执行逻辑

线程的主要执行逻辑就是runnable成员变量了，其大体的思路就是：

1. 线程启动后，开始从任务队列中循环获取任务。
2. 如果能成功获取任务，就执行获取到的任务。
3. 如果无法获取新的任务，则跳出获取任务的循环，将_threadsIdle加一，标识自身进入了idle状态
4. 当自身出于idle状态时，根据指定的条件判断是否需要杀掉空闲的线程

```java
private Runnable _runnable = new Runnable()
    {
        @Override
        public void run()
        {
            boolean shrink = false; //是否杀掉自身的标识
            boolean ignore = false;//自身是否属于意外退出的标识
            try
            {
                Runnable job = _jobs.poll(); 从队列中获取任务，不会阻塞
                if (job != null && _threadsIdle.get() == 0) 
                {
                    //如果获取到了任务，同时没有空闲线程，就创建一个线程。这里是否有必要？
                    //虽然_threadsIdle，但是当前线程不就是空闲的吗？看9.4.24.v20191120版本的实现里已经没有这逻辑了
                    startThreads(1);
                }
                loop: while (isRunning())
                {
                    //循环获取任务并执行
                    while (job != null && isRunning())
                    {
                        if (LOG.isDebugEnabled())
                            LOG.debug("run {}",job);
                        runJob(job);
                        if (LOG.isDebugEnabled())
                            LOG.debug("ran {}",job);
                        if (Thread.interrupted())
                        {
                            ignore=true;
                            break loop;
                        }
                        job = _jobs.poll();
                    }
                    //进入空闲状态
                    try
                    {
                        _threadsIdle.incrementAndGet();
                        while (isRunning() && job == null)
                        {
                            //如果没有指定idleTimeout，就阻塞地尝试获取任务
                            if (_idleTimeout <= 0)
                                job = _jobs.take();
                            else
                            {
                                //判断是否需要结束自己：1. size大于_minThreads 2. 距离上次结束线程超过了_idleTimeout
                                final int size = _threadsStarted.get();
                                if (size > _minThreads)
                                {
                                    long last = _lastShrink.get();
                                    long now = System.nanoTime();
                                    if (last == 0 || (now - last) > TimeUnit.MILLISECONDS.toNanos(_idleTimeout))
                                    {
                                        if (_lastShrink.compareAndSet(last, now) && _threadsStarted.compareAndSet(size, size - 1))
                                        {
                                            //如果满足结束的条件，就跳出外层循环，结束执行
                                            shrink=true;
                                            break loop;
                                        }
                                    }
                                }
                                //如果不满足结束的条件，就尝试阻塞获取任务，同时超时时间为idleTimeout
                                job = idleJobPoll();
                            }
                        }
                    }
                    finally
                    {
                        if (_threadsIdle.decrementAndGet() == 0)
                        {
                            startThreads(1);
                        }
                    }
                }
            }
            catch (InterruptedException e)
            {
                ignore=true;
                LOG.ignore(e);
            }
            catch (Throwable e)
            {
                LOG.warn(e);
            }
            finally
            {
                if (!shrink && isRunning())
                {
                    if (!ignore)
                        LOG.warn("Unexpected thread death: {} in {}",this,QueuedThreadPool.this);
                    if (_threadsStarted.decrementAndGet()<getMaxThreads())
                        startThreads(1);
                }
                _threads.remove(Thread.currentThread());
            }
        }
    };
```


## 总结

可以看出QueuedThreadPool比jdk中的ThreadPoolExecutor确实简单的多，而回到最开始的问题，线程池size迟迟不下降的原因就是线程池对idleTimeout的处理方式：当线程出于idle状态时，它就有可能被结束，但只是有可能而不是一定，因为每次结束一个线程都要间隔idleTimeout指定的时间。而我们的项目里指定线程数最大是2000，idleTimeout是5min，就表示每5min才会结束一个线程，所以线程池size下降的速度才会这么慢。