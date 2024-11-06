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



为了实现从数据库查询数据、使用这些数据发送HTTP请求、以及在测试结束后清理资源，我们需要结合以下几个关键组件：


### 测试计划结构

- **Test Plan**
  - **User Defined Variables** (全局变量)
  - **setupThreadGroup**
    - **JSR223 Sampler** (用于初始化)
  - **Thread Group**
    - **While Controller**
      - **JSR223 PreProcessor**
      - **HTTP Request**
        - **Response Assertion**
  - **tearDown Thread Group**
    - **JSR223 Sampler** (用于清理资源)

### 用户定义变量 (User Defined Variables)

```
# 数据库查询配置
SQL_TEMPLATE: SELECT ? AS col1, ? AS col2 FROM ? LIMIT ?
DB_COLUMN1: id
DB_COLUMN2: uetr
DB_TABLE: your_table
BATCH_SIZE: 1000

# HTTP 请求配置
HTTP_HOST: your_api_host
HTTP_PATH: /your_api_endpoint
```

### 初始化脚本 (setupThreadGroup)

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

### JSR223 PreProcessor 脚本

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
        return pool.rows(sqlTemplate, [dbColumn1, dbColumn2, dbTable, batchSize])
    } catch (SQLException e) {
        log.error("Database query failed: ${e.message}")
        return []
    }
}

// 过滤出未使用的数据
def filterUnusedData(newData) {
    def usedData = getUsedDataSet()
    def unusedData = []

    newData.each { data ->
        def key = getDataKey(data)
        if (!usedData.containsKey(key)) {
            unusedData << data
            usedData[key] = true
        }
    }

    return unusedData
}

// 获取或刷新数据
def refreshData() {
    def newData = queryDatabase()
    def unusedData = filterUnusedData(newData)

    if (unusedData) {
        def queue = getGlobalDataQueue()
        queue.addAll(unusedData)
        log.info("Added ${unusedData.size()} unique items to queue")
    } else {
        log.warn("No new unique data found")
    }
}

// 获取未使用的数据
def getUnusedData() {
    def queue = getGlobalDataQueue()
    def data = queue.poll()
    
    if (data == null && queue.isEmpty()) {
        refreshData()
        data = queue.poll()
    }
    
    return data
}

// 主逻辑
def unusedData = getUnusedData()

if (unusedData) {
    vars.put('param1', unusedData[dbColumn1])
    vars.put('param2', unusedData[dbColumn2])
    log.info("Using data: ${unusedData[dbColumn1]}, ${unusedData[dbColumn2]}")
} else {
    log.warn("No unused data available")
    return  // 直接返回，跳过当前迭代
}
```

### HTTP Request 配置

- **Protocol**: http
- **Server Name or IP**: `${HTTP_HOST}`
- **Path**: `${HTTP_PATH}`
- **Parameters**:
  - Name: id, Value: `${param1}`
  - Name: uetr, Value: `${param2}`

### tearDown Thread Group 脚本

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

### 主要改进点

1. **安全的变量获取**：
   - 添加 `safeGetVariable` 方法
   - 提供默认值
   - 防止空指针异常

2. **SQL 注入防护**：
   - 使用预编译语句
   - 参数化查询

3. **灵活的配置**：
   - 通过用户变量和系统属性配置
   - 易于在不同环境间切换

4. **日志记录**：
   - 详细的日志信息
   - 错误和警告信息清晰

5. **性能优化**：
   - 使用线程安全的集合
   - 减少不必要的锁

6. **错误处理**：
   - 捕获并记录异常
   - 提供默认行为

### 使用建议

1. 根据实际环境调整数据库连接信息
2. 配置合适的批量大小
3. 监控日志输出
4. 根据性能测试需求调整线程数和循环次数

这个方案提供了一个安全、灵活、易于维护的性能测试解决方案，可以根据具体需求进行定制。