# 达梦数据库增量同步系统实现代码示例

## 一、唯一键处理工具类

```java
package com.dmsync.utils;

import org.apache.commons.lang3.StringUtils;

import java.util.HashMap;
import java.util.Map;

/**
 * 唯一键处理工具类
 */
public class UniqueKeyUtil {
    
    /**
     * 唯一键值分隔符
     */
    private static final String UNIQUE_KEY_SEPARATOR = "|";
    
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
                sb.append(UNIQUE_KEY_SEPARATOR);
            }
        }
        
        return sb.toString();
    }
    
    /**
     * 解析复合唯一键值
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @param uniqueKeyValue 唯一键值字符串
     * @return 唯一键字段和值的映射
     */
    public static Map<String, Object> parseUniqueKeyValue(String uniqueKey, String uniqueKeyValue) {
        if (StringUtils.isBlank(uniqueKey) || StringUtils.isBlank(uniqueKeyValue)) {
            throw new IllegalArgumentException("唯一键和唯一键值不能为空");
        }
        
        String[] keyFields = uniqueKey.split(",");
        String[] keyValues = uniqueKeyValue.split(UNIQUE_KEY_SEPARATOR);
        
        if (keyFields.length != keyValues.length) {
            throw new IllegalArgumentException("唯一键字段数量与值数量不匹配");
        }
        
        Map<String, Object> result = new HashMap<>();
        
        for (int i = 0; i < keyFields.length; i++) {
            String field = keyFields[i].trim();
            String value = keyValues[i];
            
            // 处理NULL值
            if ("NULL".equals(value)) {
                result.put(field, null);
            } else {
                result.put(field, value);
            }
        }
        
        return result;
    }
    
    /**
     * 基于唯一键构建查询条件
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @param uniqueKeyValue 唯一键值字符串
     * @return 查询条件字符串
     */
    public static String buildUniqueKeyCondition(String uniqueKey, String uniqueKeyValue) {
        Map<String, Object> keyMap = parseUniqueKeyValue(uniqueKey, uniqueKeyValue);
        
        StringBuilder sb = new StringBuilder();
        int index = 0;
        
        for (Map.Entry<String, Object> entry : keyMap.entrySet()) {
            if (index > 0) {
                sb.append(" AND ");
            }
            
            String field = entry.getKey();
            Object value = entry.getValue();
            
            if (value == null) {
                sb.append(field).append(" IS NULL");
            } else {
                sb.append(field).append(" = '").append(value).append("'");
            }
            
            index++;
        }
        
        return sb.toString();
    }
    
    /**
     * 基于唯一键构建查询条件（用于PreparedStatement）
     * @param uniqueKey 唯一键字段列表(逗号分隔)
     * @return 查询条件字符串和参数列表
     */
    public static UniqueKeyCondition buildPreparedUniqueKeyCondition(String uniqueKey, Map<String, Object> data) {
        if (StringUtils.isBlank(uniqueKey)) {
            throw new IllegalArgumentException("唯一键不能为空");
        }
        
        String[] keyFields = uniqueKey.split(",");
        StringBuilder condition = new StringBuilder();
        UniqueKeyCondition result = new UniqueKeyCondition();
        
        for (int i = 0; i < keyFields.length; i++) {
            String field = keyFields[i].trim();
            
            if (i > 0) {
                condition.append(" AND ");
            }
            
            condition.append(field).append(" = ?");
            result.getParams().add(data.get(field));
        }
        
        result.setCondition(condition.toString());
        return result;
    }
    
    /**
     * 唯一键条件封装类
     */
    public static class UniqueKeyCondition {
        private String condition;
        private java.util.List<Object> params = new java.util.ArrayList<>();
        
        // getter和setter方法
        public String getCondition() {
            return condition;
        }
        
        public void setCondition(String condition) {
            this.condition = condition;
        }
        
        public java.util.List<Object> getParams() {
            return params;
        }
        
        public void setParams(java.util.List<Object> params) {
            this.params = params;
        }
    }
}
```

## 二、增量同步核心实现

```java
package com.dmsync.service.sync.impl;

import com.dmsync.entity.ChangeLog;
import com.dmsync.entity.SyncLog;
import com.dmsync.entity.SyncTask;
import com.dmsync.enums.OperationTypeEnum;
import com.dmsync.enums.SyncStatusEnum;
import com.dmsync.service.changeLog.ChangeLogService;
import com.dmsync.service.sync.IncrementalSyncService;
import com.dmsync.service.syncLog.SyncLogService;
import com.dmsync.utils.DbAccessUtil;
import com.dmsync.utils.UniqueKeyUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;

/**
 * 增量同步服务实现
 */
@Service
public class IncrementalSyncServiceImpl implements IncrementalSyncService {
    
    private static final Logger logger = LoggerFactory.getLogger(IncrementalSyncServiceImpl.class);
    
    @Autowired
    private SyncLogService syncLogService;
    
    @Autowired
    private ChangeLogService changeLogService;
    
    @Autowired
    private DbAccessUtil dbAccessUtil;
    
    /**
     * 批量处理大小
     */
    private static final int BATCH_SIZE = 1000;
    
    @Override
    public SyncResultDTO executeIncrementalSync(SyncTask task, Date lastSyncTime) {
        // 1. 创建同步日志
        SyncLog syncLog = syncLogService.createSyncLog(task);
        
        try {
            logger.info("开始执行增量同步，任务ID：{}", task.getTaskId());
            
            // 2. 查询源库增量数据
            List<Map<String, Object>> incrementalData = queryIncrementalData(task, lastSyncTime);
            logger.info("查询到增量数据：{}条", incrementalData.size());
            
            // 3. 批量同步数据
            BatchSyncResult batchResult = syncBatchData(task, incrementalData, syncLog);
            
            // 4. 处理删除操作
            List<ChangeLog> deleteLogs = changeLogService.getDeleteLogs(task, lastSyncTime);
            List<SyncItemResult> deleteResults = syncDeleteOperations(task, deleteLogs);
            
            // 5. 更新同步日志
            syncLog.setTotalCount(incrementalData.size() + deleteLogs.size());
            syncLog.setSuccessCount(batchResult.getSuccessCount() + (nt) deleteResults.stream().filter(SyncItemResult::isSuccess).count());
            syncLog.setFailedCount(batchResult.getFailedCount() + (nt) deleteResults.stream().filter(result -> !result.isSuccess()).count());
            syncLog.setStatus(SyncStatusEnum.SUCCESS);
            syncLog.setEndTime(new Date());
            syncLogService.updateSyncLog(syncLog);
            
            // 6. 更新变更日志状态
            changeLogService.updateChangeLogStatus(incrementalData, task, SyncStatusEnum.SYNCED);
            changeLogService.updateChangeLogStatus(deleteLogs, SyncStatusEnum.SYNCED);
            
            logger.info("增量同步完成，任务ID：{}", task.getTaskId());
            
            // 7. 构建同步结果
            SyncResultDTO result = new SyncResultDTO();
            result.setLogId(syncLog.getLogId());
            result.setTaskId(task.getTaskId());
            result.setTaskName(task.getTaskName());
            result.setSyncType(task.getSyncType());
            result.setStatus(syncLog.getStatus());
            result.setTotalCount(syncLog.getTotalCount());
            result.setSuccessCount(syncLog.getSuccessCount());
            result.setFailedCount(syncLog.getFailedCount());
            result.setStartTime(syncLog.getStartTime());
            result.setEndTime(syncLog.getEndTime());
            
            return result;
            
        } catch (Exception e) {
            logger.error("增量同步失败，任务ID：{}", task.getTaskId(), e);
            
            // 更新失败日志
            syncLog.setStatus(SyncStatusEnum.FAILED);
            syncLog.setErrorMessage(e.getMessage());
            syncLog.setEndTime(new Date());
            syncLogService.updateSyncLog(syncLog);
            
            throw new BusinessException("增量同步失败：" + e.getMessage());
        }
    }
    
    /**
     * 查询源库增量数据
     * @param task 同步任务
     * @param lastSyncTime 上次同步时间
     * @return 增量数据列表
     */
    private List<Map<String, Object>> queryIncrementalData(SyncTask task, Date lastSyncTime) {
        String sql = String.format(
            "SELECT * FROM %s.%s WHERE %s > ? ORDER BY %s ASC",
            task.getSrcSchema(), task.getSrcTable(), task.getTimestampField(), task.getTimestampField()
        );
        
        return dbAccessUtil.executeQuery(task.getSrcDataSource(), sql, lastSyncTime);
    }
    
    /**
     * 批量同步数据
     * @param task 同步任务
     * @param dataList 数据列表
     * @param syncLog 同步日志
     * @return 批量同步结果
     */
    @Override
    public BatchSyncResult syncBatchData(SyncTask task, List<Map<String, Object>> dataList, SyncLog syncLog) {
        BatchSyncResult result = new BatchSyncResult();
        result.setTotalCount(dataList.size());
        result.setSuccessCount(0);
        result.setFailedCount(0);
        
        if (dataList.isEmpty()) {
            return result;
        }
        
        // 分批处理
        for (int i = 0; i < dataList.size(); i += BATCH_SIZE) {
            int endIndex = Math.min(i + BATCH_SIZE, dataList.size());
            List<Map<String, Object>> batch = dataList.subList(i, endIndex);
            
            try {
                // 执行批量同步
                int[] counts = dbAccessUtil.batchSyncData(task, batch);
                
                // 统计成功和失败数量
                for (int count : counts) {
                    if (count > 0) {
                        result.setSuccessCount(result.getSuccessCount() + 1);
                    } else {
                        result.setFailedCount(result.getFailedCount() + 1);
                    }
                }
                
            } catch (Exception e) {
                logger.error("批量同步失败，批次：{}-{}", i, endIndex, e);
                result.setFailedCount(result.getFailedCount() + batch.size());
            }
        }
        
        return result;
    }
    
    /**
     * 同步删除操作
     * @param task 同步任务
     * @param deleteLogs 删除操作日志
     * @return 同步结果列表
     */
    @Override
    public List<SyncItemResult> syncDeleteOperations(SyncTask task, List<ChangeLog> deleteLogs) {
        List<SyncItemResult> results = new ArrayList<>();
        
        for (ChangeLog deleteLog : deleteLogs) {
            SyncItemResult result = new SyncItemResult();
            result.setOperationType(OperationTypeEnum.DELETE);
            
            try {
                // 构建删除条件
                String condition = UniqueKeyUtil.buildUniqueKeyCondition(
                    task.getUniqueKey(), deleteLog.getUniqueKeyValue());
                
                // 执行删除操作
                String sql = String.format(
                    "DELETE FROM %s.%s WHERE %s",
                    task.getDstSchema(), task.getDstTable(), condition
                );
                
                int count = dbAccessUtil.executeUpdate(task.getDstDataSource(), sql);
                
                result.setSuccess(count > 0);
                if (!result.isSuccess()) {
                    result.setErrorMessage("删除操作影响行数为0");
                }
                
            } catch (Exception e) {
                result.setSuccess(false);
                result.setErrorMessage(e.getMessage());
                logger.error("同步删除操作失败，日志ID：{}", deleteLog.getLogId(), e);
            }
            
            results.add(result);
        }
        
        return results;
    }
}
```

## 三、数据访问工具类

```java
package com.dmsync.utils;

import com.dmsync.entity.SyncTask;
import com.dmsync.entity.DataSource;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 数据库访问工具类
 */
public class DbAccessUtil {
    
    /**
     * 获取数据源连接池
     * @param dataSource 数据源配置
     * @return 数据源连接池
     */
    public HikariDataSource getDataSource(DataSource dataSource) {
        HikariConfig config = new HikariConfig();
        config.setDriverClassName(dataSource.getDriverClassName());
        config.setJdbcUrl(dataSource.getUrl());
        config.setUsername(dataSource.getUsername());
        config.setPassword(dataSource.getPassword());
        
        // 连接池配置
        config.setMinimumIdle(5);
        config.setMaximumPoolSize(20);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        return new HikariDataSource(config);
    }
    
    /**
     * 执行查询操作
     * @param dataSource 数据源配置
     * @param sql SQL语句
     * @param params 参数
     * @return 查询结果列表
     */
    public List<Map<String, Object>> executeQuery(DataSource dataSource, String sql, Object... params) {
        List<Map<String, Object>> result = new ArrayList<>();
        
        try (HikariDataSource ds = getDataSource(dataSource);
             Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            // 设置参数
            setParameters(ps, params);
            
            // 执行查询
            ResultSet rs = ps.executeQuery();
            
            // 处理结果集
            int columnCount = rs.getMetaData().getColumnCount();
            while (rs.next()) {
                Map<String, Object> row = new HashMap<>();
                for (int i = 1; i <= columnCount; i++) {
                    String columnName = rs.getMetaData().getColumnName(i);
                    Object value = rs.getObject(i);
                    row.put(columnName, value);
                }
                result.add(row);
            }
            
        } catch (SQLException e) {
            throw new RuntimeException("SQL查询失败：" + sql, e);
        }
        
        return result;
    }
    
    /**
     * 执行更新操作
     * @param dataSource 数据源配置
     * @param sql SQL语句
     * @param params 参数
     * @return 影响行数
     */
    public int executeUpdate(DataSource dataSource, String sql, Object... params) {
        try (HikariDataSource ds = getDataSource(dataSource);
             Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            // 设置参数
            setParameters(ps, params);
            
            // 执行更新
            return ps.executeUpdate();
            
        } catch (SQLException e) {
            throw new RuntimeException("SQL更新失败：" + sql, e);
        }
    }
    
    /**
     * 批量同步数据到目标库
     * @param task 同步任务
     * @param dataList 数据列表
     * @return 影响行数数组
     */
    public int[] batchSyncData(SyncTask task, List<Map<String, Object>> dataList) {
        if (dataList.isEmpty()) {
            return new int[0];
        }
        
        // 获取字段列表
        Map<String, Object> firstRow = dataList.get(0);
        List<String> fields = new ArrayList<>(firstRow.keySet());
        
        // 构建INSERT/UPDATE语句
        String sql = buildUpsertSql(task, fields);
        
        try (HikariDataSource ds = getDataSource(task.getDstDataSource());
             Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            // 设置批量参数
            for (Map<String, Object> row : dataList) {
                int paramIndex = 1;
                
                // 设置所有字段值
                for (String field : fields) {
                    ps.setObject(paramIndex++, row.get(field));
                }
                
                // 再次设置唯一键字段值（用于UPDATE）
                String[] uniqueKeyFields = task.getUniqueKey().split(",");
                for (String keyField : uniqueKeyFields) {
                    ps.setObject(paramIndex++, row.get(keyField.trim()));
                }
                
                ps.addBatch();
            }
            
            // 执行批量操作
            return ps.executeBatch();
            
        } catch (SQLException e) {
            throw new RuntimeException("批量同步失败：" + sql, e);
        }
    }
    
    /**
     * 构建INSERT/UPDATE语句（根据唯一键判断是插入还是更新）
     * @param task 同步任务
     * @param fields 字段列表
     * @return SQL语句
     */
    private String buildUpsertSql(SyncTask task, List<String> fields) {
        StringBuilder sql = new StringBuilder();
        
        // INSERT部分
        sql.append("INSERT INTO ").append(task.getDstSchema()).append(".").append(task.getDstTable());
        sql.append(" (");
        sql.append(String.join(", ", fields));
        sql.append(") VALUES (");
        
        // 参数占位符
        List<String> placeholders = new ArrayList<>();
        for (int i = 0; i < fields.size(); i++) {
            placeholders.add("?");
        }
        sql.append(String.join(", ", placeholders));
        sql.append(")");
        
        // UPDATE部分（达梦数据库使用MERGE INTO语法）
        sql.append(" MERGE INTO ").append(task.getDstSchema()).append(".").append(task.getDstTable());
        sql.append(" T USING (");
        sql.append("SELECT ").append(String.join(", ", fields));
        sql.append(" FROM dual WHERE 1=0");
        sql.append(") S ON (");
        
        // 构建唯一键条件
        String[] uniqueKeyFields = task.getUniqueKey().split(",");
        List<String> keyConditions = new ArrayList<>();
        for (String keyField : uniqueKeyFields) {
            keyConditions.add("T." + keyField.trim() + " = S." + keyField.trim());
        }
        sql.append(String.join(" AND ", keyConditions));
        sql.append(")");
        
        // UPDATE SET部分
        sql.append(" WHEN MATCHED THEN UPDATE SET ");
        List<String> updateSet = new ArrayList<>();
        for (String field : fields) {
            if (!task.getUniqueKey().contains(field)) {
                updateSet.add("T." + field + " = S." + field);
            }
        }
        sql.append(String.join(", ", updateSet));
        
        // INSERT部分
        sql.append(" WHEN NOT MATCHED THEN INSERT (");
        sql.append(String.join(", ", fields));
        sql.append(") VALUES (");
        sql.append(String.join(", ", placeholders));
        sql.append(")");
        
        return sql.toString();
    }
    
    /**
     * 设置PreparedStatement参数
     * @param ps PreparedStatement
     * @param params 参数列表
     * @throws SQLException SQL异常
     */
    private void setParameters(PreparedStatement ps, Object... params) throws SQLException {
        if (params != null) {
            for (int i = 0; i < params.length; i++) {
                ps.setObject(i + 1, params[i]);
            }
        }
    }
}
```

## 四、同步任务调度实现

```java
package com.dmsync.schedule;

import com.dmsync.entity.SyncTask;
import com.dmsync.service.syncTask.SyncTaskService;
import com.dmsync.service.sync.IncrementalSyncService;
import com.dmsync.utils.ApplicationContextHolder;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Date;

/**
 * 同步任务调度类
 */
public class SyncTaskJob implements Job {
    
    private static final Logger logger = LoggerFactory.getLogger(SyncTaskJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // 获取任务ID
        String taskId = (String) context.getJobDetail().getJobDataMap().get("taskId");
        
        logger.info("定时同步任务开始执行，任务ID：{}", taskId);
        
        try {
            // 获取Spring上下文和服务
            SyncTaskService syncTaskService = ApplicationContextHolder.getBean(SyncTaskService.class);
            IncrementalSyncService incrementalSyncService = ApplicationContextHolder.getBean(IncrementalSyncService.class);
            
            // 获取任务信息
            SyncTask task = syncTaskService.getById(taskId);
            
            if (task == null) {
                logger.error("定时同步任务不存在，任务ID：{}", taskId);
                return;
            }
            
            if (task.getStatus() != SyncStatusEnum.ENABLED) {
                logger.info("定时同步任务已禁用，任务ID：{}", taskId);
                return;
            }
            
            // 获取上次同步时间
            Date lastSyncTime = syncTaskService.getLastSyncTime(taskId);
            
            // 执行增量同步
            incrementalSyncService.executeIncrementalSync(task, lastSyncTime);
            
            logger.info("定时同步任务执行完成，任务ID：{}", taskId);
            
        } catch (Exception e) {
            logger.error("定时同步任务执行失败，任务ID：{}", taskId, e);
            throw new JobExecutionException("定时同步任务执行失败", e, false);
        }
    }
}
```

## 五、前端Vue组件示例

```vue
<template>
  <div class="sync-task-form">
    <el-form ref="formRef" :model="form" :rules="rules" label-width="120px">
      <!-- 基本配置 -->
      <el-card shadow="never" class="form-card">
        <template #header>
          <div class="card-header">
            <span>基本配置</span>
          </div>
        </template>
        
        <el-form-item label="任务名称" prop="taskName">
          <el-input v-model="form.taskName" placeholder="请输入任务名称" />
        </el-form-item>
        
        <el-form-item label="同步类型" prop="syncType">
          <el-radio-group v-model="form.syncType">
            <el-radio label="FULL">全量同步</el-radio>
            <el-radio label="INCREMENTAL">增量同步</el-radio>
          </el-radio-group>
        </el-form-item>
        
        <el-form-item label="定时表达式" prop="cronExpression" v-if="form.syncType === 'INCREMENTAL'">
          <el-input v-model="form.cronExpression" placeholder="请输入Cron表达式" />
        </el-form-item>
      </el-card>
      
      <!-- 源库配置 -->
      <el-card shadow="never" class="form-card">
        <template #header>
          <div class="card-header">
            <span>源库配置</span>
          </div>
        </template>
        
        <el-form-item label="数据源" prop="srcDb">
          <el-select v-model="form.srcDb" placeholder="请选择数据源">
            <el-option v-for="item in dataSourceList" :key="item.id" :label="item.name" :value="item.id" />
          </el-select>
        </el-form-item>
        
        <el-form-item label="SCHEMA" prop="srcSchema">
          <el-input v-model="form.srcSchema" placeholder="请输入SCHEMA名称" />
        </el-form-item>
        
        <el-form-item label="表名" prop="srcTable">
          <el-input v-model="form.srcTable" placeholder="请输入表名" @blur="handleTableBlur('src')" />
        </el-form-item>
        
        <el-form-item label="唯一键" prop="uniqueKey">
          <el-select v-model="form.uniqueKey" multiple placeholder="请选择唯一键">
            <el-option v-for="field in srcFieldList" :key="field" :label="field" :value="field" />
          </el-select>
        </el-form-item>
        
        <el-form-item label="时间戳字段" prop="timestampField" v-if="form.syncType === 'INCREMENTAL'">
          <el-select v-model="form.timestampField" placeholder="请选择时间戳字段">
            <el-option v-for="field in srcFieldList" :key="field" :label="field" :value="field" />
          </el-select>
        </el-form-item>
      </el-card>
      
      <!-- 目标库配置 -->
      <el-card shadow="never" class="form-card">
        <template #header>
          <div class="card-header">
            <span>目标库配置</span>
          </div>
        </template>
        
        <el-form-item label="数据源" prop="dstDb">
          <el-select v-model="form.dstDb" placeholder="请选择数据源">
            <el-option v-for="item in dataSourceList" :key="item.id" :label="item.name" :value="item.id" />
          </el-select>
        </el-form-item>
        
        <el-form-item label="SCHEMA" prop="dstSchema">
          <el-input v-model="form.dstSchema" placeholder="请输入SCHEMA名称" />
        </el-form-item>
        
        <el-form-item label="表名" prop="dstTable">
          <el-input v-model="form.dstTable" placeholder="请输入表名" />
        </el-form-item>
      </el-card>
      
      <!-- 操作按钮 -->
      <div class="form-actions">
        <el-button type="primary" @click="handleSubmit" :loading="loading">保存</el-button>
        <el-button @click="handleCancel">取消</el-button>
      </div>
    </el-form>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, watch } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { useRouter } from 'vue-router'
import { getDataSourceList, getTableFields, createTask, updateTask } from '@/api/task'

const router = useRouter()

// 表单引用
const formRef = ref()

// 加载状态
const loading = ref(false)

// 数据源列表
const dataSourceList = ref([])

// 源表字段列表
const srcFieldList = ref([])

// 表单数据
const form = reactive({
  taskId: '',
  taskName: '',
  syncType: 'INCREMENTAL',
  cronExpression: '',
  srcDb: '',
  srcSchema: '',
  srcTable: '',
  uniqueKey: [],
  timestampField: '',
  dstDb: '',
  dstSchema: '',
  dstTable: ''
})

// 表单验证规则
const rules = {
  taskName: [
    { required: true, message: '请输入任务名称', trigger: 'blur' },
    { min: 2, max: 50, message: '任务名称长度在 2 到 50 个字符', trigger: 'blur' }
  ],
  syncType: [
    { required: true, message: '请选择同步类型', trigger: 'change' }
  ],
  srcDb: [
    { required: true, message: '请选择源数据源', trigger: 'change' }
  ],
  srcSchema: [
    { required: true, message: '请输入源SCHEMA', trigger: 'blur' }
  ],
  srcTable: [
    { required: true, message: '请输入源表名', trigger: 'blur' }
  ],
  uniqueKey: [
    { required: true, message: '请选择唯一键', trigger: 'change' }
  ],
  timestampField: [
    { required: true, message: '请选择时间戳字段', trigger: 'change' }
  ],
  dstDb: [
    { required: true, message: '请选择目标数据源', trigger: 'change' }
  ],
  dstSchema: [
    { required: true, message: '请输入目标SCHEMA', trigger: 'blur' }
  ],
  dstTable: [
    { required: true, message: '请输入目标表名', trigger: 'blur' }
  ]
}

// 初始化数据源列表
const initDataSourceList = async () => {
  try {
    const response = await getDataSourceList()
    dataSourceList.value = response.data
  } catch (error) {
    ElMessage.error('获取数据源列表失败')
  }
}

// 处理表名输入失焦
const handleTableBlur = async (type) => {
  if (type === 'src' && form.srcDb && form.srcSchema && form.srcTable) {
    try {
      const response = await getTableFields(form.srcDb, form.srcSchema, form.srcTable)
      srcFieldList.value = response.data
    } catch (error) {
      ElMessage.error('获取表字段失败')
    }
  }
}

// 提交表单
const handleSubmit = async () => {
  if (!formRef.value) return
  
  try {
    await formRef.value.validate()
    loading.value = true
    
    // 将唯一键数组转换为字符串
    const submitData = { ...form }
    submitData.uniqueKey = form.uniqueKey.join(',')
    
    let response
    if (form.taskId) {
      // 更新任务
      response = await updateTask(submitData)
    } else {
      // 创建任务
      response = await createTask(submitData)
    }
    
    ElMessage.success(form.taskId ? '任务更新成功' : '任务创建成功')
    router.push('/task/list')
    
  } catch (error) {
    ElMessage.error(error.message || '提交失败')
  } finally {
    loading.value = false
  }
}

// 取消操作
const handleCancel = () => {
  router.push('/task/list')
}

// 监听同步类型变化
watch(() => form.syncType, (newVal) => {
  if (newVal === 'FULL') {
    form.timestampField = ''
    form.cronExpression = ''
  }
})

// 组件挂载
onMounted(() => {
  initDataSourceList()
})
</script>

<style scoped>
.sync-task-form {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.form-card {
  margin-bottom: 20px;
}

.card-header {
  font-size: 16px;
  font-weight: bold;
}

.form-actions {
  margin-top: 30px;
  text-align: center;
}

.form-actions .el-button {
  margin: 0 10px;
}
</style>
```

## 六、API接口实现

```java
package com.dmsync.controller;

import com.dmsync.dto.SyncTriggerDTO;
import com.dmsync.dto.SyncResultDTO;
import com.dmsync.service.syncTask.SyncTaskService;
import com.dmsync.service.sync.IncrementalSyncService;
import com.dmsync.service.sync.FullSyncService;
import com.dmsync.service.syncLog.SyncLogService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Date;

/**
 * 同步操作控制器
 */
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
    
    /**
     * 触发同步
     */
    @PostMapping("/trigger")
    @ApiOperation("触发同步")
    public SyncResultDTO triggerSync(@RequestBody SyncTriggerDTO triggerDTO) {
        // 获取任务信息
        SyncTask task = syncTaskService.getById(triggerDTO.getTaskId());
        
        if (task == null) {
            throw new BusinessException("任务不存在");
        }
        
        SyncResultDTO result;
        
        if (triggerDTO.getSyncType() == SyncTypeEnum.FULL) {
            // 执行全量同步
            result = fullSyncService.executeFullSync(task);
        } else {
            // 执行增量同步
            Date lastSyncTime = syncLogService.getLastSyncTime(task.getTaskId());
            result = incrementalSyncService.executeIncrementalSync(task, lastSyncTime);
        }
        
        return result;
    }
    
    /**
     * 获取运行中的任务
     */
    @GetMapping("/running")
    @ApiOperation("获取运行中的任务")
    public List<SyncLogVO> getRunningTasks() {
        return syncLogService.getRunningTasks();
    }
    
    /**
     * 重试同步
     */
    @PostMapping("/retry/{logId}")
    @ApiOperation("重试同步")
    public void retrySync(@PathVariable String logId) {
        syncLogService.retrySync(logId);
    }
}
```

## 七、数据库表结构

```sql
-- 同步任务表
CREATE TABLE sync_task (
    task_id VARCHAR(32) PRIMARY KEY,
    task_name VARCHAR(100) NOT NULL,
    src_db VARCHAR(50) NOT NULL,
    src_schema VARCHAR(50) NOT NULL,
    src_table VARCHAR(100) NOT NULL,
    dst_db VARCHAR(50) NOT NULL,
    dst_schema VARCHAR(50) NOT NULL,
    dst_table VARCHAR(100) NOT NULL,
    unique_key VARCHAR(200) NOT NULL,
    timestamp_field VARCHAR(100),
    sync_type VARCHAR(20) NOT NULL,
    cron_expression VARCHAR(50),
    status VARCHAR(20) NOT NULL,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 同步日志表
CREATE TABLE sync_log (
    log_id VARCHAR(32) PRIMARY KEY,
    task_id VARCHAR(32) NOT NULL,
    task_name VARCHAR(100) NOT NULL,
    sync_type VARCHAR(20) NOT NULL,
    sync_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    status VARCHAR(20) NOT NULL,
    total_count INT,
    success_count INT,
    failed_count INT,
    error_message TEXT,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 数据变更日志表
CREATE TABLE sync_change_log (
    log_id VARCHAR(32) PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    schema_name VARCHAR(50) NOT NULL,
    operation_type VARCHAR(10) NOT NULL,
    unique_key_value VARCHAR(500) NOT NULL,
    change_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    sync_status VARCHAR(20) NOT NULL,
    sync_time TIMESTAMP,
    error_message TEXT,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 数据源配置表
CREATE TABLE data_source (
    id VARCHAR(32) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL,
    driver_class_name VARCHAR(100) NOT NULL,
    url VARCHAR(200) NOT NULL,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 创建索引
CREATE INDEX idx_sync_task_status ON sync_task(status);
CREATE INDEX idx_sync_log_task_id ON sync_log(task_id);
CREATE INDEX idx_sync_log_status ON sync_log(status);
CREATE INDEX idx_change_log_table ON sync_change_log(table_name, schema_name);
CREATE INDEX idx_change_log_status ON sync_change_log(sync_status);
CREATE INDEX idx_change_log_time ON sync_change_log(change_time);
```

## 八、配置文件

```yaml
# 应用配置
spring:
  application:
    name: dm-sync
  
  # 数据源配置
  datasource:
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
  
  # Quartz配置
  quartz:
    job-store-type: jdbc
    initialize-schema: embedded
    properties:
      org:
        quartz:
          scheduler:
            instanceName: dmSyncScheduler
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            tablePrefix: QRTZ_
            isClustered: true
          threadPool:
            threadCount: 10

# 日志配置
logging:
  level:
    com.dmsync: info
  file:
    name: logs/dm-sync.log

# 服务器配置
server:
  port: 8080
  servlet:
    context-path: /

# JWT配置
jwt:
  secret: your-jwt-secret-key
  expiration: 3600000
```
