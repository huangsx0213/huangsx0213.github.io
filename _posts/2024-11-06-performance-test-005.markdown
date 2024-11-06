---
layout:     post
title:      "JMeter 从数据库查询不重复数据"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-06 11:00:00
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---


# JMeter 从数据库查询不重复数据


### 完整方案和代码

#### 1. JMeter 测试计划创建步骤

##### 1.1 创建测试计划
1. 打开 JMeter
2. 右键点击测试计划
3. 重命名为 "Database Performance Test"

#### 2. User Defined Variables 配置

##### 2.1 添加 User Defined Variables
1. 右键点击测试计划
2. 添加 "Config Element" > "User Defined Variables"
3. 配置以下变量：

```
# 数据库查询配置
SQL_TEMPLATE:        SELECT ? AS col1, ? AS col2 FROM ? LIMIT ?
DB_COLUMN1:          id
DB_COLUMN2:          uetr
DB_TABLE:            your_table
BATCH_SIZE:          1000

# HTTP 请求配置
HTTP_HOST:           your_api_host
HTTP_PATH:           /your_api_endpoint

# 测试配置
TEST_THREADS:        50
RAMP_UP_PERIOD:      10
LOOP_COUNT:          -1  // Forever
```

#### 3. Setup Thread Group

##### 3.1 添加 Setup Thread Group
1. 右键点击测试计划
2. 添加 "Threads (Users)" > "Setup Thread Group"
3. 配置：
   - 线程数：1
   - Ramp-up Period：1
   - 循环次数：1

##### 3.2 添加 JSR223 Sampler (Groovy)
```groovy
import groovy.sql.Sql
import java.util.concurrent.ConcurrentLinkedQueue
import java.util.concurrent.ConcurrentHashMap

// 安全获取变量的工具方法
def safeGetVariable(variableName, defaultValue = null) {
    def value = vars.get(variableName)
    return value != null ? value : defaultValue
}

// 初始化数据库连接池
def initConnectionPool() {
    if (props.get('dbConnectionPool') == null) {
        def dbUrl = System.getProperty('db.url') ?: 'jdbc:postgresql://localhost:5432/your_database'
        def dbUser = System.getProperty('db.user') ?: 'postgres'
        def dbPass = System.getProperty('db.pass') ?: 'password'
        
        def pool = Sql.newInstance(dbUrl, dbUser, dbPass, 'org.postgresql.Driver')
        props.put('dbConnectionPool', pool)
    }
}

// 初始化全局数据队列
def initGlobalDataQueue() {
    if (props.get('globalDataQueue') == null) {
        props.put('globalDataQueue', new ConcurrentLinkedQueue())
    }
}

// 初始化已使用数据集合
def initUsedDataSet() {
    if (props.get('usedDataSet') == null) {
        props.put('usedDataSet', new ConcurrentHashMap<String, Boolean>())
    }
}

// 执行初始化
initConnectionPool()
initGlobalDataQueue()
initUsedDataSet()

log.info("Initialization complete: connection pool, data queue, and used data set created.")
```

#### 4. Main Thread Group

##### 4.1 添加 Thread Group
1. 右键点击测试计划
2. 添加 "Threads (Users)" > "Thread Group"
3. 配置：
   - 线程数：`${TEST_THREADS}`
   - Ramp-up Period：`${RAMP_UP_PERIOD}`
   - 循环次数：`${LOOP_COUNT}`

##### 4.2 添加 While Controller
1. 右键点击 Thread Group
2. 添加 "Logic Controller" > "While Controller"
3. 条件：`${__jexl3(true)}`

##### 4.3 添加 JSR223 PreProcessor (Groovy)
```groovy
import groovy.sql.Sql
import java.sql.SQLException
import java.util.concurrent.ConcurrentLinkedQueue
import java.util.concurrent.ConcurrentHashMap

// 安全获取变量的工具方法
def safeGetVariable(variableName, defaultValue = null) {
    def value = vars.get(variableName)
    return value != null ? value : defaultValue
}

// 获取查询配置
def sqlTemplate = safeGetVariable('SQL_TEMPLATE', 'SELECT ? AS col1, ? AS col2 FROM ? LIMIT ?')
def dbColumn1 = safeGetVariable('DB_COLUMN1', 'id')
def dbColumn2 = safeGetVariable('DB_COLUMN2', 'uetr')
def dbTable = safeGetVariable('DB_TABLE', 'your_table')
def batchSize = safeGetVariable('BATCH_SIZE', '1000') as Integer

// 获取 HTTP 配置
def httpHost = safeGetVariable('HTTP_HOST', 'localhost')
def httpPath = safeGetVariable('HTTP_PATH', '/')

// 获取连接池
def getConnectionPool() {
    return props.get('dbConnectionPool')
}

// 获取全局数据队列
def getGlobalDataQueue() {
    return props.get('globalDataQueue')
}

// 获取已使用数据集合
def getUsedDataSet() {
    return props.get('usedDataSet')
}

// 生成数据的唯一键
def getDataKey(data) {
    return "${data[dbColumn1]}_${data[dbColumn2]}"
}

// 批量查询数据
def queryDatabase() {
    def pool = getConnectionPool()
    try {
        // 使用预编译语句防止 SQL 注入
        def startTime = System.currentTimeMillis()
        def result = pool.rows(sqlTemplate, [dbColumn1, dbColumn2, dbTable, batchSize])
        def endTime = System.currentTimeMillis()
        
        log.info("Database query completed in ${endTime - startTime} ms, retrieved ${result.size()} rows")
        return result
    } catch (SQLException e) {
        log.error("Database query failed: ${e.message}")
        log.error("Query details - Template: ${sqlTemplate}, Params: [${dbColumn1}, ${dbColumn2}, ${dbTable}, ${batchSize}]")
        return []
    }
}

// 过滤出未使用的数据
def filterUnusedData(newData) {
    def usedData = getUsedDataSet()
    def unusedData = []
    def duplicateCount = 0
    def uniqueCount = 0

    newData.each { data ->
        def key = getDataKey(data)
        if (!usedData.containsKey(key)) {
            unusedData << data
            usedData[key] = true
            uniqueCount++
        } else {
            duplicateCount++
        }
    }

    log.info("Data filtering results: Total received: ${newData.size()}, Unique: ${uniqueCount}, Duplicate: ${duplicateCount}")
    return unusedData
}

// 获取或刷新数据
def refreshData() {
    try {
        log.info("Initiating data refresh process")
        def startTime = System.currentTimeMillis()
        
        def newData = queryDatabase()
        def unusedData = filterUnusedData(newData)

        if (unusedData) {
            def queue = getGlobalDataQueue()
            queue.addAll(unusedData)
            log.info("Added ${unusedData.size()} unique items to queue")
        } else {
            log.warn("No new unique data found in database")
        }

        def endTime = System.currentTimeMillis()
        log.info("Data refresh process completed in ${endTime - startTime} ms")
    } catch (Exception e) {
        log.error("Error in data refresh process: ${e.message}")
    }
}

// 获取未使用的数据
def getUnusedData() {
    try {
        def queue = getGlobalDataQueue()
        def data = queue.poll()
        
        // 如果队列为空，尝试刷新数据
        if (data == null && queue.isEmpty()) {
            log.info("Data queue is empty, attempting to refresh")
            refreshData()
            data = queue.poll()
        }
        
        return data
    } catch (Exception e) {
        log.error("Error getting unused data: ${e.message}")
        return null
    }
}

// 主逻辑
def unusedData = getUnusedData()

if (unusedData) {
    // 记录使用的数据
    vars.put('param1', unusedData[dbColumn1])
    vars.put('param2', unusedData[dbColumn2])
    
    log.info("Using data: ${dbColumn1}=${unusedData[dbColumn1]}, ${dbColumn2}=${unusedData[dbColumn2]}")
} else {
    log.warn("No unused data available")
    // 直接返回，跳过当前迭代
    return
}
```

##### 4.4 添加 HTTP Request
1. 右键点击 While Controller
2. 添加 "Sampler" > "HTTP Request"
3. 配置：
   - Protocol: http
   - Server Name: `${HTTP_HOST}`
   - Path: `${HTTP_PATH}`
   - Parameters:
     - Name: id, Value: `${param1}`
     - Name: uetr, Value: `${param2}`

##### 4.5 添加 Response Assertion
1. 右键点击 HTTP Request
2. 添加 "Assertions" > "Response Assertion"
3. 配置：
   - 断言类型：Not Contains
   - 模式匹配规则：包含特定字符串（如 "error"）
   - 应用于：Text Response

#### 5. Teardown Thread Group

##### 5.1 添加 Teardown Thread Group
1. 右键点击测试计划
2. 添加 "Threads (Users)" > "Teardown Thread Group"
3. 配置：
   - 线程数：1
   - Ramp-up Period：1
   - 循环次数：1

##### 5.2 添加 JSR223 Sampler (Groovy)
```groovy
// 清理数据库连接池
def pool = props.get('dbConnectionPool')
if (pool) {
    pool.close()
    props.remove('dbConnectionPool')
}

// 清理全局数据队列
def queue = props.get('globalDataQueue')
queue?.clear()
props.remove('globalDataQueue')

// 清理已使用数据集合
def usedData = props.get('usedDataSet')
usedData?.clear()
props.remove('usedDataSet')

log.info("Resources cleaned up successfully")
```

#### 6. 性能测试监控（可选）

##### 6.1 添加监控元件
1. 右键点击测试计划
2. 添加 "Listener" > "View Results Tree"
3. 添加 "Listener" > "Aggregate Report"
4. 添加 "Listener" > "Summary Report"

#### 7. 运行配置

##### 7.1 命令行参数示例
```bash
jmeter -n -t your_test_plan.jmx \
  -Jdb.url=jdbc:postgresql://your_database_host:5432/your_database \
  -Jdb.user=your_username \
  -Jdb.pass=your_password
```

### 注意事项

1. 确保 PostgreSQL JDBC 驱动在 JMeter 的 `lib` 目录下
2. 根据实际环境调整数据库连接信息
3. 监控日志和性能报告
4. 根据系统负载调整线程数和批量大小

### 优势

1. 高度可配置
2. 安全的数据处理
3. 灵活的测试场景
4. 详细的日志记录
5. 资源自动清理

### 建议的优化点

1. 根据实际性能测试需求调整参数
2. 添加更多详细的日志记录
3. 考虑添加性能监控和告警机制

这个方案提供了一个全面、安全且灵活的 JMeter 性能测试解决方案，可以根据具体需求进行定制和调整。