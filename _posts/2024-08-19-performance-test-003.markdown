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
#  JMeter线程组,定时器使用说明
## **1. 线程组 (Thread Groups)**

线程组定义了JMeter将要模拟的线程（用户）数量以及这些线程执行的计划，是JMeter测试计划的核心要素。

#### **a. 标准线程组 (Standard Thread Group)**
- **描述：** 这是JMeter中的默认线程组，用于基本的性能测试场景。它允许配置线程数、加速时间和循环次数。
- **适用场景：** 基本的负载测试，用于模拟一定数量的用户同时访问系统。
- **主要参数：**
  - **线程数 (Number of Threads):** 要模拟的虚拟用户数。
  - **加速时间 (Ramp-Up Period):** 启动所有线程所需的时间（秒）。
  - **循环次数 (Loop Count):** 测试循环的次数。
  - **调度器 (Scheduler):** 启用调度器以设置测试的开始和结束时间。
  - **持续时间 (Duration):** 测试运行的时间（秒）。
  - **启动延迟 (Startup Delay):** 线程启动前的延迟（秒）。
- **Java API 实现：**
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

#### **b. 终极线程组 (Ultimate Thread Group)**
- **描述：** 终极线程组允许对线程的加速和减速进行精确控制，支持创建复杂的负载模式。
- **适用场景：** 复杂的负载测试场景，需要以受控方式增加和减少用户数。
- **主要参数：**
  - **启动线程数 (Start Threads Count):** 在特定时间启动的线程数。
  - **初始延迟 (Initial Delay):** 启动线程前的延迟时间（秒）。
  - **启动时间 (Start Time):** 启动线程的时间（秒）。
  - **保持负载时间 (Hold Load For):** 保持负载前的时间（秒）。
  - **停止时间 (Stop Time):** 停止线程的时间（秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.threads.UltimateThreadGroup;

  UltimateThreadGroup ultimateTG = new UltimateThreadGroup();
  ultimateTG.setNumThreads(50);  // 设置线程数
  ultimateTG.setRampUp(10);      // 设置加速时间（秒）
  ultimateTG.setHold(300);       // 设置保持负载时间（秒）
  ```

#### **c. 逐步线程组 (Stepping Thread Group)**
- **描述：** 逐步线程组允许分步骤启动和停止线程，支持增量负载测试。
- **适用场景：** 逐步增加和减少用户负载的负载测试。
- **主要参数：**
  - **初始线程数 (Start Thread Count):** 初始启动的线程数。
  - **初始延迟 (Initial Delay):** 启动线程前的延迟时间（秒）。
  - **新增线程数 (Add Thread Count):** 每个步骤新增的线程数。
  - **下一批线程启动时间 (Start Next Threads After):** 每个步骤新增线程的时间间隔（秒）。
  - **保持负载时间 (Hold Load For):** 保持负载前的时间（秒）。
  - **停止线程数 (Stop Threads Count):** 每个步骤停止的线程数。
  - **下一批线程停止时间 (Stop Next Threads After):** 每个步骤停止线程的时间间隔（秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.threads.SteppingThreadGroup;

  SteppingThreadGroup steppingTG = new SteppingThreadGroup();
  steppingTG.setNumThreads(100);  // 设置线程数
  steppingTG.setRampUp(60);       // 设置加速时间（秒）
  steppingTG.setHold(600);        // 设置保持负载时间（秒）
  ```

#### **d. 到达线程组 (Arrivals Thread Group)**
- **描述：** 到达线程组用于以每秒到达请求数的形式生成负载，可以控制每秒产生的线程数。
- **适用场景：** 需要在特定时间段内产生一定数量的请求，模拟持续且均匀的负载。
- **主要参数：**
  - **到达率 (Target Rate):** 每秒产生的线程数（用户到达率）。
  - **运行时间 (Hold):** 保持此到达率的时间（秒）。
  - **启动延迟 (Startup Delay):** 线程启动前的延迟（秒）。
  - **最大线程数 (Max Threads):** 最大允许的线程数。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.threads.ArrivalsThreadGroup;

  ArrivalsThreadGroup arrivalsTG = new ArrivalsThreadGroup();
  arrivalsTG.setName("Arrivals Thread Group");
  arrivalsTG.setTargetRate(10);     // 设置每秒到达的请求数
  arrivalsTG.setHold(600);          // 设置保持负载时间（秒）
  arrivalsTG.setDelay(5);           // 设置启动延迟（秒）
  arrivalsTG.setMaxThreads(100);    // 设置最大线程数
  ```

#### **e. 自由形式到达线程组 (Free-Form Arrivals Thread Group)**
- **描述：** 自由形式到达线程组允许定义更加复杂和灵活的负载模式，例如动态到达率的负载测试。
- **适用场景：** 需要根据不同的时间段灵活控制负载强度的场景。
- **主要参数：**
  - **计划 (Schedule):** 自定义的到达率和持续时间的列表。
  - **最大线程数 (Max Threads):** 最大允许的线程数。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.threads.FreeFormArrivalsThreadGroup;

  FreeFormArrivalsThreadGroup freeFormTG = new FreeFormArrivalsThreadGroup();
  freeFormTG.setName("Free-Form Arrivals Thread Group");
  freeFormTG.addSchedule(10, 60);  // 每秒10个请求，持续60秒
  freeFormTG.addSchedule(20, 120); // 每秒20个请求，持续120秒
  freeFormTG.setMaxThreads(200);   // 设置最大线程数
  ```

#### **f. 并发线程组 (Concurrency Thread Group)**
- **描述：** 并发线程组允许模拟并发用户的负载，提供精确控制并发数和保持时间。
- **适用场景：** 需要模拟一定数量的并发用户，并控制保持时间的场景。
- **主要参数：**
  - **并发数 (Target Concurrency):** 目标并发用户数。
  - **保持时间 (Hold):** 保持此并发数的时间（秒）。
  - **加速时间 (Ramp-Up Period):** 达到目标并发数所需的时间（秒）。
  - **启动延迟 (Startup Delay):** 线程启动前的延迟（秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.threads.ConcurrencyThreadGroup;

  ConcurrencyThreadGroup concurrencyTG = new ConcurrencyThreadGroup();
  concurrencyTG.setName("Concurrency Thread Group");
  concurrencyTG.setTargetConcurrency(100); // 设置并发用户数
  concurrencyTG.setHold(1200);              // 设置保持负载时间（秒）
  concurrencyTG.setRampUp(60);              // 设置加速时间（秒）
  concurrencyTG.setDelay(10);               // 设置启动延迟（秒）
  ```

## **2. 定时器 (Timers)**

定时器用于在请求之间定义延迟，这是模拟现实用户行为的关键。

#### **a. 常量定时器 (Constant Timer)**
- **描述：** 在每个请求前增加固定的延迟。
- **适用场景：** 需要在请求之间有固定延迟以模拟稳定的用户行为。
- **主要参数：**
  - **线程延迟 (Thread Delay):** 以毫秒为单位的延迟时间。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.timers.ConstantTimer;

  ConstantTimer constantTimer = new ConstantTimer();
  constantTimer.setDelay("5000");  // 设置5秒延迟
  ```

#### **b. 均匀随机定时器 (Uniform Random Timer)**
- **描述：** 在请求之间增加一个指定范围内的随机延迟。
- **适用场景：** 需要模拟更现实的场景，在请求之间有可变的延迟。
- **主要参数：**


  - **固定延迟偏移 (Constant Delay Offset):** 最小延迟时间（毫秒）。
  - **随机延迟最大值 (Random Delay Maximum):** 额外的最大随机延迟（毫秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.timers.UniformRandomTimer;

  UniformRandomTimer randomTimer = new UniformRandomTimer();
  randomTimer.setDelay("2000");  // 设置最小2秒延迟
  randomTimer.setRange("3000");  // 设置额外的最大3秒随机延迟
  ```

#### **c. 高斯随机定时器 (Gaussian Random Timer)**
- **描述：** 使用正态（高斯）分布在请求之间增加延迟，围绕一个固定的偏移量。
- **适用场景：** 需要延迟遵循正态分布，以更接近某些现实世界的场景。
- **主要参数：**
  - **偏差 (Deviation):** 延迟的标准偏差（毫秒）。
  - **固定延迟偏移 (Constant Delay Offset):** 基础延迟时间（毫秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.timers.GaussianRandomTimer;

  GaussianRandomTimer gaussianTimer = new GaussianRandomTimer();
  gaussianTimer.setDelay("3000");  // 设置基础延迟
  gaussianTimer.setDeviation("1000");  // 设置1秒的标准偏差
  ```

#### **d. 恒定吞吐量定时器 (Constant Throughput Timer)**
- **描述：** 通过暂停线程来达到特定的吞吐量（请求数/分钟）。
- **适用场景：** 需要将测试限制在一定的请求数/分钟，模拟特定的负载。
- **主要参数：**
  - **目标吞吐量 (Target Throughput):** 所需的吞吐量速率（每分钟的采样数）。
  - **基于以下条件计算吞吐量 (Calculate Throughput Based On):** 确定是针对每个线程应用延迟还是针对所有线程。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.timers.ConstantThroughputTimer;

  ConstantThroughputTimer throughputTimer = new ConstantThroughputTimer();
  throughputTimer.setCalcMode(ConstantThroughputTimer.BYALLTHREADS);  // 基于所有线程
  throughputTimer.setThroughput(100);  // 设置每分钟100个请求
  ```

#### **e. 泊松随机定时器 (Poisson Random Timer)**
- **描述：** 使用泊松分布在请求之间增加延迟，适用于模拟随机间隔的事件但具有已知的平均速率。
- **适用场景：** 模拟自然的突发流量，请求成簇地出现。
- **主要参数：**
  - **λ (Lambda):** 平均间隔时间（毫秒）。
- **Java API 实现：**
  ```java
  import org.apache.jmeter.timers.PoissonRandomTimer;

  PoissonRandomTimer poissonTimer = new PoissonRandomTimer();
  poissonTimer.setDelay("2000");  // 设置平均间隔2秒
  ```

### **总结**
- **线程组：** 控制测试中的用户数量和执行计划。
  - *标准线程组* 适用于简单场景。
  - *终极线程组* 适用于复杂的负载控制。
  - *逐步线程组* 适用于增量负载测试。
  - *到达线程组* 适用于模拟稳定的用户到达率。
  - *自由形式到达线程组* 适用于复杂负载模式。
  - *并发线程组* 适用于并发用户模拟。
  
- **定时器：** 控制线程组中的请求时间间隔。
  - *常量定时器* 适用于固定延迟。
  - *均匀随机定时器* 适用于范围内的可变延迟。
  - *高斯随机定时器* 适用于遵循正态分布的延迟。
  - *恒定吞吐量定时器* 适用于限制请求速率。
  - *泊松随机定时器* 适用于遵循泊松分布的延迟。