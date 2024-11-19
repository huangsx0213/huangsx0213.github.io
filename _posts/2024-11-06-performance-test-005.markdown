---
layout:     post
title:      "JMeter 高并发测试方案：多数据池线程安全参数化测试实现"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-06 11:00:01
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---



# JMeter 高并发测试方案：多数据池线程安全参数化测试实现

---

## 一、方案概述

### 1.1 目标
本方案旨在解决JMeter高并发测试中的数据参数化问题，设计并实现一个支持多数据池、线程安全、高效的数据管理系统，确保高并发测试中的数据唯一性和安全性。

### 1.2 主要功能
1. **多数据池管理**：支持独立数据池的创建和管理，适用于不同的测试场景。
2. **数据唯一性**：确保每个测试线程使用的数据不会被重复使用。
3. **自动补充机制**：当数据池的可用数据低于设定的水位线时，自动从数据库中补充数据。
4. **线程安全**：通过并发队列和同步数据结构，在高并发场景下保证数据的安全访问。
5. **实时状态监控**：提供数据池状态的实时监控和日志报告。
6. **资源管理**：支持数据库连接池管理，自动释放资源。

---

## 二、测试计划结构

```plaintext
Test Plan
├── User Defined Variables (配置参数)
├── setUp Thread Group (初始化数据池)
│   └── JSR223 Sampler (数据池初始化脚本)
├── Thread Group (测试线程组)
│   └── JSR223 Sampler (测试执行脚本)
└── tearDown Thread Group
    └── JSR223 Sampler (资源清理脚本)
```

---

## 三、配置参数

### 3.1 数据库基础配置

```properties
DB_URL: jdbc:postgresql://localhost:5432/testdb
DB_USER: postgres
DB_PASSWORD: password
```

### 3.2 数据池配置

```properties
# 转账数据池配置
TRANSFER_POOL_QUERY: SELECT id, reference_no FROM transfer WHERE status='PENDING' LIMIT ?
TRANSFER_POOL_UPDATE_SQL: UPDATE transfer SET status = ? WHERE reference_no = ?
TRANSFER_POOL_BATCH_SIZE: 1000
TRANSFER_POOL_LOW_WATER_MARK: 100
TRANSFER_POOL_CLEAN_UP_THRESHOLD: 1000
TRANSFER_POOL_FIELDS: id:String,reference_no:String
TRANSFER_POOL_KEY_FIELDS: id,reference_no
```

---

## 四、核心类设计

### 4.1 类图关系

```plaintext
DataPoolFactory
    └── DataPoolManager
        ├── DataPoolConfig
        ├── FieldDefinition
        └── DynamicTestData
```

### 4.2 主要职责

1. **FieldDefinition**: 定义数据字段的名称和类型，并提供类型转换功能。
2. **DataPoolConfig**: 数据池配置类，负责读取并管理配置参数，包括数据库连接和查询的配置信息。
3. **DynamicTestData**: 测试数据的封装类，提供数据字段的读取和唯一性判断功能。
4. **DataPoolManager**: 数据池管理类，负责数据的获取、更新以及自动补充机制。支持线程安全的多线程数据访问。
5. **DataPoolFactory**: 数据池工厂类，负责管理多个数据池实例，提供统一的状态监控和资源释放接口。

---

## 五、完整实现代码

### 5.1 `FieldDefinition` 和 `DataPoolConfig` 实现

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
```

### 5.2 `DynamicTestData` 和 `DataPoolManager` 实现

```groovy
// DynamicTestData class
class DynamicTestData implements Serializable {
    private Map<String, Object> fields
    private List<String> keyFields
    
    DynamicTestData(Map<String, Object> fields, List<String> keyFields) {
        this.fields = new HashMap<>(fields)
        this.keyFields = keyFields
    }
    
    // 生成数据唯一键
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
    
    // 获取字段值
    Object get(String fieldName) {
        if (!fields.containsKey(fieldName)) {
            throw new IllegalArgumentException(
                "Field '${fieldName}' not found in data record"
            )
        }
        return fields.get(fieldName)
    }
    
    // 获取所有字段
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

// DataPoolManager class
class DataPoolManager {
    private static final Logger log = LoggerFactory.getLogger(DataPoolManager.class)
    
    private final String poolName
    private final ConcurrentLinkedQueue<DynamicTestData> dataQueue
    private final Map<String, Boolean> usedData
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
        // 使用LinkedHashMap实现LRU缓存
        this.usedData = Collections.synchronizedMap(
            new LinkedHashMap<String, Boolean>(config.cleanUpThreshold + 1, 0.75f, false) {
                @Override
                protected boolean removeEldestEntry(Map.Entry<String, Boolean> eldest) {
                    return size() > config.cleanUpThreshold
                }
            }
        )
        this.processedCount = new AtomicInteger(0)
        this.updateCount = new AtomicInteger(0)
        this.isRefilling = new AtomicBoolean(false)
        this.refillLock = new ReentrantLock()
        this.dataSource = setupDataSource()
        
        initialFill()
    }
    
    // 设置数据库连接池
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
    
    // 获取下一个可用数据
    DynamicTestData getNextData() {
        DynamicTestData data = dataQueue.poll()
        
        if (data != null) {
            synchronized(usedData) {
                if (usedData.containsKey(data.getUniqueKey())) {
                    return getNextData()
                }
                usedData.put(data.getUniqueKey(), Boolean.TRUE)
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
    
    // 触发异步数据补充
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
    
    // 补充数据池
    private void refillDataPool() {
        if (refillLock.tryLock()) {
            try {
                Connection conn = dataSource.getConnection()
                try {
                    PreparedStatement stmt = conn.prepareStatement(config.querySQL)
                    stmt.setInt(1, config.batchSize)
                    ResultSet rs = stmt.executeQuery()

                    int totalFetched = 0    // 从数据库获取的总记录数
                    int addedCount = 0      // 实际添加到队列的记录数
                    int duplicateCount = 0  // 重复的记录数

                    while (rs.next()) {
                        totalFetched++

                        DynamicTestData data = createDataFromResultSet(rs)
                        synchronized (usedData) {
                            if (!usedData.containsKey(data.getUniqueKey())) {
                                dataQueue.offer(data)
                                addedCount++
                            } else {
                                duplicateCount++
                            }
                        }
                    }

                    // 记录数据补充统计信息
                    log.info("${poolName} - Refilled pool. Fetched: ${totalFetched}, " +
                            "Added: ${addedCount}, Duplicates: ${duplicateCount}, " +
                            "Current size: ${dataQueue.size()}")

                } finally {
                    conn.close()
                }
            } catch (Exception e) {
                log.error("${poolName} - Database error during refill", e)
                throw e
            } finally {
                refillLock.unlock()
            }
        }
    }

    // 从ResultSet创建测试数据对象
    private DynamicTestData createDataFromResultSet(ResultSet rs) {
        Map<String, Object> fields = new HashMap<>()
        config.fieldDefinitions.each { fieldDef ->
            Object value = rs.getObject(fieldDef.name)
            fields.put(fieldDef.name, value)
        }
        return new DynamicTestData(fields, config.keyFields)
    }

    // 初始化数据池
    private void initialFill() {
        refillDataPool()
    }
    
    // 更新数据状态
    void updateStatus(String status, String reference_no) {
        Connection conn = null
        PreparedStatement stmt = null
        try {
            conn = dataSource.getConnection()
            stmt = conn.prepareStatement(config.updateSQL)
            stmt.setString(1, status)
            stmt.setString(2, reference_no)
            int updated = stmt.executeUpdate()
            
            if (updated > 0) {
                updateCount.incrementAndGet()
                log.debug("${poolName} - Updated status to ${status} for record: ${reference_no}")
            } else {
                log.warn("${poolName} - No record found to update for: ${reference_no}")
            }
        } catch (SQLException e) {
            log.error("${poolName} - Error updating status for: ${reference_no}", e)
            throw e
        } finally {
            try {
                if (stmt != null) stmt.close()
                if (conn != null) conn.close()
            } catch (SQLException e) {
                log.error("${poolName} - Error closing resources", e)
                throw e
            }
        }
    }
    
    // 关闭数据池
    void shutdown() {
        try {
            dataSource.close()
            log.info("${poolName} - Data source closed")
        } catch (Exception e) {
            log.error("${poolName} - Error closing data source", e)
            throw e
        }
    }
    
    // 获取数据池状态报告
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

### 5.3 `DataPoolFactory` 和 测试计划实现

```groovy
// DataPoolFactory class
class DataPoolFactory {
    private static final ConcurrentHashMap<String, DataPoolManager> pools = new ConcurrentHashMap<>()
    private static final Logger log = LoggerFactory.getLogger(DataPoolFactory.class)
    
    // 获取或创建数据池
    static DataPoolManager getOrCreate(String poolName, Properties config) {
        return pools.computeIfAbsent(poolName, { name ->
            log.info("Creating new data pool: " + name)
            DataPoolConfig poolConfig = new DataPoolConfig(name, config)
            return new DataPoolManager(poolConfig)
        })
    }
    
    // 关闭所有数据池
    static void shutdown() {
        pools.each { name, pool -> 
            try {
                log.info("Shutting down pool: " + name)
                pool.shutdown()
            } catch (Exception e) {
                log.error("Error shutting down pool: " + name, e)
                throw e
            }
        }
        pools.clear()
    }

    // 获取所有数据池状态
    static String getPoolsStatus() {
        StringBuilder status = new StringBuilder("Data Pools Status:\n")
        pools.each { name, pool ->
            status.append(pool.getStatusReport()).append("\n")
        }
        return status.toString()
    }
}
```

---

## 六、测试脚本实现

### 6.1 `setUp Thread Group - JSR223 Sampler`

初始化数据池并存储到 JMeter 属性中：

```groovy
// 主程序：初始化配置和数据池
Properties config = new Properties()
vars.entrySet().each { entry ->
    config.put(entry.key, entry.value)
}

// 初始化数据池
def transferPool = DataPoolFactory.getOrCreate("TRANSFER_POOL", config)

// 存储到全局属性中
props.put("transferPool", transferPool)

// 将 DataPoolFactory 存储到 JMeter 属性中
props.put("DataPoolFactory", DataPoolFactory)

log.info("Data pools initialized:\n" + DataPoolFactory.getPoolsStatus())
```

### 6.2 `Thread Group - JSR223 Sampler`

执行测试时，从数据池中获取数据并更新状态：

```groovy
try {
    def transferPool = props.get("transferPool")
    def data = transferPool.getNextData()

    if (data != null) {
        // 设置测试数据到变量中
        vars.put("id", data.get("id").toString())
        vars.put("reference_no", data.get("reference_no"))
        
        // 记录日志
        log.info("===JSR223 Sampler is running with id:${data.id},reference_no:${data.reference_no} ===")
        
        // 更新数据状态
        transferPool.updateStatus("COMPLETED", data.reference_no)
    } 
} catch (Exception ex) {
    // 发生异常时记录数据池状态和错误信息
    log.info(transferPool.getStatusReport())
    log.error("Unexpected error in Transfer PreProcessor", ex)
    Thread.sleep(1000)
    return
}
```

### 6.3 `tearDown Thread Group - JSR223 Sampler`

测试结束时，清理数据池并关闭资源：

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

---

## 七、使用说明

### 7.1 配置步骤

1. **创建测试计划**：在JMeter中创建测试计划。
2. **配置参数**：在User Defined Variables中设置数据库连接信息和数据池相关的配置参数。
3. **setUp Thread Group**：在setUp Thread Group中添加JSR223 Sampler，粘贴初始化数据池的代码。
4. **Thread Group**：在Thread Group中添加JSR223 Sampler，粘贴测试执行代码。
5. **tearDown Thread Group**：在tearDown Thread Group中添加JSR223 Sampler，用于关闭数据池和清理资源。

### 7.2 参数说明

- **DB_URL**: 数据库URL
- **DB_USER**: 数据库用户名
- **DB_PASSWORD**: 数据库密码
- **TRANSFER_POOL_QUERY**: 查询待处理数据的SQL语句
- **TRANSFER_POOL_UPDATE_SQL**: 更新数据状态的SQL语句
- **TRANSFER_POOL_BATCH_SIZE**: 每次从数据库获取的记录数
- **TRANSFER_POOL_LOW_WATER_MARK**: 低水位标记，触发数据补充的阈值
- **TRANSFER_POOL_CLEAN_UP_THRESHOLD**: 用于LRU缓存的清理阈值
- **TRANSFER_POOL_FIELDS**: 数据字段定义
- **TRANSFER_POOL_KEY_FIELDS**: 唯一键字段

---

## 八、注意事项

1. **数据库连接**：确保数据库连接参数正确，监控连接池使用情况，适当设置最大连接数。
2. **内存管理**：监控JVM内存使用情况，适当调整清理阈值，注意数据补充频率。
3. **异常处理**：所有数据库操作都需要正确的异常处理，记录详细的错误日志，并实现优雅降级。
4. **性能优化**：合理设置批量查询大小，优化数据库查询，必要时添加数据库索引。

---

## 九、扩展与优化

1. **功能扩展**：
   - 数据预热机制：在测试开始前预先加载部分数据，减少冷启动影响。
   - 数据验证功能：在数据获取和使用时进行自动化的数据验证，确保数据的一致性和正确性。
   - 添加更多监控指标：提供更加丰富的运行时监控指标，如数据库查询时间、数据池命中率等。

2. **性能优化**：
   - 数据分片机制：对于大规模数据集，考虑将数据按分片存储管理，并在不同的线程组中处理不同分片的数据池。
   - 数据结构优化：对高频访问的数据结构进行优化，减少锁竞争和同步操作。
   - 缓存层优化：为数据池添加本地缓存层，减轻数据库查询负担，加速数据访问。

3. **监控改进**：
   - 性能统计：为每个数据池收集性能统计信息，如补充次数、处理时间、平均查询响应时间等。
   - 实时监控：集成实时监控工具，如Prometheus、Grafana，实时展示数据池的使用情况和性能状态。

---

通过以上方案，可以在高并发场景下实现线程安全、数据唯一性保障的参数化测试，满足复杂测试场景中的数据管理需求。