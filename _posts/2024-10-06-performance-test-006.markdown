---
layout:     post
title:      "JMeter 从数据库查询不重复数据2"
subtitle:   "「 Performance Testing 」" 
date:       2024-10-06 11:00:02
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---


# JMeter 从数据库查询不重复数据2



好的,我们将修改方案以移除 JDBC Connection Configuration,并将 JDBC Request 替换为 HTTP Request。以下是更新后的完整方案和代码:

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

### 详细配置

#### a) User Defined Variables (全局变量)

在 JMeter 的全局变量中配置以下信息：

- **DB URL**: `${__P(db.url,jdbc:postgresql://localhost:5432/your_database)}`
- **DB User**: `${__P(db.user,postgres)}`
- **DB Password**: `${__P(db.pass,password)}`
- **SQL Query**: `SELECT ${dbColumn1}, ${dbColumn2} FROM your_table LIMIT ${batchSize}`
- **dbColumn1**: `id`
- **dbColumn2**: `uetr`
- **batchSize**: `1000`
- **HTTP_HOST**: `your_api_host`
- **HTTP_PATH**: `/your_api_endpoint`

#### b) setupThreadGroup

- **Name**: setupThreadGroup
- **Number of Threads**: 1
- **Ramp-up Period**: 1
- **Loop Count**: 1

#### c) JSR223 Sampler (用于初始化,在 setupThreadGroup 中)

```groovy
import groovy.sql.Sql
import java.sql.SQLException
import java.util.concurrent.ConcurrentLinkedQueue

// 初始化连接池
def initConnectionPool() {
    if (props.get('dbConnectionPool') == null) {
        def dbUrl = props.get('db.url')
        def dbUser = props.get('db.user')
        def dbPass = props.get('db.pass')
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

// 执行初始化
initConnectionPool()
initGlobalDataQueue()

log.info("Initialized database connection pool and global data queue.")
```

#### d) Thread Group

- **Name**: API Users
- **Number of Threads**: 50
- **Ramp-up Period**: 10 seconds
- **Loop Count**: 100 或 Forever

#### e) While Controller

- **Condition**: `${__jexl3("${CONTINUE}" != "false")}`

#### f) JSR223 PreProcessor (使用 Groovy)

```groovy
import groovy.sql.Sql
import java.sql.SQLException
import java.util.concurrent.ConcurrentLinkedQueue
import java.util.concurrent.locks.ReentrantLock

// 显式锁
def lock = new ReentrantLock()

// 获取连接池
def getConnectionPool() {
    return props.get('dbConnectionPool')
}

// 获取全局数据队列
def getGlobalDataQueue() {
    return props.get('globalDataQueue')
}

// 批量查询数据
def queryDatabase() {
    def pool = getConnectionPool()
    try {
        def dbColumn1 = props.get('dbColumn1')
        def dbColumn2 = props.get('dbColumn2')
        def batchSize = props.get('batchSize')
        def query = "SELECT ${dbColumn1}, ${dbColumn2} FROM your_table LIMIT ${batchSize}"
        return pool.rows(query)
    } catch (SQLException e) {
        log.warn("Database query failed: " + e.getMessage())
        return []
    }
}

// 获取或刷新数据
def refreshData() {
    try {
        if (lock.tryLock(10, TimeUnit.SECONDS)) {
            try {
                log.info("Querying database for new data.")
                def newData = queryDatabase()
                def queue = getGlobalDataQueue()
                queue.addAll(newData)
                log.info("Added ${newData.size()} new items to the global queue.")
            } finally {
                lock.unlock()
            }
        } else {
            log.warn("Failed to acquire lock for refreshing data.")
        }
    } catch (InterruptedException e) {
        log.warn("Interrupted while trying to acquire lock: " + e.getMessage())
    }
}

// 获取未使用的数据
def getUnusedData() {
    try {
        if (lock.tryLock(10, TimeUnit.SECONDS)) {
            try {
                def queue = getGlobalDataQueue()
                def data = queue.poll()
                if (data == null && queue.isEmpty()) {
                    refreshData()
                    data = queue.poll()
                }
                return data
            } finally {
                lock.unlock()
            }
        } else {
            log.warn("Failed to acquire lock for getting unused data.")
            return null
        }
    } catch (InterruptedException e) {
        log.warn("Interrupted while trying to acquire lock: " + e.getMessage())
        return null
    }
}

// 主逻辑
def unusedData = getUnusedData()

if (unusedData != null) {
    vars.put('param1', unusedData[props.get('dbColumn1')])
    vars.put('param2', unusedData[props.get('dbColumn2')])
    log.info("Using data: ${unusedData[props.get('dbColumn1')]}, ${unusedData[props.get('dbColumn2')]}")
} else {
    log.warn("Failed to get unused data. Skipping this iteration.")
    // 使用 JMeter 的 continue 功能跳过当前循环
    SampleResult.setIgnore()
    return
}
```

#### g) HTTP Request

- 右击线程组，选择 **Add** > **Sampler** > **HTTP Request**
  - **Protocol**: http (或 https，根据您的 API 需求)
  - **Server Name or IP**: `${HTTP_HOST}`
  - **HTTP Request**: GET (或根据您的 API 需求选择适当的方法)
  - **Path**: `${HTTP_PATH}`
  - **Parameters**:
    - Name: id, Value: `${param1}`
    - Name: uetr, Value: `${param2}`

#### h) Response Assertion

- 右击 HTTP Request，选择 **Add** > **Assertions** > **Response Assertion**
  - **Assertion Type**: Response Assertion
  - **Pattern Matching Rules**: Not Contains (或根据需求选择 Contains)
  - **Pattern to Test**: 要检查不包含的字符串（如 "error"）
  - **Apply to**: Text Response

#### i) tearDown Thread Group

- **Name**: tearDown Thread Group
- **Number of Threads**: 1
- **Ramp-up Period**: 1
- **Loop Count**: 1

#### j) JSR223 Sampler (用于清理资源,在 tearDown Thread Group 中)

```groovy
// 清理连接池和全局数据
def pool = props.get('dbConnectionPool')
if (pool) {
    pool.close()
    props.remove('dbConnectionPool')
}

// 清理全局数据队列
def queue = props.get('globalDataQueue')
if (queue) {
    queue.clear()
    props.remove('globalDataQueue')
}

log.info("Cleaned up database connection pool and global data queue.")
```

### 工作流程

1. **初始化** (setupThreadGroup):
   - 初始化数据库连接池和全局数据队列。

2. **执行测试** (Thread Group):
   - While Controller 确保测试持续运行直到没有更多可用数据。
   - JSR223 PreProcessor 从数据库获取未使用的数据并设置为变量。
   - HTTP Request 使用这些变量发送 API 请求。
   - Response Assertion 检查 API 响应。

3. **清理资源** (tearDown Thread Group):
   - 关闭数据库连接池，清空全局数据队列。

### 优化点

#### 线程安全
- 在主测试线程组中使用 `ReentrantLock` 和超时机制来确保线程安全。

#### 数据刷新
- 自动刷新数据以避免数据耗尽。

#### 资源清理
- 在测试结束后通过 `tearDown Thread Group` 清理资源，确保资源释放。

#### 可配置性
- 将 SQL 查询、字段名称和 HTTP 请求参数设置为全局变量，更方便进行修改和维护。

#### 简化初始化
- 移除了 setupThreadGroup 中的锁，简化了初始化过程。

这个更新后的方案将数据库查询的结果用作 HTTP 请求的参数，允许您对外部 API 进行性能测试，同时仍然使用数据库作为数据源。您可以根据实际的 API 需求调整 HTTP Request 的配置。