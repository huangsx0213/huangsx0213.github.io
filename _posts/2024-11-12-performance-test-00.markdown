---
layout:     post
title:      "JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现（动态数据模型篇）"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-12 11:00:01
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---



# JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现 (上篇)

## 一、方案概述

本测试方案旨在解决JMeter高并发测试场景中的关键问题:如何从数据库中获取不同类型的测试数据作为HTTP请求的参数,并确保在多线程并发执行时,这些参数不会被重复使用。

### 主要特点

1. 多数据池支持:支持创建多个独立的数据池,满足不同测试场景需求
2. 数据唯一性保证:通过并发安全的数据结构确保测试数据不重复使用
3. 高性能数据获取:使用数据库连接池和批量查询优化性能
4. 动态数据补充:自动化的数据补充机制,确保测试数据持续可用
5. 资源智能管理:完整的资源初始化和清理机制
6. 状态实时监控:提供多数据池的使用情况实时监控
7. 动态字段配置:支持通过配置文件动态定义数据字段和类型
8. 灵活的唯一键定义:支持单一键和复合键,可通过配置更改

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
TRANSFER_POOL_QUERY: SELECT id, reference_no, amount, currency, status FROM transfer WHERE status='PENDING' LIMIT ?
TRANSFER_POOL_UPDATE_SQL: UPDATE transfer SET status = ? WHERE reference_no = ?
TRANSFER_POOL_BATCH_SIZE: 1000
TRANSFER_POOL_LOW_WATER_MARK: 100
TRANSFER_POOL_CLEAN_UP_THRESHOLD: 1000
TRANSFER_POOL_FIELDS: id:String,reference_no:String,amount:BigDecimal,currency:String,status:String
TRANSFER_POOL_KEY_FIELDS: id,reference_no

# 支付数据池配置
PAYMENT_POOL_QUERY: SELECT id, reference_no, payment_method, amount, status FROM payment WHERE status='INIT' LIMIT ?
PAYMENT_POOL_UPDATE_SQL: UPDATE payment SET status = ? WHERE reference_no = ?
PAYMENT_POOL_BATCH_SIZE: 500
PAYMENT_POOL_LOW_WATER_MARK: 50
PAYMENT_POOL_CLEAN_UP_THRESHOLD: 500
PAYMENT_POOL_FIELDS: id:String,reference_no:String,payment_method:String,amount:BigDecimal,status:String
PAYMENT_POOL_KEY_FIELDS: id,reference_no
```

### 2. setUp Thread Group - JSR223 Sampler

```groovy
import java.util.concurrent.*
import java.util.concurrent.atomic.*
import java.util.concurrent.locks.*
import java.sql.*
import org.apache.commons.dbcp2.*
import org.slf4j.Logger
import org.slf4j.LoggerFactory

// FieldDefinition class
class FieldDefinition implements Serializable {
    String name
    Class<?> type
    
    FieldDefinition(String name, Class<?> type) {
        this.name = name
        this.type = type
    }
    
    Object convertValue(String value) {
        if (value == null) return null
        
        switch (type) {
            case String.class:
                return value
            case Integer.class:
                return Integer.parseInt(value)
            case Long.class:
                return Long.parseLong(value)
            case BigDecimal.class:
                return new BigDecimal(value)
            case Boolean.class:
                return Boolean.parseBoolean(value)
            case Date.class:
                return Date.parse("yyyy-MM-dd HH:mm:ss", value)
            default:
                throw new IllegalArgumentException("Unsupported type: ${type}")
        }
    }
}

// DataPoolConfig class
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
    final List<FieldDefinition> fieldDefinitions
    final List<String> keyFields
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
        
        // Parse field definitions
        this.fieldDefinitions = parseFieldDefinitions(props.getProperty("${name}_FIELDS"))
        
        // Read key fields from configuration
        String keyFieldsStr = props.getProperty("${name}_KEY_FIELDS", "id,reference_no")
        this.keyFields = Arrays.asList(keyFieldsStr.split(","))
        
        // Validate key fields
        validateKeyFields()
    }
    
    private List<FieldDefinition> parseFieldDefinitions(String fieldsStr) {
        return fieldsStr.split(",").collect { fieldDef ->
            def (name, type) = fieldDef.split(":")
            new FieldDefinition(name, getClassForType(type))
        }
    }
    
    private Class<?> getClassForType(String type) {
        switch (type.toLowerCase()) {
            case "string": return String.class
            case "integer": return Integer.class
            case "long": return Long.class
            case "bigdecimal": return BigDecimal.class
            case "boolean": return Boolean.class
            case "date": return Date.class
            default: throw new IllegalArgumentException("Unsupported type: ${type}")
        }
    }
    
    private void validateKeyFields() {
        Set<String> definedFields = fieldDefinitions.collect { it.name }.toSet()
        for (String keyField : keyFields) {
            if (!definedFields.contains(keyField)) {
                throw new IllegalArgumentException(
                    "Key field '${keyField}' not found in field definitions for pool '${poolName}'"
                )
            }
        }
    }
}

// DynamicTestData class
class DynamicTestData implements Serializable {
    private Map<String, Object> fields
    private List<String> keyFields
    
    DynamicTestData(Map<String, Object> fields, List<String> keyFields) {
        this.fields = new HashMap<>(fields)
        this.keyFields = keyFields
    }
    
    String getUniqueKey() {
        return keyFields.collect { fieldName ->
            Object value = fields.get(fieldName)
            if (value == null) {
                throw new IllegalStateException(
                    "Key field '${fieldName}' has null value"
                )
            }
            return value.toString()
        }.join('|')
    }
    
    Object get(String fieldName) {
        if (!fields.containsKey(fieldName)) {
            throw new IllegalArgumentException(
                "Field '${fieldName}' not found in data record"
            )
        }
        return fields.get(fieldName)
    }
    
    Map<String, Object> getAllFields() {
        return new HashMap<>(fields)
    }
    
    @Override
    String toString() {
        StringBuilder sb = new StringBuilder("TestData{")
        // Display key fields first
        keyFields.each { field ->
            sb.append("${field}=${fields.get(field)}, ")
        }
        // Then display other fields
        fields.each { field, value ->
            if (!keyFields.contains(field)) {
                sb.append("${field}=${value}, ")
            }
        }
        sb.delete(sb.length() - 2, sb.length())
        sb.append("}")
        return sb.toString()
    }
}
```




# JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现 (下篇)

## 三、完整代码实现 (续)

### 2. setUp Thread Group - JSR223 Sampler (续)

```groovy
// DataPoolManager class
class DataPoolManager {
    private static final Logger log = LoggerFactory.getLogger(DataPoolManager.class)
    
    private final String poolName
    private final ConcurrentLinkedQueue<DynamicTestData> dataQueue
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
    
    DynamicTestData getNextData() {
        DynamicTestData data = dataQueue.poll()
        
        if (data != null) {
            if (usedData.putIfAbsent(data.getUniqueKey(), Boolean.TRUE) != null) {
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
                        DynamicTestData data = createDataFromResultSet(rs)
                        if (!usedData.containsKey(data.getUniqueKey())) {
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

    private DynamicTestData createDataFromResultSet(ResultSet rs) {
        Map<String, Object> fields = new HashMap<>()
        config.fieldDefinitions.each { fieldDef ->
            Object value = rs.getObject(fieldDef.name)
            fields.put(fieldDef.name, value)
        }
        return new DynamicTestData(fields, config.keyFields)
    }

    private void initialFill() {
        refillDataPool()
    }
    
    void updateStatus(DynamicTestData data, String status) {
        Connection conn = null
        PreparedStatement stmt = null
        try {
            conn = dataSource.getConnection()
            stmt = conn.prepareStatement(config.updateSQL)
            stmt.setString(1, status)
            stmt.setString(2, data.get("reference_no"))
            int updated = stmt.executeUpdate()
            
            if (updated > 0) {
                updateCount.incrementAndGet()
                log.debug("${poolName} - Updated status to ${status} for record: ${data}")
            } else {
                log.warn("${poolName} - No record found to update for: ${data}")
            }
        } catch (SQLException e) {
            log.error("${poolName} - Error updating status for: ${data}", e)
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

// DataPoolFactory class
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

// Main program: Initialize configuration and data pools
Properties config = new Properties()
vars.entrySet().each { entry ->
    config.put(entry.key, entry.value)
}

// Initialize data pools
def transferPool = DataPoolFactory.getOrCreate("TRANSFER_POOL", config)
def paymentPool = DataPoolFactory.getOrCreate("PAYMENT_POOL", config)

// Store in global properties
props.put("transferPool", transferPool)
props.put("paymentPool", paymentPool)
// 将 DataPoolFactory 类存储为 JMeter 属性
props.put("DataPoolFactory", DataPoolFactory)

log.info("Data pools initialized:\n" + DataPoolFactory.getPoolsStatus())
```

### 3. 转账测试线程组 - JSR223 PreProcessor

```groovy
def pool = props.get("transferPool")
def data = pool.getNextData()

if (data != null) {
    // 获取字段值
    vars.put("id", data.get("id"))
    vars.put("referenceNo", data.get("reference_no"))
    vars.put("amount", data.get("amount").toString())
    vars.put("currency", data.get("currency"))
    
    // 更新状态
    pool.updateStatus(data, "PROCESSING")
    
    if (ctx.getThreadNum() % 100 == 0) {
        log.info("Transfer data: " + data.toString())
        log.info(pool.getStatusReport())
    }
} else {
    log.error("No transfer data available")
    SampleResult.setSuccessful(false)
}
```

### 4. 支付测试线程组 - JSR223 PreProcessor

```groovy
def pool = props.get("paymentPool")
def data = pool.getNextData()

if (data != null) {
    // 获取字段值
    vars.put("id", data.get("id"))
    vars.put("referenceNo", data.get("reference_no"))
    vars.put("paymentMethod", data.get("payment_method"))
    vars.put("amount", data.get("amount").toString())
    
    // 更新状态
    pool.updateStatus(data, "PROCESSING")
    
    if (ctx.getThreadNum() % 100 == 0) {
        log.info("Payment data: " + data.toString())
        log.info(pool.getStatusReport())
    }
} else {
    log.error("No payment data available")
    SampleResult.setSuccessful(false)
}
```

### 5. tearDown Thread Group - JSR223 Sampler

```groovy
// 从 JMeter 属性中获取 DataPoolFactory
def DataPoolFactory = props.get("DataPoolFactory")

if (DataPoolFactory) {
    log.info("Final status:\n" + DataPoolFactory.getPoolsStatus())
    DataPoolFactory.shutdown()
    log.info("All data pools have been shut down")
} else {
    log.error("DataPoolFactory not found in properties")
}
```

## 四、使用说明

1. 在JMeter中创建测试计划,按照上述结构设置线程组和取样器。

2. 在User Defined Variables中设置所有必要的配置参数。

3. 将setUp Thread Group中的JSR223 Sampler代码复制到相应位置。

4. 在转账和支付测试线程组中分别添加JSR223 PreProcessor,并复制相应代码。

5. 在tearDown Thread Group中添加JSR223 Sampler,复制清理代码。

6. 根据实际需求设置HTTP请求取样器,使用变量引用从PreProcessor中获取的数据。

7. 设置适当的线程数和循环次数。

8. 运行测试并监控结果。

## 五、注意事项

1. 确保数据库连接正常,并有足够的测试数据。

2. 适当调整批量查询大小、低水位标记等参数以优化性能。

3. 监控数据库负载,必要时优化SQL查询。

4. 注意异常处理,特别是数据库连接异常和数据不足情况。

5. 定期检查日志输出,关注数据池状态报告。

6. 测试结束后确保资源正确释放。

7. 根据实际测试需求调整字段定义和唯一键配置。

8. 考虑使用JMeter的后端监听器来收集和分析测试结果。

9. 在大规模测试前,先进行小规模测试以验证配置的正确性。

10. 定期维护和更新测试数据,确保数据的有效性和多样性。

这个方案提供了一个灵活、高效且线程安全的参数化测试实现,适用于复杂的高并发测试场景。它支持多数据池、动态字段配置和灵活的唯一键定义,可以满足各种测试需求。通过合理配置和使用,可以有效进行大规模并发测试,同时确保数据的唯一性和完整性。
