---
layout:     post
title:      "Jenkins Pipeline运行JMeter测试示例"
subtitle:   "「 Performance Testing 」" 
date:       2024-08-19 17:00:00
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---


# Jenkins Pipeline运行JMeter测试示例

以下是一个完整的 Jenkins Pipeline 示例，其中包含如何设置动态变量并使用它们来运行 JMeter 测试。这个示例将涵盖参数化构建、环境变量设置、动态变量计算，以及如何在 Pipeline 中使用这些变量。

```groovy
pipeline {
    agent any

    // 参数化构建
    parameters {
        string(name: 'BASE_THREAD_COUNT', defaultValue: '50', description: '基础线程数')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'qa', 'staging', 'prod'], description: '目标环境')
        string(name: 'TEST_DURATION', defaultValue: '300', description: '测试持续时间(秒)')
    }

    // 环境变量
    environment {
        JMETER_HOME = tool 'JMeter'
        PATH = "${env.JMETER_HOME}/bin:${env.PATH}"
        TEST_PLAN = 'performance_test_plan.jmx'
        RESULTS_FILE = 'results.jtl'
        REPORT_FOLDER = 'jmeter-report'
    }

    stages {
        stage('准备环境') {
            steps {
                // 检查 JMeter 版本
                sh 'jmeter --version'
                
                // 获取测试脚本
                git 'https://github.com/your-repo/jmeter-tests.git'
            }
        }

        stage('配置测试参数') {
            steps {
                script {
                    // 动态计算线程数
                    def threadMultiplier = (env.ENVIRONMENT == 'prod') ? 2 : 1
                    env.THREAD_COUNT = "${params.BASE_THREAD_COUNT.toInteger() * threadMultiplier}"

                    // 根据环境设置目标 URL
                    switch(env.ENVIRONMENT) {
                        case 'dev':
                            env.TARGET_URL = 'http://dev.example.com'
                            break
                        case 'qa':
                            env.TARGET_URL = 'http://qa.example.com'
                            break
                        case 'staging':
                            env.TARGET_URL = 'http://staging.example.com'
                            break
                        case 'prod':
                            env.TARGET_URL = 'http://www.example.com'
                            break
                    }

                    // 打印配置信息
                    echo "测试环境: ${env.ENVIRONMENT}"
                    echo "目标 URL: ${env.TARGET_URL}"
                    echo "线程数: ${env.THREAD_COUNT}"
                    echo "测试持续时间: ${params.TEST_DURATION} 秒"
                }
            }
        }

        stage('运行 JMeter 测试') {
            steps {
                sh """
                jmeter -n -t ${env.TEST_PLAN} \
                    -JTHREAD_COUNT=${env.THREAD_COUNT} \
                    -JTEST_DURATION=${params.TEST_DURATION} \
                    -JTARGET_URL=${env.TARGET_URL} \
                    -l ${env.RESULTS_FILE} \
                    -e -o ${env.REPORT_FOLDER}
                """
            }
        }

        stage('生成性能报告') {
            steps {
                perfReport sourceDataFiles: env.RESULTS_FILE, 
                           errorFailedThreshold: 5, 
                           errorUnstableThreshold: 3, 
                           errorUnstableResponseTimeThreshold: '1000'
            }
        }
    }

    post {
        always {
            // 归档测试结果和报告
            archiveArtifacts artifacts: "${env.RESULTS_FILE}, ${env.REPORT_FOLDER}/**/*", fingerprint: true
        }
        success {
            echo '性能测试成功完成！'
        }
        failure {
            echo '性能测试失败，请检查日志。'
        }
    }
}
```

这个 Pipeline 做了以下几件事：

1. 参数化构建：允许用户在运行时指定基础线程数、目标环境和测试持续时间。  

2. 环境变量设置：设置 JMeter 路径和一些常量。  

3. 准备环境：检查 JMeter 版本并从 Git 仓库获取测试脚本。  

4. 配置测试参数：  
   - 根据选择的环境动态计算线程数。
   - 根据环境设置目标 URL。

5. 运行 JMeter 测试：使用配置的参数执行测试。  

6. 生成性能报告：使用 Jenkins 性能插件生成报告。  

7. 后处理：归档测试结果和报告，并根据测试结果输出相应的消息。  

要使用这个 Pipeline，您需要：

1. 确保 Jenkins 安装了必要的插件：Git, Performance Plugin。  

2. 在 Jenkins 的全局工具配置中设置 JMeter。  

3. 创建一个包含 JMeter 测试计划（performance_test_plan.jmx）的 Git 仓库。  

4. 在 Jenkins 中创建一个新的 Pipeline 作业，并使用这个脚本。  

5. 确保您的 JMeter 测试计划支持通过 JVM 参数传入 THREAD_COUNT、TEST_DURATION 和 TARGET_URL。  

使用这个 Pipeline，您可以轻松地运行参数化的性能测试，并根据不同的环境自动调整测试配置。这提供了很大的灵活性，允许您在不同的场景下重复使用相同的 Pipeline。

