---
layout:     post
title:      "JMeter 高并发测试方案：支持多数据池的线程安全参数化测试实现"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-06 11:00:01
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---



# JMeter 高并发测试方案：多数据池线程安全参数化测试实现（一）

## 一、方案概述

### 1.1 目标
本方案旨在解决JMeter高并发测试中的数据参数化问题，实现一个线程安全、高效的多数据池管理系统。

### 1.2 主要功能
1. **多数据池管理**：支持多个独立数据池的创建和管理
2. **数据唯一性**：确保测试数据不被重复使用
3. **自动补充机制**：数据池存量不足时自动补充
4. **线程安全**：支持高并发场景下的安全访问
5. **状态监控**：实时监控数据池状态和使用情况
6. **资源管理**：自动管理数据库连接和资源清理

### 1.3 技术特点
1. **并发控制**
   - ConcurrentLinkedQueue 用于数据存储
   - synchronized 同步访问机制
   - 原子操作计数器
   
2. **数据管理**
   - 数据库连接池
   - 批量数据获取
   - 自动清理机制
   
3. **监控统计**
   - 数据使用统计
   - 实时状态报告
   - 详细日志记录

## 二、测试计划结构

```plaintext
Test Plan
├── User Defined Variables (配置参数)
├── setUp Thread Group (初始化数据池)
│   └── JSR223 Sampler (数据池初始化脚本)
├── Thread Group (测试线程组)
│   └── JSR223 Sampler (测试执行脚本)
└── tearDown Thread Group (资源清理)
    └── JSR223 Sampler (清理脚本)
```

## 三、配置参数设置

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
1. **DataPoolFactory**: 数据池工厂，管理多个数据池实例
2. **DataPoolManager**: 数据池核心管理类，处理数据获取和更新
3. **DataPoolConfig**: 数据池配置类，管理配置参数
4. **FieldDefinition**: 字段定义类，处理字段类型和转换
5. **DynamicTestData**: 测试数据封装类，提供数据访问接口

## 五、实现代码（第一部分：FieldDefinition和DataPoolConfig）

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

这是方案的第一部分，主要包含了配置和字段定义相关的实现。下一部分将继续介绍 DynamicTestData 和 DataPoolManager 的实现。



# JMeter 高并发测试方案：多数据池线程安全参数化测试实现（二）

继续前面的实现，本部分主要介绍 DynamicTestData 和 DataPoolManager 类的实现。

## 六、实现代码（第二部分：DynamicTestData和DataPoolManager）

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

这部分代码主要实现了：

1. **DynamicTestData类**：
   - 测试数据的封装
   - 唯一键生成
   - 字段值访问
   - 数据展示格式化

2. **DataPoolManager类**：
   - 数据池核心管理
   - 线程安全的数据访问
   - 自动数据补充机制
   - 状态监控和报告
   - 数据库操作封装
   - 资源管理

下一部分将介绍 DataPoolFactory 和主程序的实现。


# JMeter 高并发测试方案：多数据池线程安全参数化测试实现（三）

## 七、实现代码（第三部分：DataPoolFactory和主程序实现）

### 7.1 DataPoolFactory 实现

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

### 7.2 测试计划实现

#### 1. setUp Thread Group - JSR223 Sampler（初始化脚本）

```groovy
// Main program: Initialize configuration and data pools
Properties config = new Properties()
vars.entrySet().each { entry ->
    config.put(entry.key, entry.value)
}

// Initialize data pools
def transferPool = DataPoolFactory.getOrCreate("TRANSFER_POOL", config)

// Store in global properties
props.put("transferPool", transferPool)

// Store DataPoolFactory as a JMeter property
props.put("DataPoolFactory", DataPoolFactory)

log.info("Data pools initialized:\n" + DataPoolFactory.getPoolsStatus())
```

#### 2. Thread Group - JSR223 Sampler（测试执行脚本）

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

#### 3. tearDown Thread Group - JSR223 Sampler（清理脚本）

```groovy
// Get DataPoolFactory from JMeter properties
def DataPoolFactory = props.get("DataPoolFactory")

if (DataPoolFactory) {
    log.info("Final status:\n" + DataPoolFactory.getPoolsStatus())
    DataPoolFactory.shutdown()
    log.info("All data pools have been shut down")
} else {
    log.error("DataPoolFactory not found in properties")
}
```

## 八、使用说明

### 8.1 配置步骤

1. 在JMeter中创建测试计划
2. 添加User Defined Variables并设置必要的配置参数
3. 创建setUp Thread Group并添加初始化脚本
4. 创建测试线程组并添加测试执行脚本
5. 创建tearDown Thread Group并添加清理脚本

### 8.2 配置参数示例

```properties
# 数据库配置
DB_URL=jdbc:postgresql://localhost:5432/testdb
DB_USER=postgres
DB_PASSWORD=password

# 转账数据池配置
TRANSFER_POOL_QUERY=SELECT id, reference_no FROM transfer WHERE status='PENDING' LIMIT ?
TRANSFER_POOL_UPDATE_SQL=UPDATE transfer SET status = ? WHERE reference_no = ?
TRANSFER_POOL_BATCH_SIZE=1000
TRANSFER_POOL_LOW_WATER_MARK=100
TRANSFER_POOL_CLEAN_UP_THRESHOLD=1000
TRANSFER_POOL_FIELDS=id:String,reference_no:String
TRANSFER_POOL_KEY_FIELDS=id,reference_no
```

## 九、监控与调优

### 9.1 监控指标

1. **数据池状态**
   - 已处理数据量（Processed）
   - 可用数据量（Available）
   - 已更新数据量（Updated）
   - 已使用数据量（Used）

2. **数据补充情况**
   - 获取的总记录数（Fetched）
   - 实际添加记录数（Added）
   - 重复记录数（Duplicates）

### 9.2 调优参数

1. **批量大小（BATCH_SIZE）**
   - 控制每次从数据库获取的数据量
   - 建议根据并发量调整

2. **低水位标记（LOW_WATER_MARK）**
   - 触发数据补充的阈值
   - 建议设置为批量大小的10%-20%

3. **清理阈值（CLEAN_UP_THRESHOLD）**
   - 已使用数据的最大保留量
   - 建议根据内存使用情况调整

## 十、注意事项

1. **数据库连接**
   - 确保数据库连接参数正确
   - 监控连接池使用情况
   - 适当设置最大连接数

2. **内存管理**
   - 监控JVM内存使用情况
   - 适当调整清理阈值
   - 注意数据补充频率

3. **异常处理**
   - 所有数据库操作都要有异常处理
   - 记录详细的错误日志
   - 实现优雅降级策略

4. **性能优化**
   - 合理设置批量查询大小
   - 优化数据库查询语句
   - 考虑添加索引

## 十一、扩展建议

1. **功能扩展**
   - 添加数据预热机制
   - 实现数据验证功能
   - 添加更多监控指标

2. **性能优化**
   - 实现数据分片机制
   - 优化数据结构
   - 添加缓存层

3. **监控改进**
   - 添加性能统计
   - 实现实时监控
   - 优化日志记录

这就是完整的方案实现。该方案提供了一个线程安全、高效的数据池管理系统，适用于各种高并发测试场景。您可以根据具体需求进行调整和优化。




以下是基于上述方案整理的一份完整的代码，包括 `setUp Thread Group`（初始化数据池），`Thread Group`（测试执行脚本），`tearDown Thread Group`（资源清理脚本），以及核心的类实现。这些代码适用于在 JMeter 中实现高并发场景下的多数据池线程安全测试。

# 完整代码实现

---

## 1. `setUp Thread Group` - JSR223 Sampler (池初始化代码)

```groovy
// Import necessary classes
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
        keyFields.each { field ->
            sb.append("${field}=${fields.get(field)}, ")
        }
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
        this.usedData = Collections.synchronizedMap(
            new LinkedHashMap<String, Boolean>(config.cleanUpThreshold + 1, 0.75f, true) {
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
                Connection conn = dataSource.getConnection()
                try {
                    PreparedStatement stmt = conn.prepareStatement(config.querySQL)
                    stmt.setInt(1, config.batchSize)
                    ResultSet rs = stmt.executeQuery()

                    int totalFetched = 0       // 从数据库获取的记录总数
                    int addedCount = 0         // 实际添加到队列的记录数
                    int duplicateCount = 0     // 重复记录数

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
                    log.info("${poolName} - Refilled pool. Fetched: ${totalFetched}, Added: ${addedCount}, Duplicates: ${duplicateCount}, Current size: ${dataQueue.size()}")
                } finally {
                    conn.close()
                }
            } catch (Exception e) {
                log.error("${poolName} - Error during refill", e)
                throw e
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
            log.error("${poolName} - Error updating status", e)
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
    
    void shutdown() {
        try {
            dataSource.close()
            log.info("${poolName} - Data source closed")
        } catch (Exception e) {
            log.error("${poolName} - Error closing data source", e)
            throw e
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
                throw e
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

---

## 2. `Thread Group` - JSR223 Sampler (测试执行代码)

```groovy
try {
    def transferPool = props.get("transferPool")
    def data = transferPool.getNextData()

    if (data != null) {
        // 设置数据到 JMeter 变量
        vars.put("id", data.get("id").toString())
        vars.put("reference_no", data.get("reference_no"))
        
        // 日志记录
        log.info("Data fetched for id: ${data.id}, reference_no: ${data.reference_no}")
        
        // 更新数据状态
        transferPool.updateStatus("COMPLETED", data.reference_no)
    } 
} catch (Exception ex) {
    // 记录错误并等待
    log.info(transferPool.getStatusReport())
    log.error("Unexpected error in data fetching", ex)
    Thread.sleep(1000)
    return
}
```

---

## 3. `tearDown Thread Group` - JSR223 Sampler (资源清理代码)

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

## 4. 配置文件示例

配置文件（例如 `User Defined Variables`）可以类似如下设置：

```properties
# 数据库配置
DB_URL=jdbc:postgresql://localhost:5432/testdb
DB_USER=postgres
DB_PASSWORD=password

# 转账数据池配置
TRANSFER_POOL_QUERY=SELECT id, reference_no FROM transfer WHERE status='PENDING' LIMIT ?
TRANSFER_POOL_UPDATE_SQL=UPDATE transfer SET status = ? WHERE reference_no = ?
TRANSFER_POOL_BATCH_SIZE=1000
TRANSFER_POOL_LOW_WATER_MARK=100
TRANSFER_POOL_CLEAN_UP_THRESHOLD=1000
TRANSFER_POOL_FIELDS=id:String,reference_no:String
TRANSFER_POOL_KEY_FIELDS=id,reference_no
```

---

## 总结

这是一份完整的方案实现，适用于高并发场景的 JMeter 测试，通过使用多数据池管理和线程安全的参数化测试，确保了数据的唯一性和稳定性。你可以根据具体需求调整批量获取大小、低水位标记、清理策略等相关参数。