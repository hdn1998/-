# 达梦数据库增量同步系统Web界面设计

## 一、前端架构概述

### 1. 技术栈

| 技术 | 版本 | 用途 | 选型理由 |
|------|------|------|----------|
| Vue.js | 3.x | 前端框架 | 响应式设计、组件化开发、性能优异 |
| Element Plus | 2.x | UI组件库 | 与Vue 3兼容、组件丰富、文档完善 |
| Vite | 4.x | 构建工具 | 快速冷启动、按需编译、热模块替换 |
| Pinia | 2.x | 状态管理 | Vue官方推荐、类型安全、轻量级 |
| Vue Router | 4.x | 路由管理 | 与Vue 3兼容、支持组合式API |
| Axios | 1.x | HTTP客户端 | 功能强大、支持拦截器、自动转换JSON |
| Day.js | 1.x | 日期处理 | 轻量级、API友好、国际化支持 |
| ECharts | 5.x | 图表库 | 丰富的图表类型、高性能、可定制性强 |

### 2. 项目结构

```
src/
├── assets/             # 静态资源
│   ├── css/            # 全局样式
│   └── images/         # 图片资源
├── components/         # 通用组件
│   ├── SyncTaskForm.vue    # 同步任务表单组件
│   ├── SyncStatus.vue      # 同步状态组件
│   ├── LogTable.vue        # 日志表格组件
│   └── DataSourceSelect.vue # 数据源选择组件
├── layouts/            # 页面布局
│   └── MainLayout.vue  # 主布局
├── pages/              # 页面组件
│   ├── task/           # 任务管理模块
│   │   ├── TaskList.vue    # 任务列表
│   │   ├── TaskDetail.vue  # 任务详情
│   │   └── TaskConfig.vue  # 任务配置
│   ├── sync/           # 同步操作模块
│   │   ├── SyncTrigger.vue # 同步触发
│   │   └── SyncMonitor.vue # 同步监控
│   ├── log/            # 日志管理模块
│   │   ├── TaskLog.vue     # 任务执行日志
│   │   └── ChangeLog.vue   # 数据变更日志
│   └── system/         # 系统管理模块
│       ├── DataSourceConfig.vue # 数据源配置
│       └── UserManage.vue       # 用户管理
├── router/             # 路由配置
│   └── index.js        # 路由定义
├── stores/             # 状态管理
│   ├── taskStore.js    # 任务管理状态
│   ├── syncStore.js    # 同步操作状态
│   ├── logStore.js     # 日志管理状态
│   └── systemStore.js  # 系统管理状态
├── services/           # API服务
│   ├── taskService.js  # 任务管理API
│   ├── syncService.js  # 同步操作API
│   ├── logService.js   # 日志管理API
│   └── systemService.js # 系统管理API
├── utils/              # 工具函数
│   ├── request.js      # Axios封装
│   ├── uniqueKey.js    # 唯一键处理
│   └── format.js       # 格式化工具
├── App.vue             # 根组件
└── main.js             # 入口文件
```

## 二、核心功能模块设计

### 1. 任务管理模块

#### 1.1 任务列表页面 (TaskList.vue)

**页面结构**：
- 页面标题："同步任务列表"
- 操作栏：新增任务按钮、刷新按钮、批量删除按钮
- 筛选区：任务名称、状态、同步类型筛选
- 任务表格：展示所有同步任务的基本信息
- 分页组件：分页显示任务列表

**主要组件**：
- `el-button`：操作按钮
- `el-input`：任务名称筛选
- `el-select`：状态和同步类型筛选
- `el-table`：任务表格
- `el-pagination`：分页组件

**交互逻辑**：
```javascript
// 任务列表数据获取
const loadTaskList = async () => {
  loading.value = true;
  try {
    const params = {
      pageNum: currentPage.value,
      pageSize: pageSize.value,
      taskName: searchForm.value.taskName,
      status: searchForm.value.status,
      syncType: searchForm.value.syncType
    };
    
    const response = await taskService.getTaskList(params);
    taskList.value = response.data.records;
    total.value = response.data.total;
  } catch (error) {
    ElMessage.error('获取任务列表失败：' + error.message);
  } finally {
    loading.value = false;
  }
};

// 新增任务
const handleAddTask = () => {
  // 打开任务配置对话框
  dialogVisible.value = true;
  isEdit.value = false;
  resetForm();
};

// 编辑任务
const handleEditTask = (row) => {
  dialogVisible.value = true;
  isEdit.value = true;
  form.value = JSON.parse(JSON.stringify(row));
};

// 删除任务
const handleDeleteTask = async (taskId) => {
  try {
    await ElMessageBox.confirm('确定要删除该任务吗？', '提示', {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    });
    
    await taskService.deleteTask(taskId);
    ElMessage.success('任务删除成功');
    loadTaskList();
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error('任务删除失败：' + error.message);
    }
  }
};
```

**表格列定义**：

| 列名 | 数据字段 | 宽度 | 类型 | 操作 |
|------|----------|------|------|------|
| 任务名称 | taskName | 200 | 文本 | - |
| 源库/SCHEMA | srcDb + "/" + srcSchema | 150 | 文本 | - |
| 源表 | srcTable | 120 | 文本 | - |
| 目标库/SCHEMA | dstDb + "/" + dstSchema | 150 | 文本 | - |
| 目标表 | dstTable | 120 | 文本 | - |
| 同步类型 | syncType | 100 | 标签 | FULL/INCREMENTAL |
| 状态 | status | 100 | 开关 | ENABLED/DISABLED |
| 创建时间 | createTime | 180 | 日期 | 格式化显示 |
| 操作 | - | 200 | 按钮组 | 编辑/删除/同步 |

#### 1.2 任务配置页面 (TaskConfig.vue)

**页面结构**：
- 页面标题："同步任务配置"
- 表单区域：分为基本配置、源库配置、目标库配置三个标签页
- 操作按钮：保存按钮、取消按钮

**主要组件**：
- `el-tabs`：标签页切换
- `el-form`：表单容器
- `el-form-item`：表单项
- `el-input`：文本输入
- `el-select`：下拉选择
- `el-button`：操作按钮

**表单验证规则**：
```javascript
const rules = {
  taskName: [
    { required: true, message: '请输入任务名称', trigger: 'blur' },
    { min: 2, max: 50, message: '任务名称长度在 2 到 50 个字符', trigger: 'blur' }
  ],
  srcDb: [
    { required: true, message: '请选择源数据库', trigger: 'change' }
  ],
  srcSchema: [
    { required: true, message: '请输入源库SCHEMA', trigger: 'blur' }
  ],
  srcTable: [
    { required: true, message: '请输入源表名', trigger: 'blur' }
  ],
  dstDb: [
    { required: true, message: '请选择目标数据库', trigger: 'change' }
  ],
  dstSchema: [
    { required: true, message: '请输入目标库SCHEMA', trigger: 'blur' }
  ],
  dstTable: [
    { required: true, message: '请输入目标表名', trigger: 'blur' }
  ],
  uniqueKey: [
    { required: true, message: '请输入唯一键字段', trigger: 'blur' }
  ],
  timestampField: [
    { required: true, message: '请输入时间戳字段', trigger: 'blur' }
  ]
};
```

**交互逻辑**：
```javascript
// 保存任务配置
const saveTask = async () => {
  await formRef.value.validate(async (valid) => {
    if (valid) {
      try {
        loading.value = true;
        
        if (isEdit.value) {
          await taskService.updateTask(form.value);
          ElMessage.success('任务更新成功');
        } else {
          await taskService.createTask(form.value);
          ElMessage.success('任务创建成功');
        }
        
        dialogVisible.value = false;
        loadTaskList();
      } catch (error) {
        ElMessage.error('保存失败：' + error.message);
      } finally {
        loading.value = false;
      }
    }
  });
};

// 加载数据库表字段
const loadTableFields = async (database, schema, table) => {
  try {
    const response = await systemService.getTableFields(database, schema, table);
    fieldList.value = response.data;
  } catch (error) {
    ElMessage.error('获取表字段失败：' + error.message);
  }
};
```

### 2. 同步操作模块

#### 2.1 同步触发页面 (SyncTrigger.vue)

**页面结构**：
- 页面标题："同步触发"
- 任务选择区：下拉选择要同步的任务
- 同步参数区：选择同步类型（全量/增量）、时间范围
- 操作区：立即同步按钮、取消按钮
- 同步结果区：显示同步进度和结果

**主要组件**：
- `el-select`：任务选择
- `el-radio-group`：同步类型选择
- `el-date-picker`：时间范围选择
- `el-button`：操作按钮
- `el-progress`：同步进度条

**交互逻辑**：
```javascript
// 立即同步
const handleSync = async () => {
  if (!form.value.taskId) {
    ElMessage.warning('请选择要同步的任务');
    return;
  }
  
  try {
    loading.value = true;
    syncResult.value = null;
    
    const response = await syncService.triggerSync(form.value);
    syncResult.value = response.data;
    
    if (syncResult.value.status === 'SUCCESS') {
      ElMessage.success('同步成功');
    } else {
      ElMessage.error('同步失败：' + syncResult.value.errorMessage);
    }
  } catch (error) {
    ElMessage.error('同步失败：' + error.message);
  } finally {
    loading.value = false;
  }
};
```

#### 2.2 同步监控页面 (SyncMonitor.vue)

**页面结构**：
- 页面标题："同步监控"
- 实时监控区：显示当前运行的同步任务
- 任务状态卡片：显示任务名称、状态、进度、成功/失败计数
- 图表区：显示同步性能指标图表（如同步延迟、吞吐量）

**主要组件**：
- `el-card`：任务状态卡片
- `el-progress`：同步进度条
- `el-timeline`：同步时间线
- `echarts`：性能指标图表

**交互逻辑**：
```javascript
// 实时更新任务状态
const updateTaskStatus = async () => {
  try {
    const response = await syncService.getRunningTasks();
    runningTasks.value = response.data;
  } catch (error) {
    console.error('获取运行任务失败：', error);
  }
};

// 定时更新任务状态
const startTimer = () => {
  timer.value = setInterval(updateTaskStatus, 3000); // 每3秒更新一次
};

// 停止定时更新
const stopTimer = () => {
  if (timer.value) {
    clearInterval(timer.value);
    timer.value = null;
  }
};

// 组件挂载时启动定时更新
onMounted(() => {
  updateTaskStatus();
  startTimer();
});

// 组件卸载时停止定时更新
onUnmounted(() => {
  stopTimer();
});
```

### 3. 日志管理模块

#### 3.1 任务执行日志页面 (TaskLog.vue)

**页面结构**：
- 页面标题："任务执行日志"
- 筛选区：任务名称、时间范围、执行状态筛选
- 日志表格：展示任务执行日志
- 日志详情：点击日志行显示详细信息

**主要组件**：
- `el-input`：任务名称筛选
- `el-date-picker`：时间范围筛选
- `el-select`：执行状态筛选
- `el-table`：日志表格
- `el-dialog`：日志详情对话框

**表格列定义**：

| 列名 | 数据字段 | 宽度 | 类型 | 操作 |
|------|----------|------|------|------|
| 任务名称 | taskName | 200 | 文本 | - |
| 同步时间 | syncTime | 180 | 日期 | - |
| 开始时间 | startTime | 180 | 日期 | - |
| 结束时间 | endTime | 180 | 日期 | - |
| 状态 | status | 100 | 标签 | SUCCESS/FAILED/RUNNING |
| 总记录数 | totalCount | 100 | 数字 | - |
| 成功记录数 | successCount | 120 | 数字 | - |
| 失败记录数 | failedCount | 120 | 数字 | - |
| 操作 | - | 100 | 按钮 | 查看详情 |

**交互逻辑**：
```javascript
// 查看日志详情
const viewLogDetail = (row) => {
  logDetail.value = row;
  dialogVisible.value = true;
};

// 重试失败的同步
const handleRetry = async (logId) => {
  try {
    await syncService.retrySync(logId);
    ElMessage.success('重试成功');
    loadLogList();
  } catch (error) {
    ElMessage.error('重试失败：' + error.message);
  }
};
```

#### 3.2 数据变更日志页面 (ChangeLog.vue)

**页面结构**：
- 页面标题："数据变更日志"
- 筛选区：表名、时间范围、操作类型、同步状态筛选
- 日志表格：展示数据变更日志
- 数据详情：点击日志行显示变更前后的数据

**主要组件**：
- `el-input`：表名筛选
- `el-date-picker`：时间范围筛选
- `el-select`：操作类型和同步状态筛选
- `el-table`：日志表格
- `el-dialog`：数据详情对话框
- `el-descriptions`：数据详情描述

**表格列定义**：

| 列名 | 数据字段 | 宽度 | 类型 | 操作 |
|------|----------|------|------|------|
| 表名 | tableName | 150 | 文本 | - |
| 操作类型 | operationType | 100 | 标签 | INSERT/UPDATE/DELETE |
| 唯一键值 | uniqueKeyValue | 200 | 文本 | 省略显示 |
| 变更时间 | changeTime | 180 | 日期 | - |
| 同步状态 | syncStatus | 100 | 标签 | PENDING/SYNCED/FAILED |
| 同步时间 | syncTime | 180 | 日期 | - |
| 操作 | - | 100 | 按钮 | 查看详情 |

### 4. 系统管理模块

#### 4.1 数据源配置页面 (DataSourceConfig.vue)

**页面结构**：
- 页面标题："数据源配置"
- 操作栏：新增数据源按钮、刷新按钮
- 数据源表格：展示所有数据源配置
- 数据源配置表单：新增或编辑数据源

**主要组件**：
- `el-button`：操作按钮
- `el-table`：数据源表格
- `el-dialog`：数据源配置对话框
- `el-form`：数据源配置表单

**表格列定义**：

| 列名 | 数据字段 | 宽度 | 类型 | 操作 |
|------|----------|------|------|------|
| 数据源名称 | name | 150 | 文本 | - |
| 数据库类型 | type | 100 | 文本 | - |
| 主机地址 | host | 150 | 文本 | - |
| 端口 | port | 80 | 数字 | - |
| 数据库名 | database | 150 | 文本 | - |
| 用户名 | username | 150 | 文本 | - |
| 状态 | status | 100 | 开关 | ENABLED/DISABLED |
| 操作 | - | 150 | 按钮组 | 编辑/删除/测试连接 |

**交互逻辑**：
```javascript
// 测试数据库连接
const testConnection = async () => {
  try {
    await systemService.testConnection(form.value);
    ElMessage.success('连接成功');
  } catch (error) {
    ElMessage.error('连接失败：' + error.message);
  }
};
```

## 三、状态管理设计

### 1. 任务管理状态 (taskStore.js)

```javascript
import { defineStore } from 'pinia';
import taskService from '@/services/taskService';

export const useTaskStore = defineStore('task', {
  state: () => ({
    taskList: [],
    currentTask: null,
    loading: false,
    total: 0
  }),
  
  getters: {
    getTaskById: (state) => (id) => {
      return state.taskList.find(task => task.taskId === id);
    }
  },
  
  actions: {
    async fetchTaskList(params) {
      this.loading = true;
      try {
        const response = await taskService.getTaskList(params);
        this.taskList = response.data.records;
        this.total = response.data.total;
      } catch (error) {
        console.error('获取任务列表失败：', error);
        throw error;
      } finally {
        this.loading = false;
      }
    },
    
    async fetchTaskDetail(taskId) {
      try {
        const response = await taskService.getTaskDetail(taskId);
        this.currentTask = response.data;
        return response.data;
      } catch (error) {
        console.error('获取任务详情失败：', error);
        throw error;
      }
    },
    
    async createTask(task) {
      try {
        const response = await taskService.createTask(task);
        return response.data;
      } catch (error) {
        console.error('创建任务失败：', error);
        throw error;
      }
    },
    
    async updateTask(task) {
      try {
        const response = await taskService.updateTask(task);
        // 更新本地任务列表
        const index = this.taskList.findIndex(item => item.taskId === task.taskId);
        if (index !== -1) {
          this.taskList[index] = response.data;
        }
        return response.data;
      } catch (error) {
        console.error('更新任务失败：', error);
        throw error;
      }
    },
    
    async deleteTask(taskId) {
      try {
        await taskService.deleteTask(taskId);
        // 从本地任务列表中删除
        this.taskList = this.taskList.filter(item => item.taskId !== taskId);
        this.total--;
      } catch (error) {
        console.error('删除任务失败：', error);
        throw error;
      }
    }
  }
});
```

### 2. 同步操作状态 (syncStore.js)

```javascript
import { defineStore } from 'pinia';
import syncService from '@/services/syncService';

export const useSyncStore = defineStore('sync', {
  state: () => ({
    runningTasks: [],
    syncResult: null,
    loading: false
  }),
  
  actions: {
    async triggerSync(params) {
      this.loading = true;
      try {
        const response = await syncService.triggerSync(params);
        this.syncResult = response.data;
        return response.data;
      } catch (error) {
        console.error('触发同步失败：', error);
        throw error;
      } finally {
        this.loading = false;
      }
    },
    
    async getRunningTasks() {
      try {
        const response = await syncService.getRunningTasks();
        this.runningTasks = response.data;
        return response.data;
      } catch (error) {
        console.error('获取运行任务失败：', error);
        throw error;
      }
    },
    
    async retrySync(logId) {
      try {
        const response = await syncService.retrySync(logId);
        return response.data;
      } catch (error) {
        console.error('重试同步失败：', error);
        throw error;
      }
    },
    
    clearSyncResult() {
      this.syncResult = null;
    }
  }
});
```

## 四、API服务设计

### 1. Axios封装 (request.js)

```javascript
import axios from 'axios';
import { ElMessage, ElMessageBox } from 'element-plus';
import router from '@/router';

// 创建axios实例
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL, // 从环境变量获取API基础URL
  timeout: 60000 // 请求超时时间
});

// 请求拦截器
service.interceptors.request.use(
  config => {
    // 从localStorage获取token
    const token = localStorage.getItem('token');
    if (token) {
      config.headers['Authorization'] = 'Bearer ' + token;
    }
    return config;
  },
  error => {
    console.error('请求错误：', error);
    return Promise.reject(error);
  }
);

// 响应拦截器
service.interceptors.response.use(
  response => {
    const res = response.data;
    
    // 非200状态码视为错误
    if (res.code !== 200) {
      ElMessage.error(res.message || '请求失败');
      
      // 401未授权，跳转到登录页面
      if (res.code === 401) {
        ElMessageBox.confirm('登录已过期，请重新登录', '提示', {
          confirmButtonText: '重新登录',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          localStorage.removeItem('token');
          router.push('/login');
        });
      }
      
      return Promise.reject(new Error(res.message || '请求失败'));
    } else {
      return res;
    }
  },
  error => {
    console.error('响应错误：', error);
    ElMessage.error(error.message || '服务器错误');
    return Promise.reject(error);
  }
);

export default service;
```

### 2. 任务管理API (taskService.js)

```javascript
import request from '@/utils/request';

const taskService = {
  // 获取任务列表
  getTaskList(params) {
    return request({
      url: '/api/task/list',
      method: 'get',
      params
    });
  },
  
  // 获取任务详情
  getTaskDetail(taskId) {
    return request({
      url: `/api/task/${taskId}`,
      method: 'get'
    });
  },
  
  // 创建任务
  createTask(data) {
    return request({
      url: '/api/task',
      method: 'post',
      data
    });
  },
  
  // 更新任务
  updateTask(data) {
    return request({
      url: '/api/task',
      method: 'put',
      data
    });
  },
  
  // 删除任务
  deleteTask(taskId) {
    return request({
      url: `/api/task/${taskId}`,
      method: 'delete'
    });
  },
  
  // 启用/禁用任务
  updateTaskStatus(taskId, status) {
    return request({
      url: `/api/task/${taskId}/status`,
      method: 'put',
      params: { status }
    });
  }
};

export default taskService;
```

### 3. 同步操作API (syncService.js)

```javascript
import request from '@/utils/request';

const syncService = {
  // 触发同步
  triggerSync(data) {
    return request({
      url: '/api/sync/trigger',
      method: 'post',
      data
    });
  },
  
  // 获取运行中的任务
  getRunningTasks() {
    return request({
      url: '/api/sync/running',
      method: 'get'
    });
  },
  
  // 重试同步
  retrySync(logId) {
    return request({
      url: `/api/sync/retry/${logId}`,
      method: 'post'
    });
  },
  
  // 停止同步
  stopSync(taskId) {
    return request({
      url: `/api/sync/stop/${taskId}`,
      method: 'post'
    });
  }
};

export default syncService;
```

## 五、性能优化策略

### 1. 前端性能优化

| 优化项 | 实现方式 | 预期效果 |
|--------|----------|----------|
| 路由懒加载 | 使用`import()`动态导入路由组件 | 减少初始加载时间 |
| 组件按需加载 | 使用Element Plus的按需导入 | 减少打包体积 |
| 虚拟滚动 | 对大数据量表格使用虚拟滚动 | 提高表格渲染性能 |
| 防抖和节流 | 对频繁触发的事件使用防抖和节流 | 减少不必要的请求 |
| 缓存策略 | 对不经常变化的数据进行本地缓存 | 减少API请求次数 |
| 图片优化 | 压缩图片、使用适当的图片格式 | 减少资源加载时间 |
| 代码分割 | 使用Vite的代码分割功能 | 优化加载性能 |

### 2. 后端API优化

| 优化项 | 实现方式 | 预期效果 |
|--------|----------|----------|
| 分页查询 | 对列表数据使用分页查询 | 减少数据传输量 |
| 字段过滤 | 只返回前端需要的字段 | 减少数据传输量 |
| 缓存API响应 | 对频繁访问的数据进行缓存 | 提高API响应速度 |
| 批量处理 | 支持批量操作 | 减少请求次数 |
| 异步处理 | 对耗时操作使用异步处理 | 提高API响应速度 |

## 六、响应式设计

### 1. 布局适配

| 设备类型 | 屏幕宽度 | 布局方式 |
|----------|----------|----------|
| 桌面端 | >1200px | 完整布局（侧边栏+内容区） |
| 平板 | 768px-1200px | 紧凑布局（可折叠侧边栏） |
| 移动端 | <768px | 自适应布局（顶部导航栏） |

### 2. 组件适配

- 表格：在小屏幕上自动调整列宽，隐藏次要列
- 表单：在小屏幕上垂直排列表单字段
- 按钮组：在小屏幕上自动换行或折叠为下拉菜单
- 弹窗：在小屏幕上全屏显示

## 七、安全设计

### 1. 认证与授权

- 基于JWT的认证机制
- 角色基础的访问控制（RBAC）
- 路由守卫，限制未授权访问

### 2. 数据安全

- 敏感数据加密存储
- 接口请求参数验证
- 防止SQL注入和XSS攻击
- 使用HTTPS加密传输

### 3. 操作安全

- 重要操作二次确认
- 操作日志审计
- 防止CSRF攻击

## 八、总结

本Web界面设计基于Vue 3 + Element Plus技术栈，实现了达梦数据库增量同步系统的核心功能模块，包括任务管理、同步操作、日志管理和系统管理。界面设计遵循了现代化、易用性和响应式原则，同时考虑了性能优化和安全设计。

Web界面提供了直观的操作界面，用户可以方便地配置同步任务、手动触发同步、监控同步状态和查看同步日志，满足了用户通过Web实现增量同步的需求。