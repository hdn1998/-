# 达梦数据库增量同步系统架构设计

## 一、需求分析

### 1. 业务目标
- 实现达梦数据库不同SCHEMA间的表数据增量同步
- 支持十几张表的独立同步配置
- 基于表的唯一键进行增量数据匹配
- 提供Web界面实现增量同步操作

### 2. 核心场景
- 配置同步任务（源库/SCHEMA/表、目标库/SCHEMA/表、唯一键、时间戳字段）
- 手动触发增量同步
- 定时自动增量同步
- 查看同步状态和日志

### 3. 技术约束
- 基于Java实现后端服务
- 使用达梦数据库作为源库和目标库
- 表有唯一键但可能不是主键
- 需要增量同步而非全量同步

## 二、架构选型

### 1. 增量捕获机制选型

| 方案 | 实现方式 | 优点 | 缺点 | 适用场景 |
|------|----------|------|------|----------|
| 触发器+时间戳 | 在源表上创建增删改触发器，记录变更到日志表 | 实现简单、可靠性高、支持复杂过滤 | 对源库有一定性能影响 | 中小型数据同步、对实时性要求不高的场景 |
| 日志挖掘 | 解析达梦数据库的重做日志（REDO LOG） | 对源库性能影响小、实时性高 | 实现复杂、依赖数据库版本、需要特定权限 | 大型数据同步、对实时性要求高的场景 |
| 日志轮询 | 定期查询源表的时间戳字段 | 实现最简单、对源库无侵入 | 实时性差、可能漏数据、对源表有查询压力 | 小型数据同步、源表无法创建触发器的场景 |

**选型决策**：采用**触发器+时间戳**方案，平衡了实现复杂度、可靠性和性能要求，适合当前需求场景。

### 2. 技术栈选型

| 分类 | 技术选型 | 版本 | 决策依据 |
|------|----------|------|----------|
| 前端框架 | Vue.js | 3.x | 轻量级、响应式、易于集成、学习成本低 |
| UI组件库 | Element Plus | 2.x | 丰富的组件、与Vue 3兼容、文档完善 |
| 后端框架 | Spring Boot | 3.x | 快速开发、自动配置、生态丰富 |
| ORM框架 | MyBatis-Plus | 3.5.x | 对MyBatis的增强、简化开发、支持达梦数据库 |
| 任务调度 | Quartz | 2.3.x | 成熟稳定、支持复杂调度规则 |
| 日志框架 | SLF4J + Logback | - | 灵活配置、性能优异 |
| 连接池 | HikariCP | 4.x | 高性能、轻量级、Spring Boot默认推荐 |
| 数据库 | 达梦数据库 | 8.x | 国产数据库、满足需求、支持触发器和时间戳 |

## 三、架构设计

### 1. 整体架构图

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│    Web前端 (Vue3)   │────▶│  Java后端服务       │────▶│  达梦数据库集群     │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
         │                           │                           │
         │ 任务配置/监控              │ 增量数据处理              │ 源库(SRC_SCHEMA)
         │                           │                           │
         │                           │                           │ 目标库(DST_SCHEMA)
         │                           │                           │
         │                           │                           │ 元数据库(META_DB)
```

### 2. 前端架构设计

#### 技术栈
- 框架：Vue 3
- 构建工具：Vite
- 状态管理：Pinia
- 路由：Vue Router
- UI组件库：Element Plus
- HTTP客户端：Axios

#### 工程化方案
- 模块化：按功能模块划分代码结构
- 组件化：封装通用组件（表格、表单、日志等）
- 规范化：统一代码风格、提交规范、命名规范

#### 性能优化策略
- 路由懒加载
- 组件按需加载
- 接口请求缓存
- 虚拟滚动表格

### 3. 后端架构设计

#### 服务分层
```
┌─────────────────────┐
│    Controller层     │   # REST API接口层
├─────────────────────┤
│    Service层        │   # 业务逻辑层
├─────────────────────┤
│    DAO层            │   # 数据访问层
└─────────────────────┘
```

#### 核心模块
- **任务配置模块**：管理同步任务的配置信息
- **增量捕获模块**：捕获源库中的增量数据
- **数据同步模块**：将增量数据同步到目标库
- **任务调度模块**：支持手动和定时触发同步任务
- **日志管理模块**：记录同步过程和结果

#### API设计规范
- 采用RESTful风格
- 统一响应格式：
  ```json
  {
    "code": 200,
    "message": "success",
    "data": {}
  }
  ```
- 分页查询统一参数：`pageNum`、`pageSize`

### 4. 数据库设计

#### 元数据库表结构

##### 1. 同步任务表 (sync_task)
| 字段名 | 数据类型 | 描述 |
|--------|----------|------|
| task_id | VARCHAR(32) | 任务ID，主键 |
| task_name | VARCHAR(100) | 任务名称 |
| src_db | VARCHAR(50) | 源库连接名 |
| src_schema | VARCHAR(50) | 源库SCHEMA |
| src_table | VARCHAR(100) | 源表名 |
| dst_db | VARCHAR(50) | 目标库连接名 |
| dst_schema | VARCHAR(50) | 目标库SCHEMA |
| dst_table | VARCHAR(100) | 目标表名 |
| unique_key | VARCHAR(200) | 唯一键字段列表(逗号分隔) |
| timestamp_field | VARCHAR(100) | 时间戳字段名 |
| sync_type | VARCHAR(20) | 同步类型(FULL/INCREMENTAL) |
| cron_expression | VARCHAR(50) | 定时任务表达式 |
| status | VARCHAR(20) | 任务状态(ENABLED/DISABLED) |
| create_time | TIMESTAMP | 创建时间 |
| update_time | TIMESTAMP | 更新时间 |

##### 2. 同步任务执行记录 (sync_task_log)
| 字段名 | 数据类型 | 描述 |
|--------|----------|------|
| log_id | VARCHAR(32) | 日志ID，主键 |
| task_id | VARCHAR(32) | 任务ID，外键 |
| sync_time | TIMESTAMP | 同步时间 |
| start_time | TIMESTAMP | 开始时间 |
| end_time | TIMESTAMP | 结束时间 |
| status | VARCHAR(20) | 执行状态(SUCCESS/FAILED/RUNNING) |
| total_count | INT | 总记录数 |
| success_count | INT | 成功记录数 |
| failed_count | INT | 失败记录数 |
| error_message | TEXT | 错误信息 |

##### 3. 源库变更日志表 (sync_change_log)
| 字段名 | 数据类型 | 描述 |
|--------|----------|------|
| log_id | VARCHAR(32) | 日志ID，主键 |
| table_name | VARCHAR(100) | 表名 |
| operation_type | VARCHAR(10) | 操作类型(INSERT/UPDATE/DELETE) |
| unique_key_value | VARCHAR(500) | 唯一键值 |
| change_time | TIMESTAMP | 变更时间 |
| sync_status | VARCHAR(20) | 同步状态(PENDING/SYNCED/FAILED) |
| sync_time | TIMESTAMP | 同步时间 |

#### 源库表结构增强
- 为源表添加时间戳字段（如`update_time`），用于记录数据变更时间
- 创建触发器，在数据增删改时更新时间戳字段并记录变更到`sync_change_log`表

## 四、核心实现方案

### 1. 增量捕获实现

#### 1.1 源表触发器创建
```sql
-- 创建INSERT触发器
CREATE TRIGGER trg_${table_name}_insert
AFTER INSERT ON ${schema}.${table_name}
FOR EACH ROW
BEGIN
    INSERT INTO sync_change_log(log_id, table_name, operation_type, unique_key_value, change_time, sync_status)
    VALUES(UUID(), '${table_name}', 'INSERT', ${unique_key_values}, CURRENT_TIMESTAMP, 'PENDING');
END;

-- 创建UPDATE触发器
CREATE TRIGGER trg_${table_name}_update
AFTER UPDATE ON ${schema}.${table_name}
FOR EACH ROW
BEGIN
    UPDATE ${schema}.${table_name} SET update_time = CURRENT_TIMESTAMP WHERE ${unique_key_conditions};
    INSERT INTO sync_change_log(log_id, table_name, operation_type, unique_key_value, change_time, sync_status)
    VALUES(UUID(), '${table_name}', 'UPDATE', ${unique_key_values}, CURRENT_TIMESTAMP, 'PENDING');
END;

-- 创建DELETE触发器
CREATE TRIGGER trg_${table_name}_delete
AFTER DELETE ON ${schema}.${table_name}
FOR EACH ROW
BEGIN
    INSERT INTO sync_change_log(log_id, table_name, operation_type, unique_key_value, change_time, sync_status)
    VALUES(UUID(), '${table_name}', 'DELETE', ${unique_key_values}, CURRENT_TIMESTAMP, 'PENDING');
END;
```

#### 1.2 增量数据查询
```sql
SELECT * FROM ${schema}.${table_name}
WHERE update_time > ?
ORDER BY update_time ASC;
```

### 2. 增量同步策略

#### 2.1 基于唯一键的同步逻辑
```java
// 1. 获取源库增量数据
List<Map<String, Object>> srcDataList = srcDao.getIncrementalData(task, lastSyncTime);

// 2. 遍历增量数据进行同步
for (Map<String, Object> srcData : srcDataList) {
    // 3. 基于唯一键构建查询条件
    String uniqueKeyValue = buildUniqueKeyValue(srcData, task.getUniqueKey());
    Map<String, Object> dstData = dstDao.getDataByUniqueKey(task, uniqueKeyValue);
    
    // 4. 执行同步操作
    if (dstData == null) {
        // 目标库不存在，执行插入
        dstDao.insertData(task, srcData);
    } else {
        // 目标库存在，执行更新
        dstDao.updateData(task, srcData, uniqueKeyValue);
    }
}

// 5. 处理删除操作
List<String> deleteUniqueKeys = srcDao.getDeletedKeys(task, lastSyncTime);
for (String uniqueKey : deleteUniqueKeys) {
    dstDao.deleteData(task, uniqueKey);
}
```

#### 2.2 批量处理优化
- 使用MyBatis-Plus的批量插入/更新功能
- 设置合理的批量处理大小（如1000条/批）
- 开启数据库事务，确保批量操作的原子性

### 3. Web界面实现

#### 3.1 功能模块

| 模块 | 功能描述 | 主要页面 |
|------|----------|----------|
| 任务管理 | 配置和管理同步任务 | 任务列表、任务详情、任务配置 |
| 同步操作 | 手动触发和定时同步 | 同步触发、同步状态监控 |
| 日志管理 | 查看同步日志 | 任务执行日志、数据变更日志 |
| 系统管理 | 数据库连接配置、用户管理 | 数据库连接配置、用户管理 |

#### 3.2 页面设计

- **任务配置页面**：表单形式配置同步任务参数
- **任务列表页面**：表格形式展示所有同步任务，支持筛选和搜索
- **同步监控页面**：实时展示同步任务执行状态，包括进度条、成功/失败计数
- **日志查询页面**：表格形式展示同步日志，支持按任务、时间、状态筛选

## 五、部署架构

### 1. 单机部署
```
┌─────────────────────┐
│  Web服务器 (Nginx)  │
├─────────────────────┤
│  Java应用 (Tomcat)  │
├─────────────────────┤
│  达梦数据库         │
│  - 源库             │
│  - 目标库           │
│  - 元数据库         │
└─────────────────────┘
```

### 2. 集群部署（可选）
```
┌─────────────────────┐
│  Nginx负载均衡      │
├─────────────────────┤
│  Java应用集群       │
│  - 应用节点1        │
│  - 应用节点2        │
├─────────────────────┤
│  达梦数据库集群     │
│  - 源库集群         │
│  - 目标库集群       │
│  - 元数据库集群     │
└─────────────────────┘
```

## 六、监控与维护

### 1. 监控指标
- 任务执行成功率
- 同步延迟时间
- 数据同步吞吐量
- 系统资源利用率

### 2. 日志管理
- 应用日志：记录系统运行状态和错误信息
- 同步日志：记录每次同步的详细信息
- 变更日志：记录源库数据的变更历史

### 3. 异常处理
- 数据库连接异常：重试机制
- 数据同步异常：记录失败原因，支持重试
- 系统异常：告警通知，自动恢复

## 七、性能优化

### 1. 数据库优化
- 为源表的时间戳字段创建索引
- 为目标表的唯一键创建索引
- 使用批量操作减少数据库交互次数

### 2. 应用优化
- 异步处理同步任务，提高系统并发能力
- 使用连接池管理数据库连接
- 缓存常用配置信息，减少数据库查询

### 3. 网络优化
- 减少数据传输量，只同步需要的字段
- 使用压缩技术减少网络传输开销

## 八、安全性设计

### 1. 数据安全
- 数据库连接信息加密存储
- 数据传输加密
- 敏感数据脱敏

### 2. 访问控制
- 基于角色的权限管理
- 操作日志审计
- 接口请求频率限制

### 3. 系统安全
- 定期备份元数据库
- 防SQL注入攻击
- 防XSS攻击

## 九、架构演进路线

### 1. 短期（1-3个月）
- 实现核心功能：任务配置、手动增量同步、同步日志
- 支持基本的Web界面操作

### 2. 中期（3-6个月）
- 增加定时同步功能
- 优化同步性能，支持批量处理
- 完善监控和告警机制

### 3. 长期（6个月以上）
- 支持更复杂的同步规则（如字段映射、数据转换）
- 实现分布式部署，提高系统扩展性
- 支持多源库到多目标库的同步
- 增加数据一致性校验功能
