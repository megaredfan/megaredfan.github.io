---
layout: post
title:  "Guava ListenableFuture"
---
## 简介

ListenableFuture继承自java.util.concurrent.Future，旨在降低异步开发的难度。

JDK中定义的Future表示一个异步操作，ListenableFuture在Future的基础上增加了一个addListener方法，可以在计算完成时触发回调逻辑，通过这样的扩展可以支持更多原生Future不支持的逻辑。ListenableFuture中新增的方法addListener(Runnable, Executor)，可以在指定Future上注册一个Runnable对象，这个Runnable会在Future完成的时候通过指定的Executor执行。

此外Guava还提供了一个方法用于在Future上注册回调，使用Futures类：Futures.addCallback(ListenableFuture<V>, FutureCallback<V>, Executor)，或者是Futures.addCallback(ListenableFuture<V>, FutureCallback<V>)。这里需要注意的是如果在注册callback时没有指定Executor，那么注册的callback默认会在当前线程执行，所以对于包含重量级逻辑的callback，最好在注册时指定Executor。

## ListenableFuture的创建

Guava对齐JDK中的ExecutorService.submit(Runnable)提供了ListeningExecutorService，ListeningExecutorService会在ExecutorService返回Future的地方返回一个ListenableFuture。将一个ExecutorService转换成ListeningExecutorService只需要执行MoreExecutorService.listeningDecorator(ExecutorService)。官方文档中的示例如下：

```java
ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
ListenableFuture<Explosion> explosion = service.submit(new Callable<Explosion>() {
  public Explosion call() {
    return pushBigRedButton();
  }
});
Futures.addCallback(explosion, new FutureCallback<Explosion>() {
  // we want this handler to run immediately after we push the big red button!
  public void onSuccess(Explosion explosion) {
    walkAwayFrom(explosion);
  }
  public void onFailure(Throwable thrown) {
    battleArchNemesis(); // escaped the explosion!
  }
});
```

如果你此前使用的FutureTask，Guava提供ListenableFutureTask.create(Callable<V>) 和 ListenableFutureTask.create(Runnable, V)。但与JDK中的FutureTask不同，Guava并不建议直接继承ListenableFutureTask类。

假如你希望直接设置Future的结果而非通过一系列计算得出结果，可以继承AbstractFuture实现，或者直接使用SettableFuture。

如果你不得不从JDK的Future转换成ListenableFuture，唯一的方式是使用JdkFutureAdapters.listenInPoolThread(Future)，但是这种方式会更重一些，因为它会针对每一个添加的Listener创建新的线程来与其绑定，所以如果可能的话，Guava强烈建议通过改造代码直接返回ListenableFuture。

## ListenableFuture的使用

推荐使用ListenableFuture的最重要的原因就是通过ListenableFuture，可以进行一系列复杂的链式异步操作。官方文档的示例如下：

```java
ListenableFuture<RowKey> rowKeyFuture = indexService.lookUp(query);
AsyncFunction<RowKey, QueryResult> queryFunction =
  new AsyncFunction<RowKey, QueryResult>() {
    public ListenableFuture<QueryResult> apply(RowKey rowKey) {
      return dataService.read(rowKey);
    }
  };
ListenableFuture<QueryResult> queryFuture =
    Futures.transformAsync(rowKeyFuture, queryFunction, queryExecutor);
```

ListenableFuture可以支持很多Future不支持的操作，每个Listener可以和不同的Executor绑定，一个ListenableFuture可以有多个操作等待。

ListenableFuture自身的语义很好的表现了fan-out操作（当一个操作开始的时候其他的一些操作也会尽快开始执行），它在完成时会触发所有的回调逻辑。要支持fan-in操作，只需要少许额外的工作，具体参考Futures.allAsList(ListenableFuture...)

| 方法 | 说明 |
| --- | --- |
| transformAsync(ListenableFuture<A>, AsyncFunction<A, B>, Executor) | 返回一个ListenableFuture，它的结果是给定的AsyncFunction利用给定的ListenableFuture的结果作为参数计算的结果。这个AsyncFunction会通过给定的Executor执行 |
| transform(ListenableFuture<A>, Function<A, B>, Executor) | 返回一个ListenableFuture，它的结果是给定的Function利用给定的ListenableFuture的结果作为参数计算的结果。这个Function会通过给定的Executor执行 |
| allAsList(Iterable<ListenableFuture<V>>) | 返回一个ListenableFuture，它的结果是一个包含入参的所有ListenableFuture的结果的List，如果参数中的任意一个ListenableFuture失败或者被取消，这个ListenableFuture就失败或者被取消。 |
| successfulAsList(Iterable<ListenableFuture<V>>) | 返回一个ListenableFuture，它的结果是一个包含入参的所有ListenableFuture的结果的List，如果参数中的任意一个ListenableFuture失败或者被取消，List中对应的元素为null。 |
