---
layout: post
title: "quatrz"
---

## quartz简介

quartz是一个基于java的开源的任务调度框架。

## quatrz API

### Scheduler

一个Scheduler通过SchedulerFactory创建和销毁。Scheduler提供对Job和Trigger的创建、销毁、暂停等一切调度相关的操作。要启动一个Scheduler，需要显式地调用它的start方法。

### Job

Job是实际的任务执行逻辑的接口，它只有一个方法execute：

```java
  package org.quartz;
  public interface Job {
    public void execute(JobExecutionContext context)
      throws JobExecutionException;
  }


```

当一个Job被触发时，Scheduler会重新创建一个Job，然后通过一个worker线程执行Job的execute方法，所以实际的Job实现类中无法维护状态字段。JobExecutionContext包含了一些“运行时”的信息，比如触发Job的Scheduler、触发的Trigger、Job对应的JobDetail等等。JobDetail是在创建Job时构建的包含Job的各种详细信息的对象，其中还包括一个JobDataMap的对象，JobDataMap中存储了Job的各种状态信息。

### Trigger

Trigger的作用就是触发Job的执行，其中包含了触发时间、触发周期等相关信息。Trigger也有对应的JobDataMap对象，可以用于向Job传递某些参数。Trigger有不同的实现类型，比如SimpleTrigger和CronTrigger。SimpleTrigger的触发类似Java中的Schedule线程池，可以支持单次执行和定时重复执行等；CronTrigger则支持通过cron表达式指定触发时机。

Trigger包含优先级的概念，当有不同的Trigger在同一时间触发时，quartz会根据其优先级决定触发的顺序，因为在资源不足的情况下，

### 将Job和Trigger定义分开的原因

第一个原因，Job和Trigger可以独立存储，并支持多对多的关联，减少重复定义。第二个原因是这样可以降低耦合程度，可以方便后续对Job或者Trigger的编辑和替换。

### 标识

Job和Triiger都有自己的身份标识，身份标识由name和group两部分组成，并规定group+name的组合必须保证在Scheduler范围内唯一。

### JobDataMap

每个JobDetail和Trigger都有与其对应的JobDataMap。用户可以在定义JobDetail和Trigger时传入指定的key-value，Job在执行时就可以从JobExecutionContext中获取对应的key-value。此外quartz还支持将JobDataMap中的key-value与Job的属性字段对应起来，只要Job实现类中定义了与JobDataMap中key-value同名的字段，quartz就会通过setter方法将其设置到创建出来的Job实例中。

quartz还支持Job的持久化，所以在选择要将什么数据存到JobDataMap时要考虑结构变化带来的序列化兼容问题。在Job实现类上加上@PersistJobDataAfterExecution就可以使quartz在任务执行后将JobDataMap更新。@PersistJobDataAfterExecution通常建议和@DisallowConcurrentExecution配合使用，后者会通知quartz不允许并发执行任务，以避免状态同步问题。

### TriggerListener 和 JobListener

quartz提供TriggerLisenter和JobListener，可以在任务触发时获得通知甚至是阻止任务执行。

## 具体逻辑

### Scheduler初始化

Scheduler是通过SchedulerFactory创建的，默认的SchedulerFactory有两种实现：

+ StdSchedulerFactory 标准实现，基于properties文件进行初始化
+ DirectSchedulerFactory 轻量级实现，可以直接通过代码初始化

#### StdSchedulerFactory

StdSchedulerFactory默认会从*当前工作路径*查找名为"quartz.properties"的文件进行初始化；如果没有找到，则会用自带的默认配置文件进行初始化，此外用户还可以通过系统属性"org.quartz.properties"额外指定要加载的配置文件的文件名。除了在配置文件中指定具体的配置，用户还可以通过环境变量以及jvm参数(通过-D指定)覆盖默认的选项。下面是默认的配置文件的内容：

```
org.quartz.scheduler.instanceName: DefaultQuartzScheduler
org.quartz.scheduler.rmi.export: false
org.quartz.scheduler.rmi.proxy: false
org.quartz.scheduler.wrapJobExecutionInUserTransaction: false
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
org.quartz.jobStore.misfireThreshold: 60000
org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```

可以看到主要有scheduler、threadPool和jobStore相关的配置，更多的配置可以参考quartz的官方文档。

StdSchedulerFactory返回的Scheduler对象时StdScheduler，而StdScheduler是对QuartzScheduler的简单封装，QuartzScheduler就是quartz中schedule的核心逻辑了。

#### QuartzScheduler

##### start

start方法会启动一个QuartzSchedulerThread，QuartzScheduler所有的触发操作都发生在这个线程。

##### scheduleJob

scheduleJob会将传入的JobDetai和Trigger存放到JobStore中然后触发相应的Listener以及唤醒scheudle线程。

#### QuartzSchedulerThread

QuartzScheduleThread相当于一个事件循环，它会在循环中执行以下几个任务：

1. 阻塞等待worker线程池可用
2. 从JobStore中获取所有满足触发条件的Trigger
3. 调用JobStore的triggersFired方法触发Trigger，注意这里只是触发Trigger，并没有执行对应的Job。这一步的目的主要是在任务执行前给Trigger一个更新自身状态的机会。
4. JobStore触发Trigger后会读取出Trigger对应的Job信息，根据返回的Job信息构建可执行的JobRunShell对象。
5. 使用worker线程池执行JobRunShell，也就是执行真正的Job逻辑。

### Job持久化（JobStore）

quartz中的JobStore负责储存Job和Trigger定义，它主要与QuartzScheduler交互。按照约定，Job和Trigger的存储都使用group+name的组合作为标识。

目前quartz提供基于内存的实现以及基于JDBC的实现。基于内存的实现把所有Job和Trigger定义存在内存中，一旦重启所有数据就会丢失；基于JDBC的实现将数据存放在数据库中，同时还支持事务。
