---
layout:     post
title:      "JMeter 高并发测试方案：基于数据库的线程安全参数化测试实现"
subtitle:   "「 Performance Testing 」" 
date:       2024-11-09 11:00:01
author:     "Vick Huang"
header-img: "img/bg-walle.jpg"
catalog: true
tags:
    - Testing
    - Performance Testing
---

# JMeter 高并发测试方案：基于数据库的线程安全参数化测试实现

## 引言

本测试方案旨在解决在 JMeter 高并发测试场景中的一个常见问题：如何从数据库中获取数据作为 HTTP 请求的参数，并确保在多线程并发执行时，这些参数不会被重复使用。

### 主要特点
1. **数据唯一性保证**：通过并发安全的数据结构确保每条测试数据只被使用一次
2. **高性能数据获取**：使用数据库连接池和批量查询优化数据获取性能
3. **动态数据补充**：实现自动化的数据补充机制，确保测试数据持续可用
4. **资源智能管理**：包含完整的资源初始化和清理机制
5. **状态实时监控**：提供数据使用情况的实时监控能力

### 适用场景
- 需要确保请求参数不重复的接口测试
- 大规模并发测试
- 基于数据库数据的性能测试
- 需要动态管理测试数据的场景

## 测试计划实现

### 1. JMeter 测试计划结构

```
Test Plan
├── User Defined Variables (配置参数)
├── setUp Thread Group (初始化数据池)
│   └── JSR223 Sampler (数据池初始化脚本)
├── Thread Group (主测试线程组)
│   ├── JSR223 PreProcessor (获取测试数据)
│   └── HTTP Request (发送请求)
└── tearDown Thread Group (清理资源)
    └── JSR223 Sampler (清理脚本)
```

### 2. User Defined Variables 配置

```properties
# 数据库配置
DB_URL: jdbc:postgresql://localhost:5432/testdb
DB_USER: postgres
DB_PASSWORD: password
DB_QUERY: SELECT *  FROM transfer where status='PENDING'  LIMIT ?

# 数据池配置
POOL_LOW_WATER_MARK: 100
POOL_BATCH_SIZE: 1000
CLEAN_UP_THRESHOLD：1000

# HTTP配置
HTTP_PROTOCOL: https
HTTP_DOMAIN: api.example.com
HTTP_PATH: /process
HTTP_METHOD: POST
```

### 3. 核心代码实现

#### 3.1 setUp Thread Group - JSR223 Sampler (初始化数据池)

```groovy
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;
import java.sql.*;
import org.apache.commons.dbcp2.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// 定义数据类
class TestData implements Serializable {
    private String id;
    private String referenceNo;
    
    TestData(String id, String referenceNo) {
        this.id = id;
        this.referenceNo = referenceNo;
    }
    
    String getId() { return id; }
    String getReferenceNo() { return referenceNo; }
}

// 数据池管理器
class DataPoolManager {
    private static final Logger log = LoggerFactory.getLogger(DataPoolManager.class);
    
    private ConcurrentLinkedQueue<TestData> dataQueue;
    private ConcurrentHashMap<String, Boolean> usedData;
    private AtomicInteger processedCount;
    private AtomicBoolean isRefilling;
    private ReentrantLock refillLock;
    private BasicDataSource dataSource;
    private Properties config;

    // 清理阈值和清理比例
    private final int cleanUpThreshold;
    private final double cleanUpRatio = 0.3;
    private final int poolLowWaterMark;
    private final int poolBatchSize;

    DataPoolManager(Properties config) {
        this.config = config;
        this.dataQueue = new ConcurrentLinkedQueue<>();
        this.usedData = new ConcurrentHashMap<>();
        this.processedCount = new AtomicInteger(0);
        this.isRefilling = new AtomicBoolean(false);
        this.refillLock = new ReentrantLock();
        this.dataSource = setupDataSource();

        // 读取清理阈值配置
        this.cleanUpThreshold = Integer.parseInt(config.getProperty("CLEAN_UP_THRESHOLD", "100"));
        this.poolLowWaterMark = Integer.parseInt(config.getProperty("POOL_LOW_WATER_MARK", "100"));
        this.poolBatchSize = Integer.parseInt(config.getProperty("POOL_BATCH_SIZE", "1000"));

        initialFill();
    }
    
    private BasicDataSource setupDataSource() {
        BasicDataSource ds = new BasicDataSource();
        ds.setUrl(config.getProperty("DB_URL"));
        ds.setUsername(config.getProperty("DB_USER"));
        ds.setPassword(config.getProperty("DB_PASSWORD"));
        ds.setInitialSize(5);
        ds.setMaxTotal(20);
        return ds;
    }
    
    TestData getNextData() {
        TestData data = dataQueue.poll();
        
        if (data != null) {
            if (usedData.putIfAbsent(data.getId(), Boolean.TRUE) != null) {
                return getNextData();
            }
            processedCount.incrementAndGet();
            
            if (dataQueue.size() < poolLowWaterMark) {
                triggerRefill();
            }
            
            return data;
        }

        refillDataPool();
        return dataQueue.poll();
    }
    
    private void triggerRefill() {
        if (isRefilling.compareAndSet(false, true)) {
            new Thread(() -> {
                try {
                    refillDataPool();
                } finally {
                    isRefilling.set(false);
                }
            }).start();
        }
    }
    
    private void refillDataPool() {
        if (refillLock.tryLock()) {
            try {
            	  // 检查usedData是否达到清理阈值
	            if (usedData.size() > cleanUpThreshold) {
	                cleanUp(); 
	            }

                String query = config.getProperty("DB_QUERY");
                Connection conn = dataSource.getConnection();
                try {
                    PreparedStatement stmt = conn.prepareStatement(query);
                    stmt.setInt(1, poolBatchSize);
                    ResultSet rs = stmt.executeQuery();
                    while (rs.next()) {
                        TestData data = new TestData(
                            rs.getString("id"),
                            rs.getString("reference_no")
                        );
                        if (!usedData.containsKey(data.getId())) {
                            dataQueue.offer(data);
                        }
                    }
                    
                    log.info("Refilled pool. Current size: " + dataQueue.size());
                } finally {
                    conn.close();
                }
            } catch (Exception e) {
                log.error("Database error during refill", e);
            } finally {
                refillLock.unlock();
            }
        }
    }
    
    private void initialFill() {
        refillDataPool();
    }
    
    void shutdown() {
        try {
            dataSource.close();
        } catch (Exception e) {
            log.error("Error closing data source", e);
        }
    }
    
    int getProcessedCount() {
        return processedCount.get();
    }
    
    int getAvailableCount() {
        return dataQueue.size();
    }
    
    void updateStatus(String referenceNo, String status) {
        Connection conn = null;
        PreparedStatement stmt = null;
        try {
            conn = dataSource.getConnection();
            String updateQuery = "UPDATE transfer SET status = ? WHERE reference_no = ?";
            stmt = conn.prepareStatement(updateQuery);
            stmt.setString(1, status);
            stmt.setString(2, referenceNo);
            int updated = stmt.executeUpdate();
            
            if (updated > 0) {
                log.debug("Updated status to " + status + " for reference_no: " + referenceNo);
            } else {
                log.warn("No record found to update for reference_no: " + referenceNo);
            }
        } catch (SQLException e) {
            log.error("Error updating status for reference_no: " + referenceNo, e);
            throw e;
        } finally {
            try {
                if (stmt != null) stmt.close();
                if (conn != null) conn.close();
            } catch (SQLException e) {
                log.error("Error closing resources", e);
            }
        }
    }

    // 清理方法
    private void cleanUp() {
        if (usedData.size() > cleanUpThreshold) {
            int itemsToRemove = (int) (usedData.size() * cleanUpRatio);
            log.info("Cleaning up " + itemsToRemove + " items from used data.");

            // 遍历并移除一定比例的数据
            Iterator<String> it = usedData.keySet().iterator();
            while (it.hasNext() && itemsToRemove > 0) {
                it.next(); // 获取下一个键
                it.remove(); // 移除该键
                itemsToRemove--;
            }

            log.info("Cleanup completed. Current used data count: " + usedData.size());
        }
    }
}

// 创建配置
Properties config = new Properties();
config.put("DB_URL", vars.get("DB_URL"));
config.put("DB_USER", vars.get("DB_USER"));
config.put("DB_PASSWORD", vars.get("DB_PASSWORD"));
config.put("DB_QUERY", vars.get("DB_QUERY"));
config.put("POOL_LOW_WATER_MARK", vars.get("POOL_LOW_WATER_MARK"));
config.put("POOL_BATCH_SIZE", vars.get("POOL_BATCH_SIZE"));
config.put("HTTP_PROTOCOL", vars.get("HTTP_PROTOCOL"));
config.put("HTTP_DOMAIN", vars.get("HTTP_DOMAIN"));
config.put("HTTP_PATH", vars.get("HTTP_PATH"));
config.put("HTTP_METHOD", vars.get("HTTP_METHOD"));
config.put("CLEAN_UP_THRESHOLD", vars.get("CLEAN_UP_THRESHOLD")); // 添加清理阈值配置

// 初始化数据池管理器
DataPoolManager dataPool = new DataPoolManager(config);
props.put("dataPool", dataPool);
props.put("config", config);

log.info("Data pool initialized successfully,pool.Current size: " + dataPool.dataQueue.size());```

#### 3.2 Thread Group - JSR223 PreProcessor (获取测试数据)

```groovy
// 获取数据池
def dataPool = props.get("dataPool")

// 获取测试数据
def data = dataPool.getNextData()

if (data != null) {
    // 设置请求参数
    vars.put("id", data.id)
    vars.put("referenceNo", data.referenceNo)
    log.info("===id: " + data.id+" referenceNo: "+data.referenceNo+"===")
    dataPool.updateStatus(data.referenceNo, "COMPLETED")
    log.info("===Updated status to CANCELLED for reference_no: ${data.referenceNo}===")
    // 记录状态（每100次请求）
    if (ctx.getThreadNum() % 100 == 0) {
        log.info(String.format(
            "Pool status - Processed: %d, Available: %d",
            dataPool.getProcessedCount(),
            dataPool.getAvailableCount()
        ))
    }
} else {
    log.error("No data available")
    return
}
```

#### 3.3 tearDown Thread Group - JSR223 Sampler (清理资源)

```groovy
// 获取数据池
def dataPool = props.get("dataPool")

if (dataPool != null) {
    // 清理资源
    dataPool.shutdown()
    log.info("Data pool resources cleaned up successfully")
}
```

### 4. 核心特性说明

1. **数据唯一性保证**
   - 使用 ConcurrentHashMap 跟踪已使用数据
   - 原子操作确保并发安全
   - 递归获取确保获得未使用数据

2. **高效数据管理**
   - 使用连接池管理数据库连接
   - 批量查询优化性能
   - 异步数据补充机制

3. **资源管理**
   - 完整的初始化和清理流程
   - 连接池资源管理
   - 异常处理机制

4. **监控和报告**
   - 实时状态监控
   - 定期状态报告
   - 详细的日志记录

### 5. 使用说明

1. **配置准备**
   - 设置数据库连接参数
   - 配置数据池参数
   - 配置 HTTP 请求参数

2. **运行测试**
   - 确保数据库可用
   - 设置适当的线程数和循环次数
   - 监控日志输出

3. **监控指标**
   - 已处理数据量
   - 可用数据量
   - 数据池状态
   - HTTP 响应状态

### 6. 注意事项

1. **性能优化**
   - 适当设置批量查询大小
   - 调整连接池参数
   - 监控数据库负载

2. **异常处理**
   - 数据库连接异常
   - 数据耗尽情况
   - 并发问题

3. **资源管理**
   - 确保资源正确释放
   - 监控内存使用
   - 避免资源泄露

此方案提供了一个完整的、可靠的解决方案，适用于需要确保参数唯一性的高并发测试场景。通过合理的数据结构和同步机制，确保了数据的线程安全和唯一性，同时保持了较高的性能。