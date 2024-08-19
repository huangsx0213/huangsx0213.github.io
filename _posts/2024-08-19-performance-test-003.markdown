---
layout:     post
title:      "JMeter线程组使用说明"
subtitle:   "「 Performance Testing 」" 
date:       2024-08-19 17:00:00
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---

# JMeter线程组详细使用说明

JMeter线程组使用说明文档，包含每种线程组的解释、使用场景和Java代码实现。

## 目录
1. [标准线程组 (Standard Thread Group)](#1-标准线程组-standard-thread-group)
2. [置身时间线程组 (Stepping Thread Group)](#2-置身时间线程组-stepping-thread-group)
3. [终极线程组 (Ultimate Thread Group)](#3-终极线程组-ultimate-thread-group)
4. [同步定时器 (Synchronizing Timer)](#4-同步定时器-synchronizing-timer)
5. [吞吐量定时器 (Throughput Timing)](#5-吞吐量定时器-throughput-timing)
6. [抵达线程组 (Arrivals Thread Group)](#6-抵达线程组-arrivals-thread-group)
7. [自由形式到达线程组 (Free-Form Arrivals Thread Group)](#7-自由形式到达线程组-free-form-arrivals-thread-group)
8. [并发线程组 (Concurrency Thread Group)](#8-并发线程组-concurrency-thread-group)

## 1. 标准线程组 (Standard Thread Group)

### 描述
最常用的线程组类型，允许设置固定数量的线程、加速时间和循环次数。

### 使用场景
- 基本的负载测试
- 模拟固定数量的并发用户
- 适合大多数常规的性能测试场景

### Java代码实现

```java
import org.apache.jmeter.threads.ThreadGroup;

ThreadGroup standardTG = new ThreadGroup();
standardTG.setName("Standard Thread Group");
standardTG.setNumThreads(100);  // 设置线程数
standardTG.setRampUp(30);       // 设置加速时间（秒）
standardTG.setLoops(10);        // 设置循环次数
standardTG.setScheduler(true);  // 启用调度器
standardTG.setDuration(3600);   // 设置持续时间（秒）
standardTG.setDelay(5);         // 设置启动延迟（秒）
```

## 2. 置身时间线程组 (Stepping Thread Group)

### 描述
允许更精细地控制线程的增加和减少，可以定义多个阶段，每个阶段可以增加或减少特定数量的线程。

### 使用场景
- 需要逐步增加负载的场景
- 模拟用户数量逐渐增加的情况
- 测试系统在不同负载水平下的性能

### Java代码实现

```java
import kg.apc.jmeter.threads.SteppingThreadGroup;

SteppingThreadGroup steppingTG = new SteppingThreadGroup();
steppingTG.setName("Stepping Thread Group");
steppingTG.setNumThreads(100);     // 最终线程数
steppingTG.setInitialDelay(10);    // 初始延迟（秒）
steppingTG.setStartUsers(10);      // 初始启动的线程数
steppingTG.setIncrementUsers(10);  // 每次增加的线程数
steppingTG.setIncrementTime(30);   // 每次增加的时间间隔（秒）
steppingTG.setHoldLoad(60);        // 保持负载的时间（秒）
steppingTG.setThreadLifetime(3600);// 每个线程的最大生存时间（秒）
steppingTG.setLoops(10);           // 每个线程执行的循环次数
steppingTG.setUnit(SteppingThreadGroup.UNIT_SECONDS); // 设置时间单位
steppingTG.setOnSampleError(SteppingThreadGroup.ON_SAMPLE_ERROR_CONTINUE); // 错误处理策略
```

## 3. 终极线程组 (Ultimate Thread Group)

### 描述
提供最灵活的线程调度选项，可以精确定义在特定时间点要运行的线程数。

### 使用场景
- 需要精确控制线程数随时间变化的场景
- 复杂的负载模式测试
- 模拟真实世界中的用户行为模式

### Java代码实现

```java
import kg.apc.jmeter.threads.UltimateThreadGroup;
import kg.apc.jmeter.threads.UltimateThreadGroupGui;

UltimateThreadGroup ultimateTG = new UltimateThreadGroup();
ultimateTG.setName("Ultimate Thread Group");
// 添加多个负载阶段
// 参数：开始时间，初始线程数，增加线程数，持续时间，结束线程数
ultimateTG.addScheduleValue(0, 0, 30, 60, 10);
ultimateTG.addScheduleValue(90, 10, 50, 120, 0);

// 如果需要使用GUI设置
UltimateThreadGroupGui gui = new UltimateThreadGroupGui();
gui.configure(ultimateTG);
```

## 4. 同步定时器 (Synchronizing Timer)

### 描述
虽然不是独立的线程组类型，但可用于使多个线程同步执行，模拟突发负载。

### 使用场景
- 模拟突发负载
- 测试系统在大量并发请求同时到达时的性能

### Java代码实现

```java
import org.apache.jmeter.timers.SyncTimer;

SyncTimer syncTimer = new SyncTimer();
syncTimer.setName("Synchronizing Timer");
syncTimer.setGroupSize(100);    // 设置需要同步的线程数
syncTimer.setTimeoutInMs(10000); // 设置超时时间（毫秒）
```

## 5. 吞吐量定时器 (Throughput Timing)

### 描述
允许基于目标吞吐量来控制请求的发送速率。

### 使用场景
- 需要控制请求速率的场景
- 测试系统在特定吞吐量下的性能

### Java代码实现

```java
import org.apache.jmeter.timers.ConstantThroughputTimer;

ConstantThroughputTimer constThroughputTimer = new ConstantThroughputTimer();
constThroughputTimer.setName("Constant Throughput Timer");
constThroughputTimer.setThroughput(1000.0); // 设置目标吞吐量（每分钟请求数）
constThroughputTimer.setCalcMode(1); // 设置计算模式（1表示基于全局线程）
```

## 6. 抵达线程组 (Arrivals Thread Group)

### 描述
关注每秒到达的用户数量，而不是并发用户数，更好地模拟真实世界的用户行为。

### 使用场景
- 模拟基于时间的用户到达模式
- 更接近真实世界的用户行为模拟

### Java代码实现

```java
import com.blazemeter.jmeter.threads.arrivals.ArrivalsThreadGroup;

ArrivalsThreadGroup arrivalsTG = new ArrivalsThreadGroup();
arrivalsTG.setName("Arrivals Thread Group");
arrivalsTG.setTargetLevel(100);    // 设置目标到达率（每分钟）
arrivalsTG.setRampUp(60);          // 设置达到目标到达率的时间（秒）
arrivalsTG.setHold(300);           // 设置保持目标到达率的时间（秒）
arrivalsTG.setUnit("M");           // 设置单位为分钟（M）或秒（S）
```

## 7. 自由形式到达线程组 (Free-Form Arrivals Thread Group)

### 描述
抵达线程组的更灵活版本，允许使用函数来定义到达率。

### 使用场景
- 需要动态调整到达率的场景
- 模拟复杂的、变化的用户行为模式

### Java代码实现

```java
import com.blazemeter.jmeter.threads.arrivals.FreeFormArrivalsThreadGroup;

FreeFormArrivalsThreadGroup freeFormTG = new FreeFormArrivalsThreadGroup();
freeFormTG.setName("Free-Form Arrivals Thread Group");
freeFormTG.setTargetLevel("${__jexl3((time / 60) % 10 == 0 ? 100 : 50)}");
freeFormTG.setRampUp(60);
freeFormTG.setHold(600);
freeFormTG.setUnit("M");  // 设置单位为分钟（M）或秒（S）
```

## 8. 并发线程组 (Concurrency Thread Group)

### 描述
允许维持一个固定的并发用户数，即使某些线程完成了它们的迭代。

### 使用场景
- 需要维持固定并发用户数的场景
- 长时间运行的测试，其中一些用户可能完成测试而其他用户继续

### Java代码实现

```java
import com.blazemeter.jmeter.threads.concurrency.ConcurrencyThreadGroup;

ConcurrencyThreadGroup concurrencyTG = new ConcurrencyThreadGroup();
concurrencyTG.setName("Concurrency Thread Group");
concurrencyTG.setTargetLevel(100);    // 设置目标并发用户数
concurrencyTG.setRampUp(60);          // 设置达到目标并发数的时间（秒）
concurrencyTG.setHold(300);           // 设置保持目标并发数的时间（秒）
concurrencyTG.setUnit("M");           // 设置单位为分钟（M）或秒（S）
```

注意：在实际使用中，你可能需要结合多种类型的线程组，并配合其他JMeter元素（如采样器、断言、监听器等）来构建完整的测试计划。选择合适的线程组类型取决于你的具体测试目标和你想要模拟的用户行为模式。
```

这个Markdown文档提供了JMeter中各种线程组的详细说明，包括每种线程组的描述、适用场景和Java代码实现。你可以根据需要进一步调整或扩展这个文档。