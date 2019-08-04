---
layout: post
title: "netty中的epoll实现"
---
## 前言

在java中，IO多路复用的功能通过nio中的`Selector`提供，在不同的操作系统下jdk会通过spi的方式加载不同的实现，比如在macos下是`KQueueSelectorProvider`，`KQueueSelectorProvider`底层使用了kqueue来进行IO多路复用；在linux 2.6以后的版本则是`EPollSelectorProvider`，`EPollSelectorProvider`底层使用的是epoll。虽然jdk自身提供了selector的epoll实现，netty仍实现了自己的epoll版本，根据[netty开发者在StackOverflow的回答](https://stackoverflow.com/a/23465481)，主要原因有两个：

1. 支持更多socket option，比如TCP_CORK和SO_REUSEPORT
2. 使用了边缘触发（ET）模式

接下来就来看看netty自己实现的epoll版本的大概逻辑。

## 总体介绍

### 使用方式

在netty中，如果需要使用netty自己的epoll实现，需要在项目中添加netty-transport-native-epoll依赖，然后将代码中的`NioEvnetLoop`、`NioSocketChannel`、`NioServerSocketChannel`等替换为Epoll开头的类即可。具体参考[Using the Linux native transport](https://netty.io/wiki/native-transports.html)

### 与jdk原生实现的区别

总的来说，不管是jdk还是netty的版本，都是直接调用了linux的epoll来提供IO多路复用，netty的epoll实现与jdk的区别主要有两个：

1. 使用了边缘触发（可以参考我的[另一篇文章](https://juejin.im/post/5cdaa67f518825691b4a5cc0)）
2. 使用了eventfd和timerfd来实现唤醒和超时控制，而jdk的实现则是使用了pipe和epoll自带的超时机制

## 具体实现

### 初始化

`EpollEventLoop`在初始化时会创建三个fd：epollFd、eventFd、timerFd。epollFd用于进一步调用epoll_wait，而另外两个fd的作用前面已经提到了。除此之外，`EpollEventLoop`内部还维护了一个`selectStrategy`变量，`selectStrategy`用于决定当前的loop中的行为，内容不算复杂，具体的就不再展开了。

`EpollEventLoop`还维护了一个`EpollEventArray`类型的对象events，events就是epoll调用时的第二个参数，表示感兴趣的描述符集合，这个变量会被传递到native方法中。

此外`EpollEventLoop`还有一个`IntObjectMap<AbstractEpollChannel>`类型的channels字段，表示当前EventLoop注册的所有Channel对象，其中key是channel对应的fd（文件描述符），因为epoll中接受的参数和返回的结果都是以整数形式的文件描述符表示的，value就是一个Channel对象，后续对Channel进行读写都会从这里查找（注：这里使用的IntObjectMap是netty自己实现的集合，主要目的是提升使用原生类型作为key或者value时的集合的性能，类似的实现还有hppc、FastUtil等等）。

### 注册感兴趣的连接

`EpollEventLoop`的`doRegister`方法中实现了注册连接的逻辑，就是调用`EpollEventLoop`的`add`方法：

```java
    void add(AbstractEpollChannel ch) throws IOException {
        assert inEventLoop();
        int fd = ch.socket.intValue();
        Native.epollCtlAdd(epollFd.intValue(), fd, ch.flags);
        AbstractEpollChannel old = channels.put(fd, ch);
    }
```

可以看到这里调用了`Native.epolCtlAdd`，从名字就可以看出来，底层是调用了epoll_ctl方法，然后op参数为EPOLL_ADD。

### 事件循环

`EpollEventLoop`的主体就在它的`run`方法里，在`run`方法的主循环中会先通过`selectStrategy`决定要进行的操作是epollWait还是epollBusyWait。epollWait和epollBusyWait的区别就在于前者会计算出适合的超时时间然后调用一次epoll_wait直到有描述符就绪或超时，而后者会循环调用epoll_wait并将超时时间设置为0（也就是立即返回）直到有连接就绪为止。

通过epollWait或者epollBusyWait获得的结果会保存在events当中，所以接下来就是调用`processReady`处理events中的各个就绪的fd。处理的过程就是根据fd从channels查到对应的channel然后进行读写等操作，详细的读写就不再展开介绍了。

### 超时和唤醒

前面提到了，netty的epoll逻辑中使用了eventfd和timerfd来实现唤醒和超时控制，evnetfd和timerfd从linux 2.6.22版本开始加入内核，其主要功能就是提供事件通知机制。eventfd可以创建一个文件描述符，在这个描述符上可以传递无符号整数，可以用来作为控制信息。timerfd也是创建一个文件描述符，在这个描述符上可以读取定时器事件，timerfd可以支持到纳秒级别。由于eventfd和timerfd都是基于描述符的，所以和select/poll/epoll这些api都比较契合。

`EpollEventLoop`在初始化时会首先创建epollfd、eventfd和timerfd，然后把eventfd和timerfd都加入到epoll的监听队列当中。eventfd用来做唤醒的支持，当需要唤醒`EpollEventLoop`时，就往eventfd写入一个数，这时eventfd就会变得可读，epoll就会及时返回。timerfd则作为epoll的超时控制，当需要超时的时候就在timerfd上设置一个时间间隔，超时时间到了之后timerfd就会变得可读，epoll也就会及时返回。这里使用timerfd作为超时控制而不是使用epoll自带的超时的原因大概有两个，一是使用timerfd可以用统一的处理方式对待超时事件和IO事件，二是timerfd支持的超时时间精度更高。

顺便提一下，在jdk原生的实现中，唤醒是通过pipe实现的，`Selector`内部维护了一个pipe，初始化时将pipe的read端加入epoll的监听队列，当需要唤醒时就在pipe的write端写入数据，这样epoll就会及时返回。epoll返回后如果发现pipe可读，则将pipe中的数据读取完。

## 其他

在之前的文章中提到过，将fd注册到epoll时如果采用了边缘触发，那么建议的使用方式是将fd设置为非阻塞模式，并且在描述符就绪时需要将就绪数据全部读取完（遇到EAGAIN）为止，否则可能会出现再也无法收到就绪通知的情况。

而在netty的epoll实现中，所有的socket都是以ET模式注册的，而eventfd和timerfd则稍有不同。在netty 4.1.38.Final以前的版本，eventfd在注册到epollfd时使用时LT而不是ET，在每次processReady时如果eventfd可读则都会对其调用一次read。timerfd在注册到epollfd时使用的时ET，但是在每次processReady时如果timerfd可读也会对其调用一次read。而在4.1.38.Final版本，eventfd和timerfd都使用了ET，但是并不在processReady方法中读取这两个fd。对于eventfd，会在每次write返回EAGAIN时调用一次read，因为eventfd内部只能存储一个整数，所以当write出现EAGAIN时就说明目前有数据需要读取。而对于timerfd则只会在epollWait出现超时的时候调用一次read，其他情况下不会对timerfd调用read。因为在netty的实现中，每次进行epoll_wait时都会重新设置timerfd的超时时间，而每次更新timerfd的超时时间时，timerfd就会重新变为不可读状态，也就不用对其调用read了。
