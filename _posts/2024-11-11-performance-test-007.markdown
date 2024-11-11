---
layout:     post
title:      "JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-11 11:00:01
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---

# JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现

## 一、引言

本测试方案旨在解决在 JMeter 高并发测试场景中的关键问题：如何从数据库中获取不同类型的测试数据作为 HTTP 请求的参数，并确保在多线程并发执行时，这些参数不会被重复使用。

### 主要特点
1. **多数据池支持**：支持创建多个独立的数据池，满足不同测试场景需求
2. **数据唯一性保证**：通过并发安全的数据结构确保测试数据不重复使用
3. **高性能数据获取**：使用数据库连接池和批量查询优化性能
4. **动态数据补充**：自动化的数据补充机制，确保测试数据持续可用
5. **资源智能管理**：完整的资源初始化和清理机制
6. **状态实时监控**：提供多数据池的使用情况实时监控

### 适用场景
- 需要多种不同类型测试数据的接口测试
- 高并发性能测试
- 基于数据库数据的自动化测试
- 需要确保参数不重复的测试场景

## 二、测试计划结构

```
Test Plan
├── User Defined Variables (配置参数)
├── setUp Thread Group (初始化数据池)
│   └── JSR223 Sampler (数据池初始化脚本)
├── Thread Group (转账测试)
│   ├── JSR223 PreProcessor (获取转账数据)
│   └── HTTP Request (转账请求)
├── Thread Group (支付测试)
│   ├── JSR223 PreProcessor (获取支付数据)
│   └── HTTP Request (支付请求)
└── tearDown Thread Group
    └── JSR223 Sampler (清理资源)
```

## 三、完整代码实现

### 1. User Defined Variables 配置

```properties
# 数据库基础配置
DB_URL: jdbc:postgresql://localhost:5432/testdb
DB_USER: postgres
DB_PASSWORD: password

# 转账数据池配置
TRANSFER_POOL_QUERY: SELECT * FROM transfer WHERE status='PENDING' LIMIT ?
TRANSFER_POOL_UPDATE_SQL: UPDATE transfer SET status = ? WHERE reference_no = ?
TRANSFER_POOL_BATCH_SIZE: 1000
TRANSFER_POOL_LOW_WATER_MARK: 100
TRANSFER_POOL_CLEAN_UP_THRESHOLD: 1000

# 支付数据池配置
PAYMENT_POOL_QUERY: SELECT * FROM payment WHERE status='INIT' LIMIT ?
PAYMENT_POOL_UPDATE_SQL: UPDATE payment SET status = ? WHERE reference_no = ?
PAYMENT_POOL_BATCH_SIZE: 500
PAYMENT_POOL_LOW_WATER_MARK: 50
PAYMENT_POOL_CLEAN_UP_THRESHOLD: 500
```

### 2. 数据池配置类

```groovy
class DataPoolConfig implements Serializable {
    final String poolName
    final String dbUrl
    final String dbUser
    final String dbPassword
    final String querySQL
    final String updateSQL
    final int batchSize
    final int lowWaterMark
    final int cleanUpThreshold
    final double cleanUpRatio = 0.3
    
    DataPoolConfig(String name, Properties props) {
        this.poolName = name
        this.dbUrl = props.getProperty("DB_URL")
        this.dbUser = props.getProperty("DB_USER")
        this.dbPassword = props.getProperty("DB_PASSWORD")
        this.querySQL = props.getProperty("${name}_QUERY")
        this.updateSQL = props.getProperty("${name}_UPDATE_SQL")
        this.batchSize = Integer.parseInt(props.getProperty("${name}_BATCH_SIZE", "1000"))
        this.lowWaterMark = Integer.parseInt(props.getProperty("${name}_LOW_WATER_MARK", "100"))
        this.cleanUpThreshold = Integer.parseInt(props.getProperty("${name}_CLEAN_UP_THRESHOLD", "1000"))
    }
}
```

### 3. 测试数据模型

```groovy
class TestData implements Serializable {
    private String id
    private String referenceNo
    
    TestData(String id, String referenceNo) {
        this.id = id
        this.referenceNo = referenceNo
    }
    
    String getId() { return id }
    String getReferenceNo() { return referenceNo }
}
```

### 4. 数据池管理器

```groovy
class DataPoolManager {
    private static final Logger log = LoggerFactory.getLogger(DataPoolManager.class)
    
    private final String poolName
    private final ConcurrentLinkedQueue<TestData> dataQueue
    private final ConcurrentHashMap<String, Boolean> usedData
    private final AtomicInteger processedCount
    private final AtomicInteger updateCount
    private final AtomicBoolean isRefilling
    private final ReentrantLock refillLock
    private final BasicDataSource dataSource
    private final DataPoolConfig config
    
    DataPoolManager(DataPoolConfig config) {
        this.poolName = config.poolName
        this.config = config
        this.dataQueue = new ConcurrentLinkedQueue<>()
        this.usedData = new ConcurrentHashMap<>()
        this.processedCount = new AtomicInteger(0)
        this.updateCount = new AtomicInteger(0)
        this.isRefilling = new AtomicBoolean(false)
        this.refillLock = new ReentrantLock()
        this.dataSource = setupDataSource()
        
        initialFill()
    }
    
    private BasicDataSource setupDataSource() {
        BasicDataSource ds = new BasicDataSource()
        ds.setUrl(config.dbUrl)
        ds.setUsername(config.dbUser)
        ds.setPassword(config.dbPassword)
        ds.setInitialSize(5)
        ds.setMaxTotal(20)
        ds.setMaxWaitMillis(10000)
        ds.setValidationQuery("SELECT 1")
        return ds
    }
    
    TestData getNextData() {
        TestData data = dataQueue.poll()
        
        if (data != null) {
            if (usedData.putIfAbsent(data.getId(), Boolean.TRUE) != null) {
                return getNextData()
            }
            processedCount.incrementAndGet()
            
            if (dataQueue.size() < config.lowWaterMark) {
                triggerRefill()
            }
            
            return data
        }
        
        refillDataPool()
        return dataQueue.poll()
    }
    
    private void triggerRefill() {
        if (isRefilling.compareAndSet(false, true)) {
            Thread.start {
                try {
                    refillDataPool()
                } finally {
                    isRefilling.set(false)
                }
            }
        }
    }
    
    private void refillDataPool() {
        if (refillLock.tryLock()) {
            try {
                if (usedData.size() > config.cleanUpThreshold) {
                    cleanUp()
                }

                Connection conn = dataSource.getConnection()
                try {
                    PreparedStatement stmt = conn.prepareStatement(config.querySQL)
                    stmt.setInt(1, config.batchSize)
                    ResultSet rs = stmt.executeQuery()
                    
                    int addedCount = 0
                    while (rs.next()) {
                        TestData data = new TestData(
                            rs.getString("id"),
                            rs.getString("reference_no")
                        )
                        if (!usedData.containsKey(data.getId())) {
                            dataQueue.offer(data)
                            addedCount++
                        }
                    }
                    
                    log.info("${poolName} - Refilled pool. Added: ${addedCount}, Current size: ${dataQueue.size()}")
                } finally {
                    conn.close()
                }
            } catch (Exception e) {
                log.error("${poolName} - Database error during refill", e)
            } finally {
                refillLock.unlock()
            }
        }
    }
    
    void updateStatus(String referenceNo, String status) {
        Connection conn = null
        PreparedStatement stmt = null
        try {
            conn = dataSource.getConnection()
            stmt = conn.prepareStatement(config.updateSQL)
            stmt.setString(1, status)
            stmt.setString(2, referenceNo)
            int updated = stmt.executeUpdate()
            
            if (updated > 0) {
                updateCount.incrementAndGet()
                log.debug("${poolName} - Updated status to ${status} for reference_no: ${referenceNo}")
            } else {
                log.warn("${poolName} - No record found to update for reference_no: ${referenceNo}")
            }
        } catch (SQLException e) {
            log.error("${poolName} - Error updating status for reference_no: ${referenceNo}", e)
            throw e
        } finally {
            try {
                if (stmt != null) stmt.close()
                if (conn != null) conn.close()
            } catch (SQLException e) {
                log.error("${poolName} - Error closing resources", e)
            }
        }
    }
    
    private void cleanUp() {
        int itemsToRemove = (int) (usedData.size() * config.cleanUpRatio)
        log.info("${poolName} - Cleaning up ${itemsToRemove} items from used data")
        
        Iterator<String> it = usedData.keySet().iterator()
        int removed = 0
        while (it.hasNext() && removed < itemsToRemove) {
            it.next()
            it.remove()
            removed++
        }
        
        log.info("${poolName} - Cleanup completed. Removed: ${removed}, Current size: ${usedData.size()}")
    }
    
    void shutdown() {
        try {
            dataSource.close()
            log.info("${poolName} - Data source closed")
        } catch (Exception e) {
            log.error("${poolName} - Error closing data source", e)
        }
    }
    
    String getStatusReport() {
        return String.format(
            "%s Status [Processed: %d, Available: %d, Updated: %d, Used: %d]",
            poolName,
            processedCount.get(),
            dataQueue.size(),
            updateCount.get(),
            usedData.size()
        )
    }
}
```

### 5. 数据池工厂

```groovy
class DataPoolFactory {
    private static final ConcurrentHashMap<String, DataPoolManager> pools = new ConcurrentHashMap<>()
    private static final Logger log = LoggerFactory.getLogger(DataPoolFactory.class)
    
    static DataPoolManager getOrCreate(String poolName, Properties config) {
        return pools.computeIfAbsent(poolName, { name ->
            log.info("Creating new data pool: " + name)
            DataPoolConfig poolConfig = new DataPoolConfig(name, config)
            return new DataPoolManager(poolConfig)
        })
    }
    
    static void shutdown() {
        pools.each { name, pool -> 
            try {
                log.info("Shutting down pool: " + name)
                pool.shutdown()
            } catch (Exception e) {
                log.error("Error shutting down pool: " + name, e)
            }
        }
        pools.clear()
    }

    static String getPoolsStatus() {
        StringBuilder status = new StringBuilder("Data Pools Status:\n")
        pools.each { name, pool ->
            status.append(pool.getStatusReport()).append("\n")
        }
        return status.toString()
    }
}
```

### 6. 测试计划实现

#### 6.1 setUp Thread Group - JSR223 Sampler

```groovy
import java.util.concurrent.*
import java.util.concurrent.atomic.*
import java.util.concurrent.locks.*
import java.sql.*
import org.apache.commons.dbcp2.*
import org.slf4j.Logger
import org.slf4j.LoggerFactory

// 数据池配置类
class DataPoolConfig implements Serializable {
    final String poolName
    final String dbUrl
    final String dbUser
    final String dbPassword
    final String querySQL
    final String updateSQL
    final int batchSize
    final int lowWaterMark
    final int cleanUpThreshold
    final double cleanUpRatio = 0.3
    
    DataPoolConfig(String name, Properties props) {
        this.poolName = name
        this.dbUrl = props.getProperty("DB_URL")
        this.dbUser = props.getProperty("DB_USER")
        this.dbPassword = props.getProperty("DB_PASSWORD")
        this.querySQL = props.getProperty("${name}_QUERY")
        this.updateSQL = props.getProperty("${name}_UPDATE_SQL")
        this.batchSize = Integer.parseInt(props.getProperty("${name}_BATCH_SIZE", "1000"))
        this.lowWaterMark = Integer.parseInt(props.getProperty("${name}_LOW_WATER_MARK", "100"))
        this.cleanUpThreshold = Integer.parseInt(props.getProperty("${name}_CLEAN_UP_THRESHOLD", "1000"))
    }
}

// 测试数据模型
class TestData implements Serializable {
    private String id
    private String referenceNo
    
    TestData(String id, String referenceNo) {
        this.id = id
        this.referenceNo = referenceNo
    }
    
    String getId() { return id }
    String getReferenceNo() { return referenceNo }
}

// 数据池管理器
class DataPoolManager {
    private static final Logger log = LoggerFactory.getLogger(DataPoolManager.class)
    
    private final String poolName
    private final ConcurrentLinkedQueue<TestData> dataQueue
    private final ConcurrentHashMap<String, Boolean> usedData
    private final AtomicInteger processedCount
    private final AtomicInteger updateCount
    private final AtomicBoolean isRefilling
    private final ReentrantLock refillLock
    private final BasicDataSource dataSource
    private final DataPoolConfig config
    
    DataPoolManager(DataPoolConfig config) {
        this.poolName = config.poolName
        this.config = config
        this.dataQueue = new ConcurrentLinkedQueue<>()
        this.usedData = new ConcurrentHashMap<>()
        this.processedCount = new AtomicInteger(0)
        this.updateCount = new AtomicInteger(0)
        this.isRefilling = new AtomicBoolean(false)
        this.refillLock = new ReentrantLock()
        this.dataSource = setupDataSource()
        
        initialFill()
    }
    
    private BasicDataSource setupDataSource() {
        BasicDataSource ds = new BasicDataSource()
        ds.setUrl(config.dbUrl)
        ds.setUsername(config.dbUser)
        ds.setPassword(config.dbPassword)
        ds.setInitialSize(5)
        ds.setMaxTotal(20)
        ds.setMaxWaitMillis(10000)
        ds.setValidationQuery("SELECT 1")
        return ds
    }
    
    TestData getNextData() {
        TestData data = dataQueue.poll()
        
        if (data != null) {
            if (usedData.putIfAbsent(data.getId(), Boolean.TRUE) != null) {
                return getNextData()
            }
            processedCount.incrementAndGet()
            
            if (dataQueue.size() < config.lowWaterMark) {
                triggerRefill()
            }
            
            return data
        }
        
        refillDataPool()
        return dataQueue.poll()
    }
    
    private void triggerRefill() {
        if (isRefilling.compareAndSet(false, true)) {
            Thread.start {
                try {
                    refillDataPool()
                } finally {
                    isRefilling.set(false)
                }
            }
        }
    }
    
    private void refillDataPool() {
        if (refillLock.tryLock()) {
            try {
                if (usedData.size() > config.cleanUpThreshold) {
                    cleanUp()
                }

                Connection conn = dataSource.getConnection()
                try {
                    PreparedStatement stmt = conn.prepareStatement(config.querySQL)
                    stmt.setInt(1, config.batchSize)
                    ResultSet rs = stmt.executeQuery()
                    
                    int addedCount = 0
                    while (rs.next()) {
                        TestData data = new TestData(
                            rs.getString("id"),
                            rs.getString("reference_no")
                        )
                        if (!usedData.containsKey(data.getId())) {
                            dataQueue.offer(data)
                            addedCount++
                        }
                    }
                    
                    log.info("${poolName} - Refilled pool. Added: ${addedCount}, Current size: ${dataQueue.size()}")
                } finally {
                    conn.close()
                }
            } catch (Exception e) {
                log.error("${poolName} - Database error during refill", e)
            } finally {
                refillLock.unlock()
            }
        }
    }

    private void initialFill() {
        refillDataPool()
    }
    
    void updateStatus(String referenceNo, String status) {
        Connection conn = null
        PreparedStatement stmt = null
        try {
            conn = dataSource.getConnection()
            stmt = conn.prepareStatement(config.updateSQL)
            stmt.setString(1, status)
            stmt.setString(2, referenceNo)
            int updated = stmt.executeUpdate()
            
            if (updated > 0) {
                updateCount.incrementAndGet()
                log.debug("${poolName} - Updated status to ${status} for reference_no: ${referenceNo}")
            } else {
                log.warn("${poolName} - No record found to update for reference_no: ${referenceNo}")
            }
        } catch (SQLException e) {
            log.error("${poolName} - Error updating status for reference_no: ${referenceNo}", e)
            throw e
        } finally {
            try {
                if (stmt != null) stmt.close()
                if (conn != null) conn.close()
            } catch (SQLException e) {
                log.error("${poolName} - Error closing resources", e)
            }
        }
    }
    
    private void cleanUp() {
        int itemsToRemove = (int) (usedData.size() * config.cleanUpRatio)
        log.info("${poolName} - Cleaning up ${itemsToRemove} items from used data")
        
        Iterator<String> it = usedData.keySet().iterator()
        int removed = 0
        while (it.hasNext() && removed < itemsToRemove) {
            it.next()
            it.remove()
            removed++
        }
        
        log.info("${poolName} - Cleanup completed. Removed: ${removed}, Current size: ${usedData.size()}")
    }
    
    void shutdown() {
        try {
            dataSource.close()
            log.info("${poolName} - Data source closed")
        } catch (Exception e) {
            log.error("${poolName} - Error closing data source", e)
        }
    }
    
    String getStatusReport() {
        return String.format(
            "%s Status [Processed: %d, Available: %d, Updated: %d, Used: %d]",
            poolName,
            processedCount.get(),
            dataQueue.size(),
            updateCount.get(),
            usedData.size()
        )
    }
}

// 数据池工厂
class DataPoolFactory {
    private static final ConcurrentHashMap<String, DataPoolManager> pools = new ConcurrentHashMap<>()
    private static final Logger log = LoggerFactory.getLogger(DataPoolFactory.class)
    
    static DataPoolManager getOrCreate(String poolName, Properties config) {
        return pools.computeIfAbsent(poolName, { name ->
            log.info("Creating new data pool: " + name)
            DataPoolConfig poolConfig = new DataPoolConfig(name, config)
            return new DataPoolManager(poolConfig)
        })
    }
    
    static void shutdown() {
        pools.each { name, pool -> 
            try {
                log.info("Shutting down pool: " + name)
                pool.shutdown()
            } catch (Exception e) {
                log.error("Error shutting down pool: " + name, e)
            }
        }
        pools.clear()
    }

    static String getPoolsStatus() {
        StringBuilder status = new StringBuilder("Data Pools Status:\n")
        pools.each { name, pool ->
            status.append(pool.getStatusReport()).append("\n")
        }
        return status.toString()
    }
}

// 主程序：初始化配置和数据池
Properties config = new Properties()
vars.entrySet().each { entry ->
    config.put(entry.key, entry.value)
}

// 初始化数据池
def transferPool = DataPoolFactory.getOrCreate("TRANSFER_POOL", config)
def paymentPool = DataPoolFactory.getOrCreate("PAYMENT_POOL", config)

// 存储到全局属性
props.put("transferPool", transferPool)
props.put("paymentPool", paymentPool)

log.info("Data pools initialized:\n" + DataPoolFactory.getPoolsStatus())
```

#### 6.2 转账测试线程组 - JSR223 PreProcessor

```groovy
def pool = props.get("transferPool")
def data = pool.getNextData()

if (data != null) {
    vars.put("transferId", data.getId())
    vars.put("transferRef", data.getReferenceNo())
    
    // 更新状态为处理中
    pool.updateStatus(data.getReferenceNo(), "PROCESSING")
    
    if (ctx.getThreadNum() % 100 == 0) {
        log.info(pool.getStatusReport())
    }
} else {
    log.error("No transfer data available")
    SampleResult.setSuccessful(false)
}
```

#### 6.3 支付测试线程组 - JSR223 PreProcessor

```groovy
def pool = props.get("paymentPool")
def data = pool.getNextData()

if (data != null) {
    vars.put("paymentId", data.getId())
    vars.put("paymentRef", data.getReferenceNo())
    
    // 更新状态为处理中
    pool.updateStatus(data.getReferenceNo(), "PROCESSING")
    
    if (ctx.getThreadNum() % 100 == 0) {
        log.info(pool.getStatusReport())
    }
} else {
    log.error("No payment data available")
    SampleResult.setSuccessful(false)
}
```

#### 6.4 tearDown Thread Group - JSR223 Sampler

```groovy
log.info("Final status:\n" + DataPoolFactory.getPoolsStatus())
DataPoolFactory.shutdown()
log.info("All data pools have been shut down")
```

## 四、使用说明

### 1. 配置说明
- 在 User Defined Variables 中设置所有必要的配置参数
- 为每个数据池配置独立的查询和更新语句
- 根据需要调整批量大小和低水位标记

### 2. 运行准备
- 确保数据库连接正常
- 检查数据表中有足够的测试数据
- 设置适当的线程数和循环次数

### 3. 监控要点
- 观察数据池状态报告
- 监控数据库连接情况
- 检查日志输出

## 五、注意事项

### 1. 性能优化
- 适当调整批量查询大小
- 监控数据库负载
- 优化连接池参数

### 2. 异常处理
- 数据库连接异常
- 数据不足情况
- 并发访问问题

### 3. 资源管理
- 确保资源正确释放
- 监控内存使用
- 避免资源泄露

此方案提供了完整的多数据池支持，确保了：
1. 数据的线程安全访问
2. 高效的数据获取和更新
3. 资源的合理管理
4. 完整的状态监控

适用于复杂的高并发测试场景，特别是需要多种不同类型测试数据的情况。