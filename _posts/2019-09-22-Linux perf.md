---
layout: post
title: "linux perf"
---
## 简介

perf是linux系统中提供的性能分析工具，它基于一个叫“Performance counters”的内核子系统实现，同时支持硬件（CPU、PMU(Performance Monitoring Unit)）和软件(软件计数器、tracepoint)层面的性能分析。

### perf中的事件

perf与其他性能调优工具一样，都是通过对监测对象进行采样，根据采样点的分布来推断整个程序的行为。通过perf list命令我们可以看到perf支持很多的采样事件，比如branch-misses、cpu-clock等等。perf中预定义的事件属于不同的类型，比如硬件产生的事件（cache 命中/分支miss）和软件产生的事件（context switch/page fault)等等。

### tracepoint

tracepoint是linux内核中定义的一些hook，如果被开启，它们就会在执行到特定逻辑时被触发，方便其他工具获取系统内部的运行状态等信息，perf就是利用了tracepoint，它会记录和统计tracepoint的各个事件，生成分析报告。

## 使用方式

perf 工具的具体使用方式如下：

```
perf [--version] [--help] COMMAND [ARGS]
```

其中的COMMAND列表可以通过执行perf --help查看，下面列举几个常用的command。

### perf stat

perf stat的作用是执行一个命令并收集其运行过程中的各个数据，它可以提供一个程序运行情况的总体概览。比如：

```
user@localhost:~$ perf stat hostname
localhost

 Performance counter stats for 'hostname':

          0.313464      task-clock (msec)         #    0.481 CPUs utilized          
                 2      context-switches          #    0.006 M/sec                  
                 0      cpu-migrations            #    0.000 K/sec                  
               153      page-faults               #    0.488 M/sec                  
           896,723      cycles                    #    2.861 GHz                    
           620,709      instructions              #    0.69  insn per cycle         
           121,143      branches                  #  386.465 M/sec                  
             6,247      branch-misses             #    5.16% of all branches        

       0.000651441 seconds time elapsed
```

上面这个例子，通过perf stat运行了hostname命令，并将其运行过程中的一些指标汇总显示了出来，比如task-clock、context-switches等待。默认情况下，perf stat 会输出几个常用的事件的统计，比如：

+ task-clock-msecs：cpu 使用率
+ context-switches：进程切换次数
+ page-faults：发生缺页的次数
+ cpu-migrations：表示进程运行过程中发生了多少次CPU迁移，即被调度器从一个CPU转移到另外一个CPU上运行
+ cycles：处理器时钟，一条机器指令可能需要多个cycles
+ instructions: 机器指令数目
+ branches：遇到的分支指令数
+ branch-misses是预测错误的分支指令数

除此之外，我们可以使用-e参数来指定我们感兴趣的事件，比如：

```
user@localhost:~$ perf stat -e cache-misses hostname
localhost

 Performance counter stats for 'hostname':

          682      cache-misses                                                

       0.000646676 seconds time elapsed
```

### perf top

perf top的作用是实时地显示系统当前的性能统计信息。前面的perf stat用于对一个特定的程序进行分析，而某些时候我们可能并不知道是哪个程序影响了系统性能，这时候就可以用perf top来查找可疑的程序。比如：

```
Samples: 775  of event 'cpu-clock', Event count (approx.): 92931021
Overhead  Shared Object       Symbol
   8.93%  [kernel]            [k] vsnprintf
   7.73%  perf                [.] rb_next
   5.92%  [kernel]            [k] kallsyms_expand_symbol.clone.0
   5.07%  [kernel]            [k] format_decode
   4.59%  [kernel]            [k] number
   3.40%  perf                [.] symbols__insert
   3.03%  libslang.so.2.2.1   [.] SLtt_smart_puts
```

上面的例子显示perf统计了cpu-clock事件的数据，根据比例进行了排序。和perf stat一样，我们可以通过-e参数指定统计其他的事件，比如perf top -e context-switches可以查看进程切换最多的top N个进程。

### perf record & perf report

perf record的作用和perf stat类似，它可以运行一个命令并生成统计信息，不过perf record不会将结果显示出来，而是将结果输出到文件中。perf record生成的文件可以用perf report来进行解析。

perf record还可以通过-g参数，在分析时生成calling graph，帮助定位更上层的逻辑分布。

## 其他

通过例子我们可以发现，perf的分析结果中的Symbol一列显示的都是c语言函数的名字。对于java来说，jit编译产生的函数就会直接显示在symbol里，而不是java的函数名，这时要定位问题就不是那么容易了，我们需要通过额外的手段将symbol和java程序的符号表对应起来，具体后续再讨论了。
