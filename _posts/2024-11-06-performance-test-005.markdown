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


这个优化版本使用了 LRU 缓存来管理已使用的数据，并改进了数据获取和重试机制。

### 完整方案

#### 1. 测试计划结构
- **Test Plan**
  - **User Defined Variables**
  - **setupThreadGroup**
    - **JSR223 Sampler** (初始化)
  - **Thread Group**
    - **While Controller**
      - **JSR223 PreProcessor**
      - **HTTP Request**
      - **Response Assertion**
  - **tearDown Thread Group**
    - **JSR223 Sampler** (清理)

#### 2. User Defined Variables 配置
```
# 数据库查询配置
SQL_TEMPLATE:           SELECT ? AS col1, ? AS col2 FROM ? LIMIT ?
DB_COLUMN1:            id
DB_COLUMN2:            uetr
DB_TABLE:              your_table
BATCH_SIZE:            1000
USED_DATA_CACHE_SIZE:  10000

# HTTP 配置
HTTP_HOST:             your_api_host
HTTP_PATH:             /your_api_endpoint

# 测试配置
TEST_THREADS:          50
RAMP_UP_PERIOD:        10
MAX_RETRY_ATTEMPTS:    5
RETRY_DELAY:           1000
```

#### 3. Setup Thread Group

##### 3.1 JSR223 Sampler (初始化)
```groovy
import groovy.sql.Sql
import java.util.concurrent.ConcurrentLinkedQueue
import java.util.concurrent.locks.ReentrantLock
import java.util.LinkedHashMap

// LRU 缓存实现
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries

    public LRUCache(int maxEntries) {
        super(maxEntries + 1, 1.0f, true)
        this.maxEntries = maxEntries
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries
    }
}

// 初始化连接池
def initConnectionPool() {
    if (props.get('dbConnectionPool') == null) {
        def dbUrl = System.getProperty('db.url') ?: 'jdbc:postgresql://localhost:5432/your_database'
        def dbUser = System.getProperty('db.user') ?: 'postgres'
        def dbPass = System.getProperty('db.pass') ?: 'password'
        
        def pool = Sql.newInstance(dbUrl, dbUser, dbPass, 'org.postgresql.Driver')
        props.put('dbConnectionPool', pool)
        log.info("Database connection pool initialized")
    }
}

// 初始化全局数据队列
def initGlobalDataQueue() {
    if (props.get('globalDataQueue') == null) {
        props.put('globalDataQueue', new ConcurrentLinkedQueue())
        log.info("Global data queue initialized")
    }
}

// 初始化 LRU 缓存
def initUsedDataCache() {
    if (props.get('usedDataCache') == null) {
        def cacheSize = vars.get('USED_DATA_CACHE_SIZE') as Integer ?: 10000
        props.put('usedDataCache', new LRUCache<String, Boolean>(cacheSize))
        log.info("Used data cache initialized with size: ${cacheSize}")
    }
}

// 初始化全局锁
if (props.get('globalLock') == null) {
    props.put('globalLock', new ReentrantLock())
    log.info("Global lock initialized")
}

// 执行所有初始化
initConnectionPool()
initGlobalDataQueue()
initUsedDataCache()

log.info("Setup completed successfully")
```

#### 4. Thread Group - While Controller

##### 4.1 While Controller 配置
- **Condition**: `${__jexl3(vars.get('stopTest') != 'true')}`

##### 4.2 JSR223 PreProcessor
```groovy
import groovy.sql.Sql
import java.sql.SQLException
import java.util.concurrent.ConcurrentLinkedQueue
import java.util.LinkedHashMap

// 安全获取变量的工具方法
def safeGetVariable(variableName, defaultValue = null) {
    def value = vars.get(variableName)
    return value != null ? value : defaultValue
}

// 获取配置
def sqlTemplate = safeGetVariable('SQL_TEMPLATE')
def dbColumn1 = safeGetVariable('DB_COLUMN1')
def dbColumn2 = safeGetVariable('DB_COLUMN2')
def dbTable = safeGetVariable('DB_TABLE')
def batchSize = safeGetVariable('BATCH_SIZE') as Integer
def maxAttempts = safeGetVariable('MAX_RETRY_ATTEMPTS', '5') as Integer
def retryDelay = safeGetVariable('RETRY_DELAY', '1000') as Long

// 获取全局资源
def globalLock = props.get('globalLock')
def pool = props.get('dbConnectionPool')
def queue = props.get('globalDataQueue')
def usedDataCache = props.get('usedDataCache')

// 生成数据的唯一键
def getDataKey(data) {
    return "${data[dbColumn1]}_${data[dbColumn2]}"
}

// 合并的方法：查询、过滤和添加数据
def queryFilterAndAddData() {
    globalLock.lock()
    try {
        log.info("Starting queryFilterAndAddData process")
        def startTime = System.currentTimeMillis()

        // 查询数据库
        def result
        try {
            result = pool.rows(sqlTemplate, [dbColumn1, dbColumn2, dbTable, batchSize])
            log.info("Database query completed, retrieved ${result.size()} rows")
        } catch (SQLException e) {
            log.error("Database query failed: ${e.message}")
            return false
        }

        // 过滤和添加数据
        def addedCount = 0
        result.each { data ->
            def key = getDataKey(data)
            if (!usedDataCache.containsKey(key)) {
                queue.offer(data)
                usedDataCache.put(key, true)
                addedCount++
            }
        }

        def endTime = System.currentTimeMillis()
        log.info("queryFilterAndAddData completed in ${endTime - startTime} ms. Added ${addedCount} new items")
        return addedCount > 0
    } finally {
        globalLock.unlock()
    }
}

// 获取未使用的数据
def getUnusedData() {
    def attempts = 0
    
    while (attempts < maxAttempts) {
        globalLock.lock()
        try {
            def data = queue.poll()
            if (data != null) {
                return data
            }
            
            log.info("Data queue is empty, attempting to refresh (Attempt ${attempts + 1}/${maxAttempts})")
            def refreshSuccess = queryFilterAndAddData()
            
            if (refreshSuccess) {
                data = queue.poll()
                if (data != null) {
                    return data
                }
            }
            
            attempts++
        } finally {
            globalLock.unlock()
        }
        
        if (attempts < maxAttempts) {
            Thread.sleep(retryDelay)
        }
    }
    
    log.error("Failed to get unused data after ${maxAttempts} attempts")
    return null
}

// 主逻辑
def unusedData = getUnusedData()

if (unusedData) {
    vars.put('param1', unusedData[dbColumn1])
    vars.put('param2', unusedData[dbColumn2])
    log.info("Using data: ${dbColumn1}=${unusedData[dbColumn1]}, ${dbColumn2}=${unusedData[dbColumn2]}")
} else {
    log.warn("No unused data available after multiple attempts")
    vars.put('stopTest', 'true')
    return
}
```

##### 4.3 HTTP Request
- **Protocol**: http/https
- **Server Name**: `${HTTP_HOST}`
- **Path**: `${HTTP_PATH}`
- **Parameters**:
  - Name: id, Value: `${param1}`
  - Name: uetr, Value: `${param2}`

#### 5. Teardown Thread Group

##### 5.1 JSR223 Sampler (清理)
```groovy
// 清理数据库连接池
def pool = props.get('dbConnectionPool')
if (pool) {
    try {
        pool.close()
        log.info("Database connection pool closed")
    } catch (Exception e) {
        log.error("Error closing database pool: ${e.message}")
    }
    props.remove('dbConnectionPool')
}

// 清理全局数据队列
def queue = props.get('globalDataQueue')
if (queue) {
    queue.clear()
    props.remove('globalDataQueue')
    log.info("Global data queue cleared")
}

// 清理 LRU 缓存
def cache = props.get('usedDataCache')
if (cache) {
    cache.clear()
    props.remove('usedDataCache')
    log.info("Used data cache cleared")
}

// 移除全局锁
props.remove('globalLock')

log.info("All resources cleaned up successfully")
```

### 主要改进点

1. **LRU 缓存实现**：
   - 使用 LinkedHashMap 实现 LRU 缓存
   - 自动移除最老的数据
   - 可配置缓存大小

2. **重试机制**：
   - 可配置最大重试次数
   - 可配置重试延迟
   - 详细的日志记录

3. **资源管理**：
   - 使用全局锁确保线程安全
   - 统一的资源初始化和清理
   - 更好的错误处理

4. **性能优化**：
   - 减少数据库查询频率
   - 优化数据缓存策略
   - 更高效的数据结构

### 使用说明

1. **配置参数**：
   - 在 User Defined Variables 中设置所有必要参数
   - 可通过命令行覆盖数据库连接信息

2. **运行测试**：
```bash
jmeter -n -t test_plan.jmx \
  -Jdb.url=jdbc:postgresql://your_host:5432/your_db \
  -Jdb.user=your_user \
  -Jdb.pass=your_password
```

3. **监控**：
   - 观察日志输出了解测试进展
   - 监控数据库连接和资源使用情况

这个优化版本提供了更好的性能和可靠性，同时保持了代码的可维护性和灵活性。