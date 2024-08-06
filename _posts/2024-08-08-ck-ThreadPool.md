---
title: ClickHouse ThreadPool  
layout: post
categories: [ClickHouse]
image: /assets/img/rose.jpg
customexcerpt: "ClickHouse线程池"
---

最近有一些适配工作，工作内容本身没什么可说的，不过工作那么多年，很少涉及多线程的代码，所以学习借鉴了一些`ClickHouse`线程池代码。

# 1. ClickHouse ThreadPool

`ClickHouse`最基础的线程池在`Common/ThreadPool.cpp`中，但其他不同模块实际使用时，会进行不同的封装。

<br>

# 2. Key Function

## 2.1 ThreadPoolImpl::ThreadPoolImpl

构造线程池时，主要传入的参数：

- `metric`：监控指标，监控线程池内的线程数量，任务数量。

- `max_threads`：线程池最大的线程数量。

- `max_free_threads`：线程池空闲时，最多允许的线程数量。

- `queue_size`：最多容纳的任务数量。

- `shutdown_on_exception`：当执行任务抛出异常，是否关闭线程池。

<br>

## 2.2 ThreadPoolImpl::scheduleImpl

`ThreadPoolImpl::scheduleImpl`是私有函数，用来向线程池提交任务，对外暴露了不同的接口，用于实现提交任务超时等待，提交任务失败抛`exception`还是返回`false`等功能。

任务队列是一个优先级队列，每个任务由用户设置优先级然后提交给线程池。

<br>

## 2.3 ThreadPoolImpl::wait

等待线程池执行完所有任务，如果任务执行时有异常，则抛出第一个出现的异常。

该函数的使用场景为需要批量执行若干数量任务的场景。

<br>

## 2.4 ThreadPoolImpl::worker

每个线程从任务队列按照优先级高低获取任务并执行，当线程数量被改小后，当前`worker`函数也会自动退出并销毁当前线程。

<br>

# 3. Conclusion

线程池整体的实现没有什么特别的，最后记录一些有意思的点。

## 3.1 Exception Handling

- 线程池中执行的任务本身抛出的异常会被线程池`catch`并保存第一个被`catch`的。但是线程池本身代码执行也可能`throw`出来东西，没错就是内存分配失败导致的`std::bad_alloc`，为了尽量避免这种情况，对于任务队列，内存是提前`reserve`的。但是对于动态增加线程的情况，真的遇到了`std::bad_alloc`也会被代码`catch`住然后抛单独的`exception`给提交任务方。

- 工作线程在执行时，也会尽量避免内存分配，尽量避免线程池本身代码抛出`std::bad_alloc`，比如使用了`DENY_ALLOCATIONS_IN_SCOPE`用于在特定`debug`模式下检查是否有内存动态分配的情况产生。

<br>

## 3.2 Metric

- 线程池初始化时，会传入`metric`用于上报线程池的状态，比如线程数量，任务数量。

- 每次提交任务时，如果打开`enable_job_stack_trace`，会记录当前线程的`frame pointer`，当出现任务内嵌套提交任务的情况，就能记录所有的`frame pointer`，最后如果有错误抛出`exception`，打印`exception`时可以把所有之前的`frame pointer`都打印出来。

- 如果开启了`trace`，比如顺序执行多个任务的情况，会在`Span`内统计每个子任务执行的时间等信息。详见`Common/OpenTelemetryTraceContext.cpp`。


