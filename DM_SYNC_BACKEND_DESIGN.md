# 达梦数据库增量同步系统Java后端设计

## 一、后端架构概述

### 1. 技术栈

| 技术 | 版本 | 用途 | 选型理由 |
|------|------|------|----------|
| Spring Boot | 3.x | 后端框架 | 快速开发、自动配置、生态丰富 |
| MyBatis-Plus | 3.5.x | ORM框架 | 对MyBatis的增强、简化开发、支持达梦数据库 |
| Spring Data Redis | 3.x | 缓存框架 | 支持分布式锁、缓存API响应 |
| Quartz | 2.3.x | 任务调度 | 成熟稳定、支持复杂调度规则 |
| Spring Security | 6.x | 安全框架 | 提供认证与授权功能 |
| JWT | - | 认证机制 | 无状态认证、便于分布式部署 |
| HikariCP | 4.x | 数据库连接池 | 高性能、轻量级 |
| Slf4j + Logback | - | 日志框架 | 灵活配置、性能优异 |
| Swagger3 | 2.x | API文档 | 自动生成API文档、便于调试 |
| Docker | - | 容器化部署 | 简化部署、环境一致性 |

### 2. 项目结构

```
src/
├── main/
│   ├── java/com/dmsync/
│   │   ├── DmSyncApplication.java  # 应用入口
│   │   ├── config/                 # 配置类
│   │   │   ├── DataSourceConfig.java       # 数据源配置
│   │   │   ├── MyBatisPlusConfig.java      # MyBatis-Plus配置
│   │   │   ├── QuartzConfig.java           # Quartz配置
│   │   │   ├── RedisConfig.java            # Redis配置
│   │   │   ├── SecurityConfig.java         # 安全配置
│   │   │   └── SwaggerConfig.java          # Swagger配置
│   │   ├── controller/             # 控制器层
│   │   │   ├── TaskController.java         # 任务管理API
│   │   │   ├── SyncController.java         # 同步操作API
│   │   │   ├── LogController.java          # 日志管理API
│   │   │   └── SystemController.java       # 系统管理API
│   │   ├── service/               # 服务层
│   │   │   ├── task/             # 任务管理服务
│   │   │   │   ├── SyncTaskService.java       # 同步任务服务
│   │   │   │   └── impl/                      # 实现类
│   │   │   ├── sync/             # 同步操作服务
│   │   │   │   ├── IncrementalSyncService.java # 增量同步服务
│   │   │   │   ├── FullSyncService.java        # 全量同步服务
│   │   │   │   └── impl/                      # 实现类
│   │   │   ├── log/              # 日志管理服务
│   │   │   │   ├── SyncLogService.java         # 同步日志服务
│   │   │   │   ├── ChangeLogService.java       # 变更日志服务
│   │   │   │   └── impl/                      # 实现类
│   │   │   ├── system/           # 系统管理服务
│   │   │   │   ├── DataSourceService.java      # 数据源服务
│   │   │   │   └── impl/                      # 实现类
│   │   │   └── common/           # 通用服务
│   │   │       ├── UniqueKeyService.java       # 唯一键服务
│   │   │       ├── DistributedLockService.java # 分布式锁服务
│   │   │       └── impl/                      # 实现类
│   │   ├── dao/                  # 数据访问层
│   │   │   ├── SyncTaskMapper.java            # 同步任务Mapper
│   │   │   ├── SyncLogMapper.java             # 同步日志Mapper
│   │   │   ├── ChangeLogMapper.java           # 变更日志Mapper
│   │   │   └── DataSourceMapper.java          # 数据源Mapper
│   │   ├── entity/               # 实体类
│   │   │   ├── SyncTask.java                  # 同步任务实体
│   │   │   ├── SyncLog.java                   # 同步日志实体
│   │   │   ├── ChangeLog.java                 # 变更日志实体
│   │   │   └── DataSource.java                # 数据源实体
│   │   ├── dto/                  # 数据传输对象
│   │   │   ├── task/             # 任务相关DTO
│   │   │   ├── sync/             # 同步相关DTO
│   │   │   ├── log/              # 日志相关DTO
│   │   │   └── system/           # 系统相关DTO
│   │   ├── vo/                   # 视图对象
│   │   │   ├── task/             # 任务相关VO
│   │   │   ├── sync/             # 同步相关VO
│   │   │   ├── log/              # 日志相关VO
│   │   │   └── system/           # 系统相关VO
│   │   ├── enums/                # 枚举类
│   │   │   ├── SyncTypeEnum.java              # 同步类型枚举
│   │   │   ├── SyncStatusEnum.java            # 同步状态枚举
│   │   │   ├── OperationTypeEnum.java         # 操作类型枚举
│   │   │   └── DataSourceTypeEnum.java        # 数据源类型枚举
│   │   ├── exception/            # 异常类
│   │   │   ├── BusinessException.java         # 业务异常
│   │   │   └── GlobalExceptionHandler.java    # 全局异常处理器
│   │   ├── utils/                # 工具类
│   │   │   ├── UniqueKeyUtil.java             # 唯一键工具
│   │   │   ├── DateUtil.java                  # 日期工具
│   │   │   ├── JwtUtil.java                   # JWT工具
│   │   │   └── SqlUtil.java                   # SQL工具
│   │   └── schedule/             # 定时任务
│   │       ├── SyncTaskJob.java               # 同步任务Job
│   │       └── JobListener.java               # Job监听器
│   └── resources/
│       ├── application.yml        # 应用配置
│       ├── application-dev.yml    # 开发环境配置
│       ├── application-prod.yml   # 生产环境配置
│       ├── mapper/                # MyBatis映射文件
│       │   ├── SyncTaskMapper.xml             # 同步任务映射文件
│       │   ├── SyncLogMapper.xml              # 同步日志映射文件
│       │   ├── ChangeLogMapper.xml            # 变更日志映射文件
│       │   └── DataSourceMapper.xml           # 数据源映射文件
│       └── static/                # 静态资源
└── test/                          # 测试代码
    ├── java/com/dmsync/           # 单元测试
    └── resources/                 # 测试资源
```

## 二、核心模块设计

### 1. 任务配置模块

#### 1.1 同步任务实体 (SyncTask.java)

```java
@Data
@TableName("sync_task")
public class SyncTask {
    @TableId(type = IdType.ASSIGN_UUID)
    private String taskId;
    
    private String taskName;
    
    private String srcDb;
    
    private String srcSchema;
    
    private String srcTable;
    
    private String dstDb;
    
    private String dstSchema;
    
    private String dstTable;
    
    private String uniqueKey;
    
    private String timestampField;
    
    @EnumValue
    private SyncTypeEnum syncType;
    
    private String cronExpression;
    
    @EnumValue
    private SyncStatusEnum status;
    
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Date updateTime;
    
    @TableField(exist = false)
    private DataSource srcDataSource;
    
    @TableField(exist = false)
    private DataSource dstDataSource;
}
```

#### 1.2 同步任务服务 (SyncTaskService.java)

```java
public interface SyncTaskService extends IService<SyncTask> {
    /**
     * 获取任务列表
     * @param params 查询参数
     * @return 分页结果
     */
    Page<SyncTaskVO> getTaskList(TaskQueryDTO params);
    
    /**
     * 获取任务详情
     * @param taskId 任务ID
     * @return 任务详情
     */
    SyncTaskVO getTaskDetail(String taskId);
    
    /**
     * 创建任务
     * @param dto 任务创建DTO
     * @return 任务ID
     */
    String createTask(TaskCreateDTO dto);
    
    /**
     * 更新任务
     * @param dto 任务更新DTO
     */
    void updateTask(TaskUpdateDTO dto);
    
    /**
     * 删除任务
     * @param taskId 任务ID
     */
    void deleteTask(String taskId);
    
    /**
     * 更新任务状态
     * @param taskId 任务ID
     * @param status 任务状态
     */
    void updateTaskStatus(String taskId, SyncStatusEnum status);
    
    /**
     * 生成触发器和时间戳字段
     * @param task 同步任务
     */
    void generateTriggersAndTimestamp(SyncTask task);
    
    /**
     * 删除触发器
     * @param task 同步任务
     */
    void deleteTriggers(SyncTask task);
}
```

#### 1.3 同步任务实现类 (SyncTaskServiceImpl.java)

```java
@Service
@Transactional(rollbackFor = Exception.class)
public class SyncTaskServiceImpl extends ServiceImpl<SyncTaskMapper, SyncTask> implements SyncTaskService {
    
    @Autowired
    private SyncTaskMapper syncTaskMapper;
    
    @Autowired
    private DataSourceService dataSourceService;
    
    @Autowired
    private SqlUtil sqlUtil;
    
    @Override
    public Page<SyncTaskVO> getTaskList(TaskQueryDTO params) {
        Page<SyncTask> page = new Page<>(params.getPageNum(), params.getPageSize());
        QueryWrapper<SyncTask> wrapper = new QueryWrapper<>();
        
        // 构建查询条件
        if (StringUtils.isNotBlank(params.getTaskName())) {
            wrapper.like("task_name", params.getTaskName());
        }
        if (params.getStatus() != null) {
            wrapper.eq("status", params.getStatus());
        }
        if (params.getSyncType() != null) {
            wrapper.eq("sync_type", params.getSyncType());
        }
        
        Page<SyncTask> taskPage = syncTaskMapper.selectPage(page, wrapper);
        
        // 转换为VO
        return taskPage.convert(task -> {
            SyncTaskVO vo = new SyncTaskVO();
            BeanUtils.copyProperties(task, vo);
            return vo;
        });
    }
    
    @Override
    public void generateTriggersAndTimestamp(SyncTask task) {
        // 获取源数据源配置
        DataSource srcDs = dataSourceService.getById(task.getSrcDb());
        
        // 1. 添加时间戳字段
        String addTimestampSql = sqlUtil.generateAddTimestampSql(
            task.getSrcSchema(), task.getSrcTable(), task.getTimestampField());
        sqlUtil.executeSql(srcDs, addTimestampSql);
        
        // 2. 创建触发器
        String createTriggerSql = sqlUtil.generateTriggerSql(
            task.getSrcSchema(), task.getSrcTable(), task.getUniqueKey(), task.getTimestampField());
        sqlUtil.executeSql(srcDs, createTriggerSql);
    }
    
    // 其他方法实现...
}
```

### 2. 增量捕获模块

#### 2.1 唯一键服务 (UniqueKeyService.java)

```java
public interface UniqueKeyService {
    /**
     * 构建唯一键值字符串
     * @param data 数据记录
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @return 唯一键值字符串
     */
    String buildUniqueKeyValue(Map<String, Object> data, String uniqueKey);
    
    /**
     * 基于唯一键构建查询条件
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @param uniqueKeyValue 唯一键值字符串
     * @return 查询条件
     */
    String buildUniqueKeyCondition(String uniqueKey, String uniqueKeyValue);
    
    /**
     * 解析复合唯一键值
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @param uniqueKeyValue 唯一键值字符串
     * @return 唯一键字段和值的映射
     */
    Map<String, Object> parseUniqueKeyValue(String uniqueKey, String uniqueKeyValue);
}
```

#### 2.2 SQL工具类 (SqlUtil.java)

```java
@Component
public class SqlUtil {
    
    /**
     * 生成添加时间戳字段的SQL
     * @param schema SCHEMA名称
     * @param table 表名
     * @param timestampField 时间戳字段名
     * @return SQL语句
     */
    public String generateAddTimestampSql(String schema, String table, String timestampField) {
        StringBuilder sql = new StringBuilder();
        sql.append("ALTER TABLE ").append(schema).append(".").append(table);
        sql.append(" ADD COLUMN ").append(timestampField).append(" TIMESTAMP DEFAULT CURRENT_TIMESTAMP;");
        return sql.toString();
    }
    
    /**
     * 生成触发器SQL
     * @param schema SCHEMA名称
     * @param table 表名
     * @param uniqueKey 唯一键
     * @param timestampField 时间戳字段名
     * @return SQL语句
     */
    public String generateTriggerSql(String schema, String table, String uniqueKey, String timestampField) {
        StringBuilder sql = new StringBuilder();
        
        // 生成INSERT触发器
        sql.append("CREATE TRIGGER trg_").append(table).append("_insert\n");
        sql.append("AFTER INSERT ON ").append(schema).append(".").append(table).append("\n");
        sql.append("FOR EACH ROW\nBEGIN\n");
        sql.append("    INSERT INTO sync_meta.sync_change_log(\n");
        sql.append("        log_id, table_name, schema_name, operation_type,\n");
        sql.append("        unique_key_value, change_time, sync_status\n");
        sql.append("    ) VALUES (\n");
        sql.append("        sys_guid(), '").append(table).append("', '").append(schema).append("', 'INSERT',\n");
        sql.append("        ").append(buildUniqueKeyRef(uniqueKey)).append(", CURRENT_TIMESTAMP, 'PENDING'\n");
        sql.append("    );
END;\n\n");
        
        // 生成UPDATE触发器
        sql.append("CREATE TRIGGER trg_").append(table).append("_update\n");
        sql.append("AFTER UPDATE ON ").append(schema).append(".").append(table).append("\n");
        sql.append("FOR EACH ROW\nBEGIN\n");
        sql.append("    UPDATE ").append(schema).append(".").append(table).append("\n");
        sql.append("    SET ").append(timestampField).append(" = CURRENT_TIMESTAMP\n");
        sql.append("    WHERE ").append(buildUniqueKeyCondition(uniqueKey)).append(";\n");
        sql.append("    INSERT INTO sync_meta.sync_change_log(\n");
        sql.append("        log_id, table_name, schema_name, operation_type,\n");
        sql.append("        unique_key_value, change_time, sync_status\n");
        sql.append("    ) VALUES (\n");
        sql.append("        sys_guid(), '").append(table).append("', '").append(schema).append("', 'UPDATE',\n");
        sql.append("        ").append(buildUniqueKeyRef(uniqueKey)).append(", CURRENT_TIMESTAMP, 'PENDING'\n");
        sql.append("    );
END;\n\n");
        
        // 生成DELETE触发器
        sql.append("CREATE TRIGGER trg_").append(table).append("_delete\n");
        sql.append("AFTER DELETE ON ").append(schema).append(".").append(table).append("\n");
        sql.append("FOR EACH ROW\nBEGIN\n");
        sql.append("    INSERT INTO sync_meta.sync_change_log(\n");
        sql.append("        log_id, table_name, schema_name, operation_type,\n");
        sql.append("        unique_key_value, change_time, sync_status\n");
        sql.append("    ) VALUES (\n");
        sql.append("        sys_guid(), '").append(table).append("', '").append(schema).append("', 'DELETE',\n");
        sql.append("        ").append(buildUniqueKeyRef(uniqueKey)).append(", CURRENT_TIMESTAMP, 'PENDING'\n");
        sql.append("    );
END;");
        
        return sql.toString();
    }
    
    // 其他方法实现...
}
```

### 3. 数据同步模块

#### 3.1 增量同步服务 (IncrementalSyncService.java)

```java
public interface IncrementalSyncService {
    /**
     * 执行增量同步
     * @param task 同步任务
     * @param lastSyncTime 上次同步时间
     * @return 同步结果
     */
    SyncResultDTO executeIncrementalSync(SyncTask task, Date lastSyncTime);
    
    /**
     * 同步单条增量数据
     * @param task 同步任务
     * @param srcData 源库数据
     * @param syncLog 同步日志
     * @return 同步结果
     */
    SyncItemResult syncSingleData(SyncTask task, Map<String, Object> srcData, SyncLog syncLog);
    
    /**
     * 同步删除操作
     * @param task 同步任务
     * @param deleteLogs 删除操作日志
     * @return 同步结果列表
     */
    List<SyncItemResult> syncDeleteOperations(SyncTask task, List<ChangeLog> deleteLogs);
    
    /**
     * 批量同步数据
     * @param task 同步任务
     * @param dataList 数据列表
     * @param syncLog 同步日志
     * @return 同步结果
     */
    BatchSyncResult syncBatchData(SyncTask task, List<Map<String, Object>> dataList, SyncLog syncLog);
}
```

#### 3.2 增量同步实现类 (IncrementalSyncServiceImpl.java)

```java
@Service
public class IncrementalSyncServiceImpl implements IncrementalSyncService {
    
    private static final Logger logger = LoggerFactory.getLogger(IncrementalSyncServiceImpl.class);
    
    @Autowired
    private SyncLogService syncLogService;
    
    @Autowired
    private ChangeLogService changeLogService;
    
    @Autowired
    private UniqueKeyService uniqueKeyService;
    
    @Autowired
    private DistributedLockService distributedLockService;
    
    @Override
    public SyncResultDTO executeIncrementalSync(SyncTask task, Date lastSyncTime) {
        // 1. 获取分布式锁
        String lockKey = "sync:task:" + task.getTaskId();
        boolean locked = distributedLockService.lock(lockKey, 300); // 5分钟锁过期
        
        if (!locked) {
            throw new BusinessException("任务正在执行中，请稍后重试");
        }
        
        SyncLog syncLog = null;
        
        try {
            // 2. 创建同步日志
            syncLog = syncLogService.createSyncLog(task, SyncTypeEnum.INCREMENTAL);
            
            // 3. 查询源库增量数据
            List<Map<String, Object>> srcDataList = queryIncrementalData(task, lastSyncTime);
            
            // 4. 批量同步数据
            BatchSyncResult batchResult = syncBatchData(task, srcDataList, syncLog);
            
            // 5. 处理删除操作
            List<ChangeLog> deleteLogs = changeLogService.getDeletedLogs(task, lastSyncTime);
            List<SyncItemResult> deleteResults = syncDeleteOperations(task, deleteLogs);
            
            // 6. 更新同步日志
            syncLog.setTotalCount(srcDataList.size() + deleteLogs.size());
            syncLog.setSuccessCount(batchResult.getSuccessCount() + deleteResults.stream().filter(SyncItemResult::isSuccess).count());
            syncLog.setFailedCount(batchResult.getFailedCount() + deleteResults.stream().filter(result -> !result.isSuccess()).count());
            syncLog.setStatus(SyncStatusEnum.SUCCESS);
            syncLog.setEndTime(new Date());
            syncLogService.updateById(syncLog);
            
            // 7. 更新变更日志状态
            changeLogService.updateChangeLogStatus(srcDataList, task, SyncStatusEnum.SYNCED);
            changeLogService.updateChangeLogStatus(deleteLogs, SyncStatusEnum.SYNCED);
            
            // 8. 返回同步结果
            SyncResultDTO result = new SyncResultDTO();
            BeanUtils.copyProperties(syncLog, result);
            return result;
            
        } catch (Exception e) {
            logger.error("增量同步失败，任务ID：{}", task.getTaskId(), e);
            
            if (syncLog != null) {
                syncLog.setStatus(SyncStatusEnum.FAILED);
                syncLog.setErrorMessage(e.getMessage());
                syncLog.setEndTime(new Date());
                syncLogService.updateById(syncLog);
            }
            
            throw new BusinessException("增量同步失败：" + e.getMessage());
            
        } finally {
            // 释放分布式锁
            distributedLockService.unlock(lockKey);
        }
    }
    
    // 其他方法实现...
}
```

### 4. 任务调度模块

#### 4.1 同步任务Job (SyncTaskJob.java)

```java
public class SyncTaskJob implements Job {
    
    private static final Logger logger = LoggerFactory.getLogger(SyncTaskJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // 获取任务ID
        String taskId = (String) context.getJobDetail().getJobDataMap().get("taskId");
        
        logger.info("开始执行同步任务，任务ID：{}", taskId);
        
        try {
            // 获取Spring上下文
            ApplicationContext applicationContext = ApplicationContextHolder.getApplicationContext();
            
            // 获取服务
            SyncTaskService syncTaskService = applicationContext.getBean(SyncTaskService.class);
            SyncController syncController = applicationContext.getBean(SyncController.class);
            
            // 获取任务信息
            SyncTask task = syncTaskService.getById(taskId);
            
            if (task == null) {
                logger.error("同步任务不存在，任务ID：{}", taskId);
                return;
            }
            
            if (task.getStatus() != SyncStatusEnum.ENABLED) {
                logger.info("同步任务已禁用，任务ID：{}", taskId);
                return;
            }
            
            // 触发同步
            SyncTriggerDTO triggerDTO = new SyncTriggerDTO();
            triggerDTO.setTaskId(taskId);
            triggerDTO.setSyncType(SyncTypeEnum.INCREMENTAL);
            
            syncController.triggerSync(triggerDTO);
            
            logger.info("同步任务执行完成，任务ID：{}", taskId);
            
        } catch (Exception e) {
            logger.error("同步任务执行失败，任务ID：{}", taskId, e);
            throw new JobExecutionException("同步任务执行失败", e, false);
        }
    }
}
```

#### 4.2 任务调度服务 (ScheduleService.java)

```java
public interface ScheduleService {
    /**
     * 创建定时任务
     * @param task 同步任务
     */
    void createScheduleTask(SyncTask task);
    
    /**
     * 更新定时任务
     * @param task 同步任务
     */
    void updateScheduleTask(SyncTask task);
    
    /**
     * 删除定时任务
     * @param taskId 任务ID
     */
    void deleteScheduleTask(String taskId);
    
    /**
     * 暂停定时任务
     * @param taskId 任务ID
     */
    void pauseScheduleTask(String taskId);
    
    /**
     * 恢复定时任务
     * @param taskId 任务ID
     */
    void resumeScheduleTask(String taskId);
    
    /**
     * 立即执行定时任务
     * @param taskId 任务ID
     */
    void executeScheduleTask(String taskId);
}
```

### 5. 日志管理模块

#### 5.1 同步日志实体 (SyncLog.java)

```java
@Data
@TableName("sync_log")
public class SyncLog {
    @TableId(type = IdType.ASSIGN_UUID)
    private String logId;
    
    private String taskId;
    
    private String taskName;
    
    @EnumValue
    private SyncTypeEnum syncType;
    
    private Date syncTime;
    
    private Date startTime;
    
    private Date endTime;
    
    @EnumValue
    private SyncStatusEnum status;
    
    private Integer totalCount;
    
    private Integer successCount;
    
    private Integer failedCount;
    
    private String errorMessage;
    
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;
}
```

#### 5.2 变更日志实体 (ChangeLog.java)

```java
@Data
@TableName("sync_change_log")
public class ChangeLog {
    @TableId(type = IdType.ASSIGN_UUID)
    private String logId;
    
    private String tableName;
    
    private String schemaName;
    
    @EnumValue
    private OperationTypeEnum operationType;
    
    private String uniqueKeyValue;
    
    private Date changeTime;
    
    @EnumValue
    private SyncStatusEnum syncStatus;
    
    private Date syncTime;
    
    private String errorMessage;
    
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;
}
```

## 三、数据库访问层设计

### 1. 动态数据源配置 (DynamicDataSourceConfig.java)

```java
@Configuration
public class DynamicDataSourceConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.dynamic.datasource.meta")
    public DataSource metaDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DynamicDataSourceProvider dynamicDataSourceProvider() {
        return new DynamicDataSourceProvider() {
            @Override
            public Map<String, DataSource> loadDataSources() {
                Map<String, DataSource> dataSourceMap = new HashMap<>();
                // 加载元数据库
                dataSourceMap.put("meta", metaDataSource());
                return dataSourceMap;
            }
        };
    }
    
    @Bean
    public DataSource dynamicDataSource(DynamicDataSourceProvider dynamicDataSourceProvider) {
        DynamicRoutingDataSource dataSource = new DynamicRoutingDataSource();
        dataSource.setPrimary("meta");
        dataSource.setDataSourceProvider(dynamicDataSourceProvider);
        return dataSource;
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 2. 数据访问工具类 (DbAccessUtil.java)

```java
@Component
public class DbAccessUtil {
    
    @Autowired
    private DynamicRoutingDataSource dynamicDataSource;
    
    /**
     * 获取数据源
     * @param dataSource 数据源配置
     * @return 数据源
     */
    public DataSource getDataSource(DataSource dataSource) {
        // 构建数据源
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(dataSource.getDriverClassName());
        ds.setUrl(dataSource.getUrl());
        ds.setUsername(dataSource.getUsername());
        ds.setPassword(dataSource.getPassword());
        
        // 设置连接池参数
        ds.setInitialSize(5);
        ds.setMinIdle(5);
        ds.setMaxActive(20);
        ds.setMaxWait(60000);
        
        return ds;
    }
    
    /**
     * 执行查询SQL
     * @param dataSource 数据源
     * @param sql SQL语句
     * @param params 参数
     * @return 查询结果列表
     */
    public List<Map<String, Object>> executeQuery(DataSource dataSource, String sql, Object... params) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(getDataSource(dataSource));
        return jdbcTemplate.queryForList(sql, params);
    }
    
    /**
     * 执行更新SQL
     * @param dataSource 数据源
     * @param sql SQL语句
     * @param params 参数
     * @return 影响行数
     */
    public int executeUpdate(DataSource dataSource, String sql, Object... params) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(getDataSource(dataSource));
        return jdbcTemplate.update(sql, params);
    }
    
    /**
     * 批量执行SQL
     * @param dataSource 数据源
     * @param sql SQL语句
     * @param batchArgs 批量参数
     * @return 影响行数数组
     */
    public int[] executeBatch(DataSource dataSource, String sql, List<Object[]> batchArgs) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(getDataSource(dataSource));
        return jdbcTemplate.batchUpdate(sql, batchArgs);
    }
}
```

## 四、API接口设计

### 1. 任务管理API (TaskController.java)

```java
@RestController
@RequestMapping("/api/task")
@Api(tags = "同步任务管理")
public class TaskController {
    
    @Autowired
    private SyncTaskService syncTaskService;
    
    @GetMapping("/list")
    @ApiOperation("获取任务列表")
    public Result<Page<SyncTaskVO>> getTaskList(TaskQueryDTO params) {
        Page<SyncTaskVO> page = syncTaskService.getTaskList(params);
        return Result.success(page);
    }
    
    @GetMapping("/{taskId}")
    @ApiOperation("获取任务详情")
    public Result<SyncTaskVO> getTaskDetail(@PathVariable String taskId) {
        SyncTaskVO detail = syncTaskService.getTaskDetail(taskId);
        return Result.success(detail);
    }
    
    @PostMapping
    @ApiOperation("创建任务")
    public Result<String> createTask(@RequestBody TaskCreateDTO dto) {
        String taskId = syncTaskService.createTask(dto);
        return Result.success(taskId, "任务创建成功");
    }
    
    @PutMapping
    @ApiOperation("更新任务")
    public Result<Void> updateTask(@RequestBody TaskUpdateDTO dto) {
        syncTaskService.updateTask(dto);
        return Result.success("任务更新成功");
    }
    
    @DeleteMapping("/{taskId}")
    @ApiOperation("删除任务")
    public Result<Void> deleteTask(@PathVariable String taskId) {
        syncTaskService.deleteTask(taskId);
        return Result.success("任务删除成功");
    }
    
    @PutMapping("/{taskId}/status")
    @ApiOperation("更新任务状态")
    public Result<Void> updateTaskStatus(@PathVariable String taskId, @RequestParam SyncStatusEnum status) {
        syncTaskService.updateTaskStatus(taskId, status);
        return Result.success("任务状态更新成功");
    }
}
```

### 2. 同步操作API (SyncController.java)

```java
@RestController
@RequestMapping("/api/sync")
@Api(tags = "同步操作")
public class SyncController {
    
    @Autowired
    private SyncTaskService syncTaskService;
    
    @Autowired
    private IncrementalSyncService incrementalSyncService;
    
    @Autowired
    private FullSyncService fullSyncService;
    
    @Autowired
    private SyncLogService syncLogService;
    
    @PostMapping("/trigger")
    @ApiOperation("触发同步")
    public Result<SyncResultDTO> triggerSync(@RequestBody SyncTriggerDTO dto) {
        // 获取任务信息
        SyncTask task = syncTaskService.getById(dto.getTaskId());
        
        if (task == null) {
            throw new BusinessException("任务不存在");
        }
        
        SyncResultDTO result;
        
        if (dto.getSyncType() == SyncTypeEnum.FULL) {
            // 执行全量同步
            result = fullSyncService.executeFullSync(task);
        } else {
            // 执行增量同步
            Date lastSyncTime = syncLogService.getLastSyncTime(task.getTaskId());
            result = incrementalSyncService.executeIncrementalSync(task, lastSyncTime);
        }
        
        return Result.success(result, "同步触发成功");
    }
    
    @GetMapping("/running")
    @ApiOperation("获取运行中的任务")
    public Result<List<SyncLogVO>> getRunningTasks() {
        List<SyncLogVO> runningTasks = syncLogService.getRunningTasks();
        return Result.success(runningTasks);
    }
    
    @PostMapping("/retry/{logId}")
    @ApiOperation("重试同步")
    public Result<Void> retrySync(@PathVariable String logId) {
        // 重试失败的同步
        syncLogService.retrySync(logId);
        return Result.success("重试成功");
    }
}
```

## 五、配置管理

### 1. 应用配置 (application.yml)

```yaml
# 应用配置
spring:
  application:
    name: dm-sync
  
  # 数据源配置
  datasource:
    dynamic:
      primary: meta
      datasource:
        meta:
          driver-class-name: dm.jdbc.driver.DmDriver
          url: jdbc:dm://localhost:5236/sync_meta
          username: SYSDBA
          password: SYSDBA
  
  # Redis配置
  redis:
    host: localhost
    port: 6379
    password:
    database: 0
    timeout: 3000
  
  # Quartz配置
  quartz:
    job-store-type: jdbc
    properties:
      org:
        quartz:
          scheduler:
            instanceName: dmSyncScheduler
            instanceId: AUTO
          threadPool:
            threadCount: 10
  
  # JPA配置
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: none
  
# MyBatis-Plus配置
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  type-aliases-package: com.dmsync.entity
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  
# 日志配置
logging:
  level:
    com.dmsync: info
    org.springframework: warn
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n"
  file:
    name: logs/dm-sync.log
  
# 端口配置
server:
  port: 8080
  servlet:
    context-path: /
  
# Swagger配置
springdoc:
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: alpha
  api-docs:
    path: /v3/api-docs
  packages-to-scan: com.dmsync.controller
```

## 六、异常处理

### 1. 业务异常类 (BusinessException.java)

```java
public class BusinessException extends RuntimeException {
    private static final long serialVersionUID = 1L;
    
    private String code;
    
    public BusinessException(String message) {
        super(message);
        this.code = "500";
    }
    
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
    
    public String getCode() {
        return code;
    }
    
    public void setCode(String code) {
        this.code = code;
    }
}
```

### 2. 全局异常处理器 (GlobalExceptionHandler.java)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    /**
     * 处理业务异常
     */
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        logger.error("业务异常：{}", e.getMessage());
        return Result.error(e.getCode(), e.getMessage());
    }
    
    /**
     * 处理SQL异常
     */
    @ExceptionHandler(SQLException.class)
    public Result<Void> handleSQLException(SQLException e) {
        logger.error("SQL异常：{}", e.getMessage(), e);
        return Result.error("500", "数据库操作异常：" + e.getMessage());
    }
    
    /**
     * 处理空指针异常
     */
    @ExceptionHandler(NullPointerException.class)
    public Result<Void> handleNullPointerException(NullPointerException e) {
        logger.error("空指针异常：{}", e.getMessage(), e);
        return Result.error("500", "系统异常：空指针");
    }
    
    /**
     * 处理其他异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        logger.error("系统异常：{}", e.getMessage(), e);
        return Result.error("500", "系统异常：" + e.getMessage());
    }
}
```

## 七、安全设计

### 1. Spring Security配置 (SecurityConfig.java)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // 禁用CSRF
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll() // 允许访问认证和Swagger接口
                .anyRequest().authenticated() // 其他接口需要认证
            )
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // 无状态会话
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class); // 添加JWT过滤器
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 2. JWT工具类 (JwtUtil.java)

```java
@Component
public class JwtUtil {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private Long expiration;
    
    /**
     * 生成JWT令牌
     * @param username 用户名
     * @return JWT令牌
     */
    public String generateToken(String username) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);
        
        return Jwts.builder()
                .setSubject(username)
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }
    
    /**
     * 从JWT令牌中获取用户名
     * @param token JWT令牌
     * @return 用户名
     */
    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parser()
                .setSigningKey(secret)
                .parseClaimsJws(token)
                .getBody();
        
        return claims.getSubject();
    }
    
    /**
     * 验证JWT令牌
     * @param token JWT令牌
     * @return 是否有效
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (SignatureException ex) {
            logger.error("无效的JWT签名");
        } catch (MalformedJwtException ex) {
            logger.error("无效的JWT令牌");
        } catch (ExpiredJwtException ex) {
            logger.error("JWT令牌已过期");
        } catch (UnsupportedJwtException ex) {
            logger.error("不支持的JWT令牌");
        } catch (IllegalArgumentException ex) {
            logger.error("JWT令牌参数为空");
        }
        
        return false;
    }
}
```

## 八、总结

本Java后端设计基于Spring Boot 3 + MyBatis-Plus技术栈，实现了达梦数据库增量同步系统的核心功能，包括：

1. **任务配置模块**：管理同步任务的创建、更新、删除和状态管理
2. **增量捕获模块**：基于触发器+时间戳机制捕获源库增量数据
3. **数据同步模块**：实现基于唯一键的增量同步逻辑，支持批量处理
4. **任务调度模块**：基于Quartz实现定时同步功能
5. **日志管理模块**：记录同步任务执行日志和数据变更日志
6. **API接口模块**：提供RESTful API接口，支持Web前端调用

系统采用了分布式锁确保任务并发安全，使用动态数据源支持多数据库连接，实现了完善的异常处理和安全认证机制。整体架构设计合理，性能优化到位，能够满足用户通过Web界面实现达梦数据库不同SCHEMA间表数据增量同步的需求。