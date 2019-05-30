---
layout: post
title: "Java中的代理"
---

## 序

代理模式是指为一个对象提供另一种访问方式，即通过一个代理对象来访问我们的目标对象，使得我们可以详细地控制访问行为。一般来说代理分为两种：静态代理和动态代理。

## 静态代理

静态代理中通常有以下三个对象：

+ 目标对象（Source）：目标对象实现了真正的业务逻辑，业务逻辑一般用一个接口（Sourceable）定义
+ 代理对象（Proxy）：代理对象实现了和目标对象同样的接口，负责对外提供接口，他接收来自客户的请求
+ 客户对象（Client）：负责请求，并不关心其中的具体实现

下面是一个静态代理的例子：

```java
public interface Sourceable {
    //需要代理的接口
    public void method();
    //接口实现类,操作
    public static class Source implements Sourceable{
        @Override
        public void add() {
            System.out.println("doing");
        }
    }
    //静态代理类的实现.代码已经实现好了.
    public static class Proxy implements Sourceable{
        private Sourceable sourceable;
        public Proxy(Sourceable sourceable) {
            this.sourceable=sourceable;
        }
        @Override
        public void method() {
            //执行一些操作
            System.out.println("begin ");
            sourceable.method();
            System.out.println("end ");
​
        }
    }
}
```

现在如果我们想对这个方法做一层静态的代理，这儿实现了一个简单的代理类实现了Sourceable，构造函数传入的参数是真正的实现类，但是在调用这个代理类的method方法的时候我们在实现方法执行的前后分别做了一些输出日志的操作。

观察代码可以发现每一个代理类只能为一个接口服务，一个Proxy 类实现了一个Sourceable接口，那么我要是有多个接口，是不是要写多个Proxy类与之对应。这样一来程序开发中必然会产生过多的代理，而且，所有的代理操作除了调用的方法不一样之外，其他的操作都一样，则此时肯定是重复代码。解决这一问题最好的做法是可以通过一个代理类完成全部的代理功能，那就是动态代理出现的原因。

## 动态代理

动态代理有以下几个特点：

1. 代理对象不需要实现接口
2. 动态地在内存中构建代理对象，只需要我们指定创建代理对象/目标对象实现的接口的类型

### JDK动态代理

jdk中有自带的动态代理的实现方式，具体示例如下：

```java
public class ProxyFactory implements InvocationHandler {
    private Class<?> target;
    private Object real;
    //委托类class
    public ProxyFactory(Class<?> target){
        this.target=target;
    }
    //实际执行类bind
    public Object bind(Object real){
        this.real=real;
        //利用JDK提供的Proxy实现动态代理
        return  Proxy.newProxyInstance(target.getClassLoader(),new Class[]{target},this);
    }
    @Override
    public Object invoke(Object o, Method method, Object[] args) throws Throwable {
        //代理环绕
        System.out.println("begin");
        //执行实际的方法
        Object invoke = method.invoke(real, args);
        System.out.println("end");
        return invoke;
    }

    public static void main(String[] args) {
        Sourceable proxy =(Sourceable) new ProxyFactory(Sourceable.class).bind(new Sourceable.Source());
        proxy.method();
    }
}
```

JDK 自带的动态代理中有比较关键的InvocationHandler，InvocationHandler的作用就是负责拦截和处理目标对象的方法调用，它的invoke方法分别传入的是代理对象，本次访问的方法以及本次访问的方法参数，返回结果就是本次调用实际需要返回的结果。我们可以在invoke方法内部根据方法或者参数的不同执行我们的拦截逻辑甚至直接返回不同的结果。

JDK动态代理的几个特性：

+ JDK动态代理只能代理有接口的类,并且是能代理接口方法,不能代理一般的类中的方法
+ 必须传入一个InvocationHandler对象，在InvocationHandler中实现代理逻辑
+ java.lang.Object类的equals、hashCode、toString也可以被代理
+ 在invoke方法中我们甚至可以不用Method.invoke方法调用实现类就返回。这种方式常常用在RPC框架中,在invoke方法中发起通信调用远端的接口等

### cglib代理

JDK代理中有一个局限，那就是只能代理实现了接口的目标对象，那对于没有实现接口的对象要如何进行代理呢？这时候就轮到cglib出场了。cglib不同于JDK的实现，采用了继承的方式对目标对象进行代理，示例如下：

```java
Enhancer enhancer=new Enhancer();
enhancer.setSuperclass(Souceable.class);
enhancer.setCallback(new MethodInterceptor() {
            //类似InvocationHandler的invoke方法
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                System.out.println("begin");
                Object invoke = methodProxy.invoke(new Calculator.CalculatorImpl(), objects);
                System.out.println("end");
                return invoke;
            }
});
Sourceable proxy =(Sourceable) enhancer.create();
proxy.method();
```

cglib中的MethodInterceptor与JDK中的InvocationHandler类似，它的intercept方法中传入了代理对象、方法对象以及对应的方法参数，我们可以在这个intercept方法中实现我们的代理逻辑，MethodProxy对象则是目标对象的引用。

cglib的几个特性：

+ 代理的类不能为final,否则报错
+ 目标对象的方法如果为final/static,那么就不会被拦截,即不会执行目标对象额外的业务方法
+ cglib会默认代理Object中finalize,equals,toString,hashCode,clone等方法。比JDK代理多了finalize和clone。
