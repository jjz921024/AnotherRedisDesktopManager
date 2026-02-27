# WERedis Connection 定制开发说明

**定制日期**: 2026-02-27
**定制版本**: v1.0

---

## 一、定制概述

本次定制对 Redis 桌面管理工具的连接功能进行了改造，实现了基于 HTTP API 动态获取 Redis 地址的连接方式。

### 核心变更

1. **连接方式固定**：仅支持 Redis 单节点连接，隐藏 Cluster 和 Sentinel 选项
2. **动态地址获取**：Redis 连接地址通过 HTTP API 动态获取，而非用户手动输入
3. **新增认证字段**：ClusterName（集群名称）、UM Account、UM Password
4. **密码动态拼接**：连接密码由 UM 凭证按规则拼接生成

---

## 二、HTTP API 接口规范

### 2.1 获取集群名称列表

**接口地址**: `GET http://127.0.0.1:8080/api/weredis/getAllClusterNames`

**调用时机**: 打开新建连接对话框时自动调用

**响应报文**:
```json
{
  "code": "0",
  "msg": "请求处理成功",
  "resultData": [
    "GNS_GENERAL_PRESSURE_REDIS_CLUSTER_DATASTORE",
    "RPD_GENERAL_REDIS_CLUSTER_CACHE",
    "test-cluster"
  ],
  "page": null,
  "others": {}
}
```

**字段说明**:
- `code`: 状态码，"0" 表示成功
- `msg`: 响应消息
- `resultData`: 集群名称字符串数组

### 2.2 获取 Redis 代理地址

**接口地址**: `GET http://127.0.0.1:19091/redis_observer/proxy_online_list?clusterName={clusterName}`

**调用时机**: 用户点击打开连接时调用

**响应报文**:
```json
{
  "code": "0",
  "msg": "请求处理成功",
  "result": [
    {"host": "xx", "port": 6379}
  ]
}
```

**字段说明**:
- `code`: 状态码，"0" 表示成功
- `msg`: 响应消息
- `result`: 代理地址对象数组
  - `host`: Redis 代理主机地址
  - `port`: Redis 代理端口

**使用规则**: 使用 `result[0]` 的第一个代理地址

---

## 三、密码生成规则

**格式**: `{UM_ACCOUNT}:GUI|||{UM_PASSWORD}`

**示例**:
- UM Account: `user123`
- UM Password: `pass456`
- 生成的 Redis 密码: `user123:GUI|||pass456`

---

## 四、界面修改说明

### 4.1 新建连接对话框

**新增字段**:

| 字段名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Cluster Name | 下拉选择 | 是 | 从 HTTP API 获取的集群名称列表，支持搜索过滤 |
| UM Account | 文本输入 | 是 | UM 账号 |
| UM Password | 密码输入 | 是 | UM 密码 |

**隐藏字段**:
- Host（主机地址）
- Port（端口）
- Auth（密码）
- Username（用户名）
- Cluster（集群模式复选框）
- Sentinel（哨兵模式复选框）
- Sentinel 配置区域

**保留字段**:
- Connection Name（连接名称，可选）
- SSH、SSL、Readonly 等高级选项

### 4.2 连接存储格式

**新增字段**:
- `weredis`: 布尔值，标记为 WERedis 连接
- `clusterName`: 字符串，选中的集群名称
- `umAccount`: 字符串，UM 账号
- `umPassword`: 字符串，UM 密码

**不存储字段**:
- `host`, `port`: 运行时从 HTTP API 动态获取
- `auth`: 运行时根据 UM 凭证动态拼接

---

## 五、连接建立流程

### 5.1 创建连接流程

```
用户点击"新建连接"
    ↓
调用 HTTP API 获取集群名称列表
    ↓
显示 Cluster Name 下拉选项（带加载状态）
    ↓
用户选择集群，填写 UM Account 和 UM Password
    ↓
保存连接配置（标记 weredis: true）
```

### 5.2 打开连接流程

```
用户点击已保存的连接
    ↓
检测到 weredis: true 标识
    ↓
调用 HTTP API 获取代理地址（传入 clusterName）
    ↓
提取 host 和 port
    ↓
生成密码：{UM_ACCOUNT}:GUI|||{UM_PASSWORD}
    ↓
使用动态地址和拼接密码建立 Redis 连接
```

---

## 六、错误处理

| 场景 | 处理方式 |
|------|---------|
| 集群名称 API 失败 | 显示错误信息，禁用保存按钮 |
| 集群名称 API 超时（5秒） | 显示 "Connection timeout" |
| 代理地址 API 失败 | 显示错误提示，阻止连接 |
| 代理地址 API 超时（5秒） | 显示 "Connection timeout when fetching proxy address" |
| 代理地址返回空数组 | 显示 "No available proxies" |
