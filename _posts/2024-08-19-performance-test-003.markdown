---
layout:     post
title:      "JMeter线程组,定时器使用说明"
subtitle:   "「 Performance Testing 」" 
date:       2024-08-19 17:00:00
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---
好的,我将根据您提供的信息重新整理一份完整的JMeter线程组和定时器使用说明。

---

# JMeter线程组和定时器详细使用说明

## 目录
1. [线程组 (Thread Groups)](#1-线程组-thread-groups)
   1. [标准线程组 (Standard Thread Group)](#11-标准线程组-standard-thread-group)
   2. [逐步线程组 (Stepping Thread Group)](#12-逐步线程组-stepping-thread-group)
   3. [终极线程组 (Ultimate Thread Group)](#13-终极线程组-ultimate-thread-group)
   4. [抵达线程组 (Arrivals Thread Group)](#14-抵达线程组-arrivals-thread-group)
   5. [自由形式到达线程组 (Free-Form Arrivals Thread Group)](#15-自由形式到达线程组-free-form-arrivals-thread-group)
   6. [并发线程组 (Concurrency Thread Group)](#16-并发线程组-concurrency-thread-group)
2. [定时器 (Timers)](#2-定时器-timers)
   1. [常量定时器 (Constant Timer)](#21-常量定时器-constant-timer)
   2. [均匀随机定时器 (Uniform Random Timer)](#22-均匀随机定时器-uniform-random-timer)
   3. [高斯随机定时器 (Gaussian Random Timer)](#23-高斯随机定时器-gaussian-random-timer)
   4. [恒定吞吐量定时器 (Constant Throughput Timer)](#24-恒定吞吐量定时器-constant-throughput-timer)
   5. [泊松随机定时器 (Poisson Random Timer)](#25-泊松随机定时器-poisson-random-timer)

## 1. 线程组 (Thread Groups)

线程组定义了JMeter将要模拟的线程（用户）数量以及这些线程执行的计划，是JMeter测试计划的核心要素。

### 1.1 标准线程组 (Standard Thread Group)

#### 描述
最常用的线程组类型，允许设置固定数量的线程、加速时间和循环次数。

#### 使用场景
- 基本的负载测试
- 模拟固定数量的并发用户
- 适合大多数常规的性能测试场景

#### Java代码实现

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

### 1.2 逐步线程组 (Stepping Thread Group)

#### 描述
允许更精细地控制线程的增加和减少，可以定义多个阶段，每个阶段可以增加或减少特定数量的线程。

#### 使用场景
- 需要逐步增加负载的场景
- 模拟用户数量逐渐增加的情况
- 测试系统在不同负载水平下的性能

#### Java代码实现

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

### 1.3 终极线程组 (Ultimate Thread Group)

#### 描述
提供最灵活的线程调度选项，可以精确定义在特定时间点要运行的线程数。

#### 使用场景
- 需要精确控制线程数随时间变化的场景
- 复杂的负载模式测试
- 模拟真实世界中的用户行为模式

#### Java代码实现

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

### 1.4 抵达线程组 (Arrivals Thread Group)

#### 描述
关注每秒到达的用户数量，而不是并发用户数，更好地模拟真实世界的用户行为。

#### 使用场景
- 模拟基于时间的用户到达模式
- 更接近真实世界的用户行为模拟

#### Java代码实现

```java
import com.blazemeter.jmeter.threads.arrivals.ArrivalsThreadGroup;

ArrivalsThreadGroup arrivalsTG = new ArrivalsThreadGroup();
arrivalsTG.setName("Arrivals Thread Group");
arrivalsTG.setTargetLevel(100);    // 设置目标到达率（每分钟）
arrivalsTG.setRampUp(60);          // 设置达到目标到达率的时间（秒）
arrivalsTG.setHold(300);           // 设置保持目标到达率的时间（秒）
arrivalsTG.setUnit("M");           // 设置单位为分钟（M）或秒（S）
```

### 1.5 自由形式到达线程组 (Free-Form Arrivals Thread Group)

#### 描述
抵达线程组的更灵活版本，允许使用函数来定义到达率。

#### 使用场景
- 需要动态调整到达率的场景
- 模拟复杂的、变化的用户行为模式

#### Java代码实现

```java
import com.blazemeter.jmeter.threads.arrivals.FreeFormArrivalsThreadGroup;

FreeFormArrivalsThreadGroup freeFormTG = new FreeFormArrivalsThreadGroup();
freeFormTG.setName("Free-Form Arrivals Thread Group");
freeFormTG.setTargetLevel("${__jexl3((time / 60) % 10 == 0 ? 100 : 50)}");
freeFormTG.setRampUp(60);
freeFormTG.setHold(600);
freeFormTG.setUnit("M");  // 设置单位为分钟（M）或秒（S）
```

### 1.6 并发线程组 (Concurrency Thread Group)

#### 描述
允许维持一个固定的并发用户数，即使某些线程完成了它们的迭代。

#### 使用场景
- 需要维持固定并发用户数的场景
- 长时间运行的测试，其中一些用户可能完成测试而其他用户继续

#### Java代码实现

```java
import com.blazemeter.jmeter.threads.concurrency.ConcurrencyThreadGroup;

ConcurrencyThreadGroup concurrencyTG = new ConcurrencyThreadGroup();
concurrencyTG.setName("Concurrency Thread Group");
concurrencyTG.setTargetLevel(100);    // 设置目标并发用户数
concurrencyTG.setRampUp(60);          // 设置达到目标并发数的时间（秒）
concurrencyTG.setHold(300);           // 设置保持目标并发数的时间（秒）
concurrencyTG.setUnit("M");           // 设置单位为分钟（M）或秒（S）
```

## 2. 定时器 (Timers)

定时器用于在请求之间定义延迟，这是模拟现实用户行为的关键。

### 2.1 常量定时器 (Constant Timer)

#### 描述
在每个请求前增加固定的延迟。

#### 适用场景
需要在请求之间有固定延迟以模拟稳定的用户行为。

#### 主要参数
- **线程延迟 (Thread Delay):** 以毫秒为单位的延迟时间。

#### Java代码实现

```java
import org.apache.jmeter.timers.ConstantTimer;
import org.apache.jmeter.testelement.TestElement;

// 创建常量定时器
ConstantTimer constantTimer = new ConstantTimer();

// 设置定时器名称
constantTimer.setName("Constant Timer");

// 设置5秒延迟 (5000毫秒)
constantTimer.setDelay("5000");

// 设置定时器是否启用
constantTimer.setEnabled(true);

// 如果需要将定时器添加到测试计划中
TestElement testPlan = new TestElement();
testPlan.addTestElement(constantTimer);
```

### 2.2 均匀随机定时器 (Uniform Random Timer)

#### 描述
在请求之间增加一个指定范围内的随机延迟。

#### 适用场景
需要模拟更现实的场景，在请求之间有可变的延迟。

#### 主要参数
- **固定延迟偏移 (Constant Delay Offset):** 最小延迟时间（毫秒）。
- **随机延迟最大值 (Random Delay Maximum):** 额外的最大随机延迟（毫秒）。

#### Java代码实现

```java
import org.apache.jmeter.timers.UniformRandomTimer;
import org.apache.jmeter.testelement.TestElement;

// 创建均匀随机定时器
UniformRandomTimer randomTimer = new UniformRandomTimer();

// 设置定时器名称
randomTimer.setName("Uniform Random Timer");

// 设置最小2秒延迟 (2000毫秒)
randomTimer.setDelay("2000");

// 设置额外的最大3秒随机延迟 (3000毫秒)
randomTimer.setRange("3000");

// 设置定时器是否启用
randomTimer.setEnabled(true);

// 如果需要将定时器添加到测试计划中
TestElement testPlan = new TestElement();
testPlan.addTestElement(randomTimer);
```

### 2.3 高斯随机定时器 (Gaussian Random Timer)

#### 描述
使用正态（高斯）分布在请求之间增加延迟，围绕一个固定的偏移量。

#### 适用场景
需要延迟遵循正态分布，以更接近某些现实世界的场景。

#### 主要参数
- **偏差 (Deviation):** 延迟的标准偏差（毫秒）。
- **固定延迟偏移 (Constant Delay Offset):** 基础延迟时间（毫秒）。

#### Java代码实现

```java
import org.apache.jmeter.timers.GaussianRandomTimer;
import org.apache.jmeter.testelement.TestElement;

// 创建高斯随机定时器
GaussianRandomTimer gaussianTimer = new GaussianRandomTimer();

// 设置定时器名称
gaussianTimer.setName("Gaussian Random Timer");

// 设置基础延迟为3秒 (3000毫秒)
gaussianTimer.setDelay("3000");

// 设置1秒的标准偏差 (1000毫秒)
gaussianTimer.setRange("1000");

// 设置定时器是否启用
gaussianTimer.setEnabled(true);

// 如果需要将定时器添加到测试计划中
TestElement testPlan = new TestElement();
testPlan.addTestElement(gaussianTimer);
```

### 2.4 恒定吞吐量定时器 (Constant Throughput Timer)

#### 描述
通过暂停线程来达到特定的吞吐量（请求数/分钟）。

#### 适用场景
需要将测试限制在一定的请求数/分钟，模拟特定的负载。

#### 主要参数
- **目标吞吐量 (Target Throughput):** 所需的吞吐量速率（每分钟的采样数）。
- **基于以下条件计算吞吐量 (Calculate Throughput Based On):** 确定是针对每个线程应用延迟还是针对所有线程。

#### Java代码实现

```java
import org.apache.jmeter.timers.ConstantThroughputTimer;
import org.apache.jmeter.testelement.TestElement;

// 创建恒定吞吐量定时器
ConstantThroughputTimer throughputTimer = new ConstantThroughputTimer();

// 设置定时器名称
throughputTimer.setName("Constant Throughput Timer");

// 设置基于所有线程计算吞吐量
throughputTimer.setCalcMode(ConstantThroughputTimer.Mode.AllActiveThreads);

// 设置每分钟100个请求的吞吐量
throughputTimer.setT
```


### 2.5 泊松随机定时器 (Poisson Random Timer)

#### 描述
使用泊松分布在请求之间增加延迟，适用于模拟随机间隔的事件但具有已知的平均速率。

#### 适用场景
模拟自然的突发流量，请求成簇地出现。这种定时器特别适合模拟真实世界中的许多随机事件，如客户到达、网络请求等。

#### 主要参数
- **λ (Lambda):** 平均间隔时间（毫秒）。这个参数决定了事件发生的平均速率。
- **固定延迟偏移 (Constant Delay Offset):** 在泊松分布计算的延迟之上添加的固定延迟（毫秒）。

#### Java代码实现

```java
import org.apache.jmeter.timers.PoissonRandomTimer;
import org.apache.jmeter.testelement.TestElement;

// 创建泊松随机定时器
PoissonRandomTimer poissonTimer = new PoissonRandomTimer();

// 设置定时器名称
poissonTimer.setName("Poisson Random Timer");

// 设置平均间隔2秒 (2000毫秒)
poissonTimer.setDelay("2000");

// 设置固定延迟偏移量为1秒 (1000毫秒)
poissonTimer.setOffset("1000");

// 设置定时器是否启用
poissonTimer.setEnabled(true);

// 如果需要将定时器添加到测试计划中
TestElement testPlan = new TestElement();
testPlan.addTestElement(poissonTimer);
```

#### 使用注意事项
1. λ (Lambda) 参数代表平均事件间隔。例如，如果设置为2000毫秒，意味着平均每2秒发生一次事件。
2. 泊松分布可能会产生一些非常短或非常长的间隔。为了避免极端情况，可以使用固定延迟偏移来确保最小延迟。
3. 这个定时器特别适合模拟不规则但有一定模式的用户行为，如网站访问、客户服务请求等。

