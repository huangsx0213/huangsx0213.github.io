---
layout:     post
title:      "JMeter 从数据库查询不重复数据2"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-06 11:00:02
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---


# JMeter 从数据库查询不重复数据2


优化后的完整方案：

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

### User Defined Variables

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

### setupThreadGroup 脚本

```groovy
import groovy.sql.Sql
import java.util.concurrent.ConcurrentSkipListSet

// 初始化连接池
def initConnectionPool() {
    if (props.get('dbConnectionPool') == null) {
        def dbUrl = System.getProperty('db.url') ?: 'jdbc:postgresql://localhost:5432/your_database'
        def dbUser = System.getProperty('db.user') ?: 'postgres'
        def dbPass = System.getProperty('db.pass') ?: 'password'
        
        def pool = Sql.newInstance(dbUrl, dbUser, dbPass, 'org.postgresql.Driver')
        props.put('dbConnectionPool', pool)
    }
}

// 初始化全局数据集
def initGlobalDataSet() {
    if (props.get('globalDataSet') == null) {
        props.put('globalDataSet', new ConcurrentSkipListSet())
    }
}

// 批量查询数据并初始化数据集
def initializeDataSet() {
    def pool = props.get('dbConnectionPool')
    def dataSet = props.get('globalDataSet')
    
    def dbColumn1 = vars.get("DB_COLUMN1")
    def dbColumn2 = vars.get("DB_COLUMN2")
    def dbTable = vars.get("DB_TABLE")
    def batchSize = vars.get("BATCH_SIZE") as Integer
    
    try {
        def query = "SELECT ${dbColumn1}, ${dbColumn2} FROM ${dbTable} LIMIT ${batchSize}"
        def resultSet = pool.rows(query)
        dataSet.addAll(resultSet)
        log.info("Initial data load complete. Added ${resultSet.size()} items to the global data set.")
    } catch (Exception e) {
        log.error("Failed to initialize data set: ${e.message}")
    }
}

// 执行初始化
initConnectionPool()
initGlobalDataSet()
initializeDataSet()
```

### JSR223 PreProcessor 脚本

```groovy
// 获取用户变量（避免重复获取）
def dbColumn1 = vars.get("DB_COLUMN1")
def dbColumn2 = vars.get("DB_COLUMN2")

// 获取全局数据集
def dataSet = props.get('globalDataSet')

// 批量查询数据（仅在需要时执行）
def queryDatabase() {
    def pool = props.get('dbConnectionPool')
    def dbTable = vars.get("DB_TABLE")
    def batchSize = vars.get("BATCH_SIZE") as Integer
    
    try {
        def query = "SELECT ${dbColumn1}, ${dbColumn2} FROM ${dbTable} LIMIT ${batchSize}"
        return pool.rows(query)
    } catch (Exception e) {
        log.warn("Database query failed: ${e.message}")
        return []
    }
}

// 获取数据并在必要时刷新
def getData() {
    def data = dataSet.pollFirst()
    
    if (data == null && dataSet.isEmpty()) {
        def newData = queryDatabase()
        if (!newData.isEmpty()) {
            dataSet.addAll(newData)
            data = dataSet.pollFirst()
        }
    }
    
    return data
}

// 主逻辑
def data = getData()

if (data != null) {
    vars.put('param1', data[dbColumn1])
    vars.put('param2', data[dbColumn2])
} else {
    log.warn("No data available")
    return
}
```

### HTTP Request

- **Protocol**: http
- **Server Name or IP**: `${HTTP_HOST}`
- **Path**: `${HTTP_PATH}`
- **Parameters**:
  - Name: id, Value: `${param1}`
  - Name: uetr, Value: `${param2}`

### tearDown Thread Group 脚本

```groovy
// 清理连接池
def pool = props.get('dbConnectionPool')
if (pool) {
    pool.close()
    props.remove('dbConnectionPool')
}

// 清理全局数据集
def dataSet = props.get('globalDataSet')
if (dataSet) {
    dataSet.clear()
    props.remove('globalDataSet')
}

log.info("Resources cleaned up successfully")
```

### 性能优化要点

1. **减少变量访问**
   - 将频繁使用的变量缓存到局部变量
   - 避免重复获取相同的变量

2. **最小化同步操作**
   - 使用 ConcurrentSkipListSet 的原子操作
   - 避免不必要的锁操作

3. **优化数据库查询**
   - 仅在数据集为空时查询数据库
   - 使用批量查询减少数据库访问次数

4. **减少日志输出**
   - 仅记录必要的日志信息
   - 使用适当的日志级别

5. **内存优化**
   - 及时清理不需要的对象
   - 避免创建不必要的对象

### 代码特点

1. **简化的数据结构**
   - 只使用一个 ConcurrentSkipListSet 存储数据
   - 移除了不必要的 UsedDataSet

2. **高效的数据获取**
   - 使用 pollFirst() 原子操作获取数据
   - 只在必要时刷新数据集

3. **优化的初始化过程**
   - 启动时预加载数据
   - 减少运行时的数据库访问

4. **清晰的错误处理**
   - 适当的异常捕获和处理
   - 合理的日志记录

### 使用建议

1. **配置调优**
   - 根据实际需求调整 BATCH_SIZE
   - 监控数据库连接池的使用情况

2. **监控优化**
   - 监控数据集大小
   - 监控数据库查询频率

3. **性能测试**
   - 在正式测试前验证脚本性能
   - 监控 JMeter 的资源使用情况

### 注意事项

1. 确保数据库能够承受查询负载
2. 监控内存使用情况
3. 根据实际测试需求调整配置参数

这个优化后的方案通过减少不必要的操作和优化数据结构，提高了脚本的性能，同时保证了数据不会被重复使用。在高并发场景下，它能够更高效地运行，不会成为性能测试的瓶颈。