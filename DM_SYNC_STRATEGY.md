# 达梦数据库基于唯一键的增量同步策略设计

## 一、唯一键处理机制

### 1. 唯一键类型支持

| 唯一键类型 | 定义 | 示例 | 处理方式 |
|------------|------|------|----------|
| 单字段唯一键 | 由单个字段构成的唯一键 | `user_id` | 直接使用该字段值作为唯一标识 |
| 复合唯一键 | 由多个字段构成的唯一键 | `user_id, product_id` | 将多个字段值组合成字符串，使用分隔符连接 |
| 唯一约束 | 表上定义的唯一约束 | `UNIQUE (email)` | 等同于单字段或复合唯一键 |
| 主键 | 表的主键 | `id` | 主键是特殊的唯一键，优先使用 |

### 2. 唯一键值生成规则

```java
/**
 * 构建唯一键值字符串
 * @param data 数据记录
 * @param uniqueKey 唯一键字段列表(逗号分隔)
 * @return 唯一键值字符串
 */
public static String buildUniqueKeyValue(Map<String, Object> data, String uniqueKey) {
    if (StringUtils.isBlank(uniqueKey)) {
        throw new IllegalArgumentException("唯一键不能为空");
    }
    
    String[] keyFields = uniqueKey.split(",");
    StringBuilder sb = new StringBuilder();
    
    for (int i = 0; i < keyFields.length; i++) {
        String field = keyFields[i].trim();
        Object value = data.get(field);
        
        if (value == null) {
            sb.append("NULL");
        } else {
            sb.append(value.toString());
        }
        
        if (i < keyFields.length - 1) {
            sb.append("|"); // 使用竖线作为分隔符
        }
    }
    
    return sb.toString();
}
```

## 二、增量数据捕获策略

### 1. 时间戳字段设计

| 字段名 | 数据类型 | 约束 | 描述 |
|--------|----------|------|------|
| `create_time` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | 记录数据创建时间 |
| `update_time` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` | 记录数据更新时间 |

### 2. 触发器设计

#### 2.1 创建时间戳字段（如果不存在）

```sql
-- 为源表添加时间戳字段
alter table ${schema}.${table_name} add column create_time timestamp default current_timestamp;
alter table ${schema}.${table_name} add column update_time timestamp default current_timestamp;

-- 为现有数据设置默认时间戳
update ${schema}.${table_name} set create_time = current_timestamp, update_time = current_timestamp;

-- 创建索引提高查询性能
create index idx_${table_name}_update_time on ${schema}.${table_name}(update_time);
```

#### 2.2 增删改触发器

```sql
-- 插入触发器
create trigger trg_${table_name}_insert
after insert on ${schema}.${table_name}
for each row
begin
    -- 插入变更日志
    insert into sync_meta.sync_change_log(
        log_id, table_name, schema_name, operation_type,
        unique_key_value, change_time, sync_status
    ) values (
        sys_guid(), '${table_name}', '${schema}', 'INSERT',
        ${unique_key_values}, current_timestamp, 'PENDING'
    );
end;

-- 更新触发器
create trigger trg_${table_name}_update
after update on ${schema}.${table_name}
for each row
begin
    -- 更新时间戳
    update ${schema}.${table_name} 
    set update_time = current_timestamp 
    where ${unique_key_conditions};
    
    -- 插入变更日志
    insert into sync_meta.sync_change_log(
        log_id, table_name, schema_name, operation_type,
        unique_key_value, change_time, sync_status
    ) values (
        sys_guid(), '${table_name}', '${schema}', 'UPDATE',
        ${unique_key_values}, current_timestamp, 'PENDING'
    );
end;

-- 删除触发器
create trigger trg_${table_name}_delete
after delete on ${schema}.${table_name}
for each row
begin
    -- 插入变更日志
    insert into sync_meta.sync_change_log(
        log_id, table_name, schema_name, operation_type,
        unique_key_value, change_time, sync_status
    ) values (
        sys_guid(), '${table_name}', '${schema}', 'DELETE',
        ${unique_key_values}, current_timestamp, 'PENDING'
    );
end;
```

### 3. 增量数据查询

#### 3.1 基于时间戳的增量数据查询

```sql
-- 查询源库增量数据
select * from ${src_schema}.${table_name}
where update_time > ?
order by update_time asc
limit ? offset ?;
```

#### 3.2 基于变更日志的增量数据查询

```sql
-- 查询待同步的变更日志
select * from sync_meta.sync_change_log
where table_name = ?
  and schema_name = ?
  and sync_status = 'PENDING'
  and change_time > ?
order by change_time asc;

-- 根据变更日志查询源库数据
select * from ${src_schema}.${table_name}
where ${unique_key_conditions};
```

## 三、增量同步核心逻辑

### 1. 同步流程图

```
┌─────────────────────┐
│  开始同步任务       │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  获取上次同步时间   │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  查询源库增量数据   │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  按批次处理增量数据 │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  遍历处理单条数据   │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  基于唯一键查询目标库 │
└─────────────────────┘
          │
          ▼
┌────────────┐     ┌─────────────┐     ┌─────────────┐
│  数据存在？├─────►    是      │     │     否      │
└────────────┘     └─────────────┘     └─────────────┘
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐      ┌─────────────┐
                  │  更新目标库  │      │  插入目标库  │
                  └─────────────┘      └─────────────┘
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐      ┌─────────────┐
                  │  更新成功？  │      │  插入成功？  │
                  └─────────────┘      └─────────────┘
                         │                     │
                         ▼                     ▼
           ┌────────────┐         ┌─────────────────────┐
           │     是     │         │         否          │
           └────────────┘         └─────────────────────┘
                  │                     │
                  ▼                     ▼
┌──────────────────────────────────────────────────────┐
│                      记录同步结果                     │
└──────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────┐
│  更新同步任务状态   │
└─────────────────────┘
                  │
                  ▼
┌─────────────────────┐
│    结束同步任务     │
└─────────────────────┘
```

### 2. 单条数据同步逻辑

```java
/**
 * 同步单条增量数据
 * @param task 同步任务配置
 * @param srcData 源库数据
 * @param syncLog 同步日志
 * @return 同步结果
 */
public SyncResult syncSingleData(SyncTask task, Map<String, Object> srcData, SyncLog syncLog) {
    SyncResult result = new SyncResult();
    result.setSuccess(false);
    
    try {
        // 1. 构建唯一键值
        String uniqueKey = task.getUniqueKey();
        String uniqueKeyValue = UniqueKeyUtil.buildUniqueKeyValue(srcData, uniqueKey);
        
        // 2. 基于唯一键查询目标库数据
        Map<String, Object> dstData = dstDao.getDataByUniqueKey(task, uniqueKeyValue);
        
        // 3. 执行同步操作
        if (dstData == null) {
            // 目标库不存在，执行插入
            int insertCount = dstDao.insertData(task, srcData);
            result.setSuccess(insertCount > 0);
            result.setOperationType("INSERT");
        } else {
            // 目标库存在，执行更新
            int updateCount = dstDao.updateData(task, srcData, uniqueKeyValue);
            result.setSuccess(updateCount > 0);
            result.setOperationType("UPDATE");
        }
        
        // 4. 更新同步日志状态
        if (result.isSuccess()) {
            syncLog.setStatus("SUCCESS");
            syncLog.setSyncTime(new Date());
        } else {
            syncLog.setStatus("FAILED");
            syncLog.setErrorMessage("同步失败：操作影响行数为0");
        }
        
    } catch (Exception e) {
        result.setSuccess(false);
        result.setErrorMessage("同步失败：" + e.getMessage());
        syncLog.setStatus("FAILED");
        syncLog.setErrorMessage(e.getMessage());
    }
    
    return result;
}
```

### 3. 删除操作同步逻辑

```java
/**
 * 同步删除操作
 * @param task 同步任务配置
 * @param deleteLogs 删除操作日志
 * @return 同步结果
 */
public List<SyncResult> syncDeleteOperations(SyncTask task, List<SyncChangeLog> deleteLogs) {
    List<SyncResult> results = new ArrayList<>();
    
    for (SyncChangeLog log : deleteLogs) {
        SyncResult result = new SyncResult();
        result.setSuccess(false);
        result.setOperationType("DELETE");
        
        try {
            // 1. 解析唯一键值
            String uniqueKeyValue = log.getUniqueKeyValue();
            
            // 2. 执行删除操作
            int deleteCount = dstDao.deleteDataByUniqueKey(task, uniqueKeyValue);
            
            // 3. 更新结果
            result.setSuccess(deleteCount > 0);
            
            // 4. 更新同步日志状态
            if (result.isSuccess()) {
                log.setSyncStatus("SYNCED");
                log.setSyncTime(new Date());
            } else {
                log.setSyncStatus("FAILED");
                log.setErrorMessage("删除失败：未找到对应数据");
            }
            
        } catch (Exception e) {
            result.setSuccess(false);
            result.setErrorMessage("删除失败：" + e.getMessage());
            log.setSyncStatus("FAILED");
            log.setErrorMessage(e.getMessage());
        }
        
        results.add(result);
    }
    
    return results;
}
```

## 四、并发控制与事务管理

### 1. 同步任务并发控制

| 并发控制策略 | 实现方式 | 优点 | 缺点 |
|--------------|----------|------|------|
| 任务级锁 | 在任务执行前获取分布式锁，执行完成后释放 | 实现简单、有效防止任务并发执行 | 可能导致任务排队，影响性能 |
| 数据级锁 | 对同步的数据范围加锁，允许多个任务并发执行不同数据范围 | 提高并发性能 | 实现复杂，可能导致死锁 |
| 乐观锁 | 使用版本号或时间戳字段，检测数据冲突 | 并发性能高 | 需要额外字段支持，冲突时需要重试 |

**选型决策**：采用**任务级锁**，确保同一同步任务不会并发执行，避免数据不一致问题。

### 2. 分布式锁实现

```java
/**
 * 同步任务分布式锁
 */
@Component
public class SyncTaskLock {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    /**
     * 获取任务锁
     * @param taskId 任务ID
     * @param timeout 超时时间(秒)
     * @return 是否获取成功
     */
    public boolean lock(String taskId, long timeout) {
        String lockKey = "sync:task:lock:" + taskId;
        String lockValue = UUID.randomUUID().toString();
        
        // 使用Redis的SETNX命令实现分布式锁
        Boolean result = redisTemplate.opsForValue().setIfAbsent(lockKey, lockValue, timeout, TimeUnit.SECONDS);
        
        return Boolean.TRUE.equals(result);
    }
    
    /**
     * 释放任务锁
     * @param taskId 任务ID
     */
    public void unlock(String taskId) {
        String lockKey = "sync:task:lock:" + taskId;
        redisTemplate.delete(lockKey);
    }
}
```

### 3. 数据库事务管理

```java
/**
 * 批量同步数据（带事务控制）
 * @param task 同步任务配置
 * @param dataList 增量数据列表
 * @return 同步结果
 */
@Transactional(rollbackFor = Exception.class)
public BatchSyncResult batchSyncData(SyncTask task, List<Map<String, Object>> dataList) {
    BatchSyncResult result = new BatchSyncResult();
    result.setTotalCount(dataList.size());
    result.setSuccessCount(0);
    result.setFailedCount(0);
    result.setFailedDataList(new ArrayList<>());
    
    for (Map<String, Object> data : dataList) {
        SyncResult syncResult = syncSingleData(task, data, new SyncLog());
        if (syncResult.isSuccess()) {
            result.setSuccessCount(result.getSuccessCount() + 1);
        } else {
            result.setFailedCount(result.getFailedCount() + 1);
            result.getFailedDataList().add(data);
        }
    }
    
    return result;
}
```

## 五、错误处理与重试机制

### 1. 错误分类与处理策略

| 错误类型 | 描述 | 处理策略 |
|----------|------|----------|
| 数据库连接错误 | 无法连接到源库或目标库 | 重试机制，指数退避算法 |
| SQL执行错误 | SQL语法错误、约束冲突等 | 记录错误日志，人工介入 |
| 数据格式错误 | 数据类型不匹配、格式错误等 | 数据转换、记录错误日志 |
| 并发冲突错误 | 数据被其他事务修改 | 重试机制，最多3次 |
| 系统资源错误 | 内存不足、磁盘空间不足等 | 记录错误日志，告警通知 |

### 2. 重试机制实现

```java
/**
 * 带重试机制的同步方法
 * @param task 同步任务配置
 * @param dataList 增量数据列表
 * @param maxRetries 最大重试次数
 * @return 同步结果
 */
public BatchSyncResult syncWithRetry(SyncTask task, List<Map<String, Object>> dataList, int maxRetries) {
    BatchSyncResult result = null;
    
    for (int retry = 0; retry <= maxRetries; retry++) {
        try {
            result = batchSyncData(task, dataList);
            
            // 如果同步成功，或重试次数已达上限，退出循环
            if (result.getFailedCount() == 0 || retry == maxRetries) {
                break;
            }
            
            // 等待一段时间后重试
            Thread.sleep((long) (Math.pow(2, retry) * 1000));
            
            // 更新待重试的数据列表
            dataList = result.getFailedDataList();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        } catch (Exception e) {
            // 记录异常日志
            log.error("同步重试失败，重试次数：{}", retry, e);
            
            // 如果重试次数已达上限，抛出异常
            if (retry == maxRetries) {
                throw new SyncException("同步失败，已达最大重试次数", e);
            }
            
            // 等待一段时间后重试
            try {
                Thread.sleep((long) (Math.pow(2, retry) * 1000));
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    return result;
}
```

## 六、性能优化策略

### 1. 批量处理优化

| 优化项 | 实现方式 | 预期效果 |
|--------|----------|----------|
| 批量查询 | 使用`limit`和`offset`分页查询增量数据 | 减少单次查询的数据量，降低内存占用 |
| 批量插入/更新 | 使用MyBatis-Plus的批量操作功能 | 减少数据库交互次数，提高同步效率 |
| 批量提交 | 设置合理的事务提交批次大小 | 平衡事务开销和数据一致性 |

### 2. 批量操作实现

```java
/**
 * 批量插入数据
 * @param task 同步任务配置
 * @param dataList 数据列表
 * @return 插入成功的数量
 */
public int batchInsertData(SyncTask task, List<Map<String, Object>> dataList) {
    if (CollectionUtils.isEmpty(dataList)) {
        return 0;
    }
    
    // 构建批量插入SQL
    StringBuilder sql = new StringBuilder();
    sql.append("INSERT INTO ").append(task.getDstSchema()).append(".").append(task.getDstTable());
    
    // 获取字段列表
    Set<String> fields = dataList.get(0).keySet();
    sql.append(" (");
    sql.append(String.join(", ", fields));
    sql.append(") VALUES ");
    
    // 构建参数占位符
    List<String> placeholders = new ArrayList<>();
    List<Object> params = new ArrayList<>();
    
    for (Map<String, Object> data : dataList) {
        StringBuilder rowPlaceholder = new StringBuilder("(");
        
        for (String field : fields) {
            rowPlaceholder.append("?,");
            params.add(data.get(field));
        }
        
        rowPlaceholder.deleteCharAt(rowPlaceholder.length() - 1);
        rowPlaceholder.append(")");
        placeholders.add(rowPlaceholder.toString());
    }
    
    sql.append(String.join(", ", placeholders));
    
    // 执行批量插入
    return jdbcTemplate.update(sql.toString(), params.toArray());
}
```

### 3. 索引优化

| 优化项 | 实现方式 | 预期效果 |
|--------|----------|----------|
| 源表索引 | 为时间戳字段创建索引 | 提高增量数据查询性能 |
| 目标表索引 | 为唯一键字段创建索引 | 提高基于唯一键的查询和更新性能 |
| 变更日志表索引 | 为表名、同步状态和变更时间创建组合索引 | 提高变更日志查询性能 |

### 4. 数据库连接池优化

```yaml
# HikariCP连接池配置
spring:
  datasource:
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-test-query: SELECT 1 FROM DUAL
```

## 七、数据一致性保障

### 1. 最终一致性实现

| 保障机制 | 实现方式 | 作用 |
|----------|----------|------|
| 变更日志 | 记录所有数据变更操作 | 确保数据变更可追溯，支持失败重试 |
| 重试机制 | 对失败的同步操作进行重试 | 确保数据最终同步到目标库 |
| 同步日志 | 记录每次同步的详细信息 | 便于排查数据不一致问题 |
| 数据校验 | 定期比对源库和目标库的数据 | 发现并修复数据不一致问题 |

### 2. 数据校验机制

```java
/**
 * 数据一致性校验
 * @param task 同步任务配置
 * @param startTime 开始时间
 * @param endTime 结束时间
 * @return 校验结果
 */
public DataCheckResult checkDataConsistency(SyncTask task, Date startTime, Date endTime) {
    DataCheckResult result = new DataCheckResult();
    result.setConsistent(true);
    result.setInconsistentDataList(new ArrayList<>());
    
    try {
        // 1. 查询源库指定时间范围内的数据
        List<Map<String, Object>> srcDataList = srcDao.getDataByTimeRange(task, startTime, endTime);
        
        // 2. 查询目标库指定时间范围内的数据
        List<Map<String, Object>> dstDataList = dstDao.getDataByTimeRange(task, startTime, endTime);
        
        // 3. 基于唯一键构建数据映射
        Map<String, Map<String, Object>> srcDataMap = buildDataMap(srcDataList, task.getUniqueKey());
        Map<String, Map<String, Object>> dstDataMap = buildDataMap(dstDataList, task.getUniqueKey());
        
        // 4. 比对源库和目标库的数据
        for (Map.Entry<String, Map<String, Object>> entry : srcDataMap.entrySet()) {
            String uniqueKey = entry.getKey();
            Map<String, Object> srcData = entry.getValue();
            Map<String, Object> dstData = dstDataMap.get(uniqueKey);
            
            if (dstData == null) {
                // 目标库缺少数据
                result.setConsistent(false);
                result.getInconsistentDataList().add(new InconsistentData(uniqueKey, srcData, null));
            } else if (!dataEquals(srcData, dstData)) {
                // 数据不一致
                result.setConsistent(false);
                result.getInconsistentDataList().add(new InconsistentData(uniqueKey, srcData, dstData));
            }
        }
        
        // 5. 检查目标库是否有多余数据
        for (Map.Entry<String, Map<String, Object>> entry : dstDataMap.entrySet()) {
            String uniqueKey = entry.getKey();
            
            if (!srcDataMap.containsKey(uniqueKey)) {
                // 目标库有多余数据
                result.setConsistent(false);
                result.getInconsistentDataList().add(new InconsistentData(uniqueKey, null, entry.getValue()));
            }
        }
        
    } catch (Exception e) {
        result.setConsistent(false);
        result.setErrorMessage(e.getMessage());
    }
    
    return result;
}
```

## 八、监控与告警

### 1. 监控指标

| 指标名称 | 类型 | 描述 | 告警阈值 |
|----------|------|------|----------|
| 同步任务执行次数 | Counter | 同步任务执行的总次数 | - |
| 同步任务成功次数 | Counter | 同步任务成功执行的次数 | - |
| 同步任务失败次数 | Counter | 同步任务失败执行的次数 | >0 |
| 数据同步成功数 | Counter | 成功同步的数据记录数 | - |
| 数据同步失败数 | Counter | 失败同步的数据记录数 | >0 |
| 同步任务平均执行时间 | Gauge | 同步任务的平均执行时间 | >30秒 |
| 同步延迟时间 | Gauge | 增量数据从产生到同步完成的延迟时间 | >60秒 |

### 2. 告警机制

| 告警类型 | 触发条件 | 告警方式 | 处理流程 |
|----------|----------|----------|----------|
| 同步任务失败 | 同步任务执行失败 | 邮件、短信、钉钉 | 查看同步日志，排查失败原因，手动重试 |
| 同步延迟过高 | 同步延迟超过阈值 | 邮件、短信 | 检查系统性能，优化同步策略 |
| 数据不一致 | 数据校验发现不一致 | 邮件、短信、钉钉 | 数据修复，排查同步逻辑问题 |
| 系统资源不足 | CPU/内存/磁盘使用率超过阈值 | 邮件、短信 | 扩容系统资源，优化代码 |

## 九、总结

本增量同步策略基于唯一键实现了达梦数据库不同SCHEMA间的表数据增量同步，具有以下特点：

1. **可靠性高**：通过触发器+时间戳机制确保增量数据不丢失，变更日志支持失败重试
2. **性能优化**：批量处理、事务控制、索引优化等提高同步效率
3. **数据一致性**：分布式锁、事务管理、数据校验等确保数据最终一致性
4. **可监控性**：详细的同步日志和监控指标，便于问题排查和性能优化
5. **可扩展性**：支持单字段和复合唯一键，可扩展支持更多类型的唯一键

该策略适用于中小规模的数据同步场景，能够满足用户通过Web实现增量同步的需求。