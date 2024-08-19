---
layout:     post
title:      "JMeter WebPerformanceTests.java"
subtitle:   "「 Performance Testing 」" 
date:       2024-08-19 17:00:00
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---


# Web性能测试工具包括主程序、配置文件和测试数据文件

1. 主程序 (WebPerformanceTests.java)

```java
import org.apache.jmeter.config.Arguments;
import org.apache.jmeter.config.CSVDataSet;
import org.apache.jmeter.control.LoopController;
import org.apache.jmeter.engine.StandardJMeterEngine;
import org.apache.jmeter.protocol.http.control.gui.HttpTestSampleGui;
import org.apache.jmeter.protocol.http.sampler.HTTPSamplerProxy;
import org.apache.jmeter.reporters.ResultCollector;
import org.apache.jmeter.reporters.Summariser;
import org.apache.jmeter.save.SaveService;
import org.apache.jmeter.testelement.TestElement;
import org.apache.jmeter.testelement.TestPlan;
import org.apache.jmeter.threads.ThreadGroup;
import org.apache.jmeter.timers.ConstantThroughputTimer;
import org.apache.jmeter.util.JMeterUtils;
import org.apache.jorphan.collections.HashTree;
import org.apache.jmeter.visualizers.SummaryReport;
import org.apache.jmeter.visualizers.ViewResultsFullVisualizer;
import org.apache.jmeter.visualizers.GraphVisualizer;
import org.apache.jmeter.visualizers.ResponseTimeGraphVisualizer;
import org.apache.jmeter.assertions.ResponseAssertion;
import org.apache.jmeter.assertions.gui.AssertionGui;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.util.Properties;
import org.apache.log4j.Logger;
import org.apache.log4j.PropertyConfigurator;

public class WebPerformanceTests {
    private static final Logger logger = Logger.getLogger(WebPerformanceTests.class);
    private static Properties config = new Properties();

    public static void main(String[] args) {
        try {
            // 加载配置
            config.load(new FileInputStream("config.properties"));
            PropertyConfigurator.configure("log4j.properties");

            // 初始化 JMeter
            JMeterUtils.loadJMeterProperties(config.getProperty("jmeter.home") + "/bin/jmeter.properties");
            JMeterUtils.initLogging();
            JMeterUtils.initLocale();

            if (args.length == 0 || "peak".equalsIgnoreCase(args[0])) {
                logger.info("Starting Peak Test");
                createTest("peak");
            }
            if (args.length == 0 || "soak".equalsIgnoreCase(args[0])) {
                logger.info("Starting Soak Test");
                createTest("soak");
            }
        } catch (Exception e) {
            logger.error("Error in test execution", e);
        }
    }

    private static void createTest(String testType) throws Exception {
        HashTree testPlanTree = new HashTree();

        TestPlan testPlan = new TestPlan("Web Application " + testType.toUpperCase() + " Test");
        testPlan.setProperty(TestElement.TEST_CLASS, TestPlan.class.getName());
        testPlan.setProperty(TestElement.GUI_CLASS, TestPlan.class.getName());
        testPlan.setUserDefinedVariables((Arguments) new Arguments().getArgument());

        ThreadGroup threadGroup = new ThreadGroup();
        threadGroup.setName(testType + " Test Thread Group");
        threadGroup.setNumThreads(Integer.parseInt(config.getProperty(testType + ".users")));
        threadGroup.setRampUp(Integer.parseInt(config.getProperty("ramp.up.period")));
        threadGroup.setScheduler(true);
        threadGroup.setDuration(Integer.parseInt(config.getProperty("test.duration")));
        threadGroup.setProperty(TestElement.TEST_CLASS, ThreadGroup.class.getName());
        threadGroup.setProperty(TestElement.GUI_CLASS, ThreadGroup.class.getName());

        LoopController loopController = new LoopController();
        loopController.setLoops(-1);
        loopController.setProperty(TestElement.TEST_CLASS, LoopController.class.getName());
        loopController.setProperty(TestElement.GUI_CLASS, LoopController.class.getName());
        threadGroup.setSamplerController(loopController);

        HTTPSamplerProxy httpSampler = new HTTPSamplerProxy();
        httpSampler.setDomain(config.getProperty("target.host"));
        httpSampler.setPort(Integer.parseInt(config.getProperty("target.port")));
        httpSampler.setPath("/");
        httpSampler.setMethod("GET");
        httpSampler.setName(testType + " Test Request");
        httpSampler.setProperty(TestElement.TEST_CLASS, HTTPSamplerProxy.class.getName());
        httpSampler.setProperty(TestElement.GUI_CLASS, HttpTestSampleGui.class.getName());

        ResponseAssertion responseAssertion = new ResponseAssertion();
        responseAssertion.setName("Response Assertion");
        responseAssertion.setProperty(TestElement.TEST_CLASS, ResponseAssertion.class.getName());
        responseAssertion.setProperty(TestElement.GUI_CLASS, AssertionGui.class.getName());
        responseAssertion.setTestFieldResponseCode();
        responseAssertion.setToEqualsInteger();
        responseAssertion.addTestString("200");

        ConstantThroughputTimer timer = new ConstantThroughputTimer();
        timer.setName("Constant Throughput Timer");
        timer.setProperty(TestElement.TEST_CLASS, ConstantThroughputTimer.class.getName());
        timer.setProperty(TestElement.GUI_CLASS, ConstantThroughputTimer.class.getName());
        timer.setCalcMode(ConstantThroughputTimer.Mode.AllActiveThreads.ordinal());
        timer.setThroughput(Double.parseDouble(config.getProperty(testType + ".throughput")) * 60);

        CSVDataSet csvDataSet = new CSVDataSet();
        csvDataSet.setName("CSV Data Set");
        csvDataSet.setProperty(TestElement.TEST_CLASS, CSVDataSet.class.getName());
        csvDataSet.setProperty(TestElement.GUI_CLASS, CSVDataSet.class.getName());
        csvDataSet.setFilename("test_data.csv");
        csvDataSet.setVariableNames("username,password");

        testPlanTree.add(testPlan);
        HashTree threadGroupHashTree = testPlanTree.add(testPlan, threadGroup);
        threadGroupHashTree.add(httpSampler);
        threadGroupHashTree.add(timer);
        threadGroupHashTree.add(responseAssertion);
        threadGroupHashTree.add(csvDataSet);

        // 添加性能监控和统计
        addListeners(testPlanTree, testType);

        SaveService.saveTree(testPlanTree, new FileOutputStream(config.getProperty("test.plan." + testType)));

        // 运行测试
        StandardJMeterEngine jmeter = new StandardJMeterEngine();
        jmeter.configure(testPlanTree);
        jmeter.run();
    }

    private static void addListeners(HashTree testPlanTree, String testType) {
        SummaryReport summaryReport = new SummaryReport();
        summaryReport.setName("Summary Report");
        summaryReport.setProperty(TestElement.TEST_CLASS, SummaryReport.class.getName());
        summaryReport.setProperty(TestElement.GUI_CLASS, SummaryReport.class.getName());

        ViewResultsFullVisualizer resultsTree = new ViewResultsFullVisualizer();
        resultsTree.setName("View Results Tree");
        resultsTree.setProperty(TestElement.TEST_CLASS, ViewResultsFullVisualizer.class.getName());
        resultsTree.setProperty(TestElement.GUI_CLASS, ViewResultsFullVisualizer.class.getName());

        GraphVisualizer graphVisualizer = new GraphVisualizer();
        graphVisualizer.setName("Graph Results");
        graphVisualizer.setProperty(TestElement.TEST_CLASS, GraphVisualizer.class.getName());
        graphVisualizer.setProperty(TestElement.GUI_CLASS, GraphVisualizer.class.getName());

        ResponseTimeGraphVisualizer responseTimeGraphVisualizer = new ResponseTimeGraphVisualizer();
        responseTimeGraphVisualizer.setName("Response Time Graph");
        responseTimeGraphVisualizer.setProperty(TestElement.TEST_CLASS, ResponseTimeGraphVisualizer.class.getName());
        responseTimeGraphVisualizer.setProperty(TestElement.GUI_CLASS, ResponseTimeGraphVisualizer.class.getName());

        testPlanTree.add(testPlanTree.getArray()[0], summaryReport);
        testPlanTree.add(testPlanTree.getArray()[0], resultsTree);
        testPlanTree.add(testPlanTree.getArray()[0], graphVisualizer);
        testPlanTree.add(testPlanTree.getArray()[0], responseTimeGraphVisualizer);

        Summariser summer = null;
        String summariserName = JMeterUtils.getPropDefault("summariser.name", "summary");
        if (summariserName.length() > 0) {
            summer = new Summariser(summariserName);
        }
        ResultCollector logger = new ResultCollector(summer);
        logger.setFilename(config.getProperty("result.file." + testType));
        testPlanTree.add(testPlanTree.getArray()[0], logger);
    }
}
```

2. 配置文件 (config.properties)

```properties
# JMeter配置
jmeter.home=/path/to/apache-jmeter

# 目标网站配置
target.host=example.com
target.port=80

# 测试持续时间（秒）
test.duration=3600

# 线程组加载时间（秒）
ramp.up.period=300

# 峰值测试配置
peak.users=1000
peak.throughput=100

# 持久测试配置
soak.users=500
soak.throughput=50

# 结果文件路径
result.file.peak=/path/to/peak_test_results.jtl
result.file.soak=/path/to/soak_test_results.jtl

# 测试计划文件路径
test.plan.peak=/path/to/peak_test_plan.jmx
test.plan.soak=/path/to/soak_test_plan.jmx
```

3. 日志配置文件 (log4j.properties)

```properties
# 根日志记录器配置
log4j.rootLogger=INFO, file, console

# 控制台输出配置
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

# 文件输出配置
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.File=web_performance_tests.log
log4j.appender.file.MaxFileSize=10MB
log4j.appender.file.MaxBackupIndex=10
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
```

4. 测试数据文件 (test_data.csv)

```csv
username,password
user1,pass1
user2,pass2
user3,pass3
user4,pass4
user5,pass5
```

详细说明：

1. 主程序 (WebPerformanceTests.java)
   - 这个程序使用Apache JMeter API创建和执行Web性能测试。
   - 它支持两种测试类型：峰值测试（Peak Test）和持久测试（Soak Test）。
   - 主要组件包括：
     - TestPlan：整个测试的顶层容器
     - ThreadGroup：定义并发用户数和测试持续时间
     - HTTPSamplerProxy：模拟HTTP请求
     - ConstantThroughputTimer：控制请求的吞吐量
     - ResponseAssertion：验证响应的正确性
     - CSVDataSet：从CSV文件读取测试数据
     - 各种监听器：收集和展示测试结果

2. 配置文件 (config.properties)
   - 包含所有可配置的参数，如目标网站、并发用户数、吞吐量等。
   - 分别为峰值测试和持久测试设置了不同的参数。
   - 指定了结果文件和测试计划文件的保存路径。

3. 日志配置文件 (log4j.properties)
   - 配置了日志输出格式和目标（控制台和文件）。
   - 设置了日志文件的大小限制和备份策略。

4. 测试数据文件 (test_data.csv)
   - 包含模拟用户的登录信息。
   - 在测试中用于模拟不同用户的请求。

使用说明：

1. 确保已安装Apache JMeter和所有必要的依赖项。
2. 将主程序、配置文件和测试数据文件放在同一目录下。
3. 根据实际情况修改config.properties文件中的配置。
4. 编译Java程序：
   ```
   javac -cp .:path/to/jmeter/lib/* WebPerformanceTests.java
   ```
5. 运行程序：
   - 执行峰值测试：
     ```
     java -cp .:path/to/jmeter/lib/* WebPerformanceTests peak
     ```
   - 执行持久测试：
     ```
     java -cp .:path/to/jmeter/lib/* WebPerformanceTests soak
     ```
   - 执行两种测试：
     ```
     java -cp .:path/to/jmeter/lib/* WebPerformanceTests
     ```

6. 测试完成后，查看生成的JTL文件获取详细结果，或使用JMeter GUI打开生成的JMX文件查看图形化报告。

