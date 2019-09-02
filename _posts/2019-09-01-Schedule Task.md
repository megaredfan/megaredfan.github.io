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
