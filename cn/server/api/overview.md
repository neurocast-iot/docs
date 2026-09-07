# NeuroCast Server API 概览

> Base URL: `http://<host>:8189`

## 认证方式

| 认证方式 | 适用场景 | 传递方式 |
|---|---|---|
| Redis Token | 管理后台操作 | `Authorization: Bearer <accessToken>` |
| JWT | C 端会员 | `Authorization: Bearer <accessToken>` |
| X-API-KEY | 第三方服务/设备事件推送 | `X-API-KEY: <apiKey>` |

**路径规范**：
- `/api/admin/**` — 管理后台接口（需 Redis Token 认证）
- `/api/app/**` — C 端会员接口（需 JWT 认证）
- `/api/auth/**` — 管理后台认证接口（公开）
- `/api/device/event` — TB 回调（X-API-KEY）

---

## 统一响应格式

```json
{
  "code": 0,
  "message": null,
  "data": { ... }
}
```

- `code = 0`：成功
- `code != 0`：失败，`message` 包含错误描述

### 分页响应

```json
{
  "code": 0,
  "data": {
    "total": 100,
    "items": [ ... ]
  }
}
```

### 分页查询参数

| 参数 | 类型 | 说明 |
|---|---|---|
| pageNum | int | 页码，从 1 开始 |
| pageSize | int | 每页大小，默认 10，最大 100 |
| orderByName | String | 排序字段 |
| orderByDesc | Boolean | 是否倒序 |

---

## 错误码总表

所有接口错误统一返回 `{ "code": <错误码>, "data": null, "message": "<描述>" }` 格式。前端可根据 `code` 做精确分支处理。

### 通用码

| code | 常量名 | 说明 |
|---|---|---|
| 0 | `SUCCESS` | 成功 |
| 1 | `FAILED` | 通用失败（仅框架内部兜底，业务接口不会返回） |
| 1000 | `VALIDATE_FAILED` | 参数校验失败 |
| 1002 | `PARAMETER_FORMAT_ERROR` | 参数格式错误 |
| 2000 | `ERROR` | 系统异常 |
| 2001 | `TIMEOUT` | 请求超时 |
| 2003 | `HTTP_REQUEST_METHOD_NOT_SUPPORTED` | 请求方式不支持 |
| 2004 | `FORBIDDEN` | 无访问权限 |
| 2005 | `UNAUTHORIZED` | 未认证或认证已过期 |
| 2006 | `NO_PERMISSION_ACCESS` | 无权访问 |
| 2007 | `ACCESS_TOKEN_INVALID` | 访问令牌无效 |
| 2008 | `REFRESH_TOKEN_INVALID` | 刷新令牌失效 |
| 2009 | `REFRESH_TOKEN_INCORRECT` | 刷新令牌错误 |
| 2010 | `API_INTERFACE_LIMIT` | 接口限流 |
| 2011 | `DUPLICATE_KEY` | 数据已存在（数据库唯一约束冲突） |
| 2012 | `LOGIN_USERNAME_PASSWORD_ERROR` | 用户名或密码错误 |
| 2013 | `LOGIN_USER_DISABLED` | 用户已被禁用 |
| 2014 | `API_KEY_INVALID` | API Key 无效 |

### ThingsBoard 集成（3000-3999）

| code | 常量名 | 说明 |
|---|---|---|
| 3000 | `BUSINESS_THINGSBOARD_ERROR` | ThingsBoard 调用失败 |
| 3001 | `BUSINESS_THINGSBOARD_DEVICE_EXISTED` | ThingsBoard 设备已存在 |
| 3002 | `BUSINESS_THINGSBOARD_DEVICE_NOT_EXISTED` | ThingsBoard 设备不存在 |

### 设备与产品管理（4000-4999）

| code | 常量名 | 说明 |
|---|---|---|
| 4000 | `BUSINESS_DEVICE_NOT_EXISTED` | 设备不存在 |
| 4001 | `BUSINESS_DEVICE_EXISTED` | 设备已存在 |
| 4002 | `BUSINESS_DEVICE_NOT_ONLINE` | 设备不在线 |
| 4003 | `BUSINESS_DEVICE_NO_AVAILABLE_PORT` | 没有可用的端口 |
| 4004 | `BUSINESS_PRODUCT_NOT_EXISTED` | 产品不存在 |
| 4005 | `BUSINESS_PRODUCT_HAS_DEVICES` | 产品下还有设备，无法删除 |
| 4006 | `BUSINESS_PRODUCT_NO_TB_PROFILE` | 产品无 ThingsBoard Profile |
| 4008 | `BUSINESS_DEVICE_GLOBAL_CONFIG_NOT_EXISTED` | 设备全局配置项不存在 |
| 4009 | `BUSINESS_SYS_CONFIG_NOT_EXISTED` | 系统配置项不存在 |

### 媒体文件与流（5000-5999）

| code | 常量名 | 说明 |
|---|---|---|
| 5000 | `BUSINESS_FILE_NOT_EXISTED` | 文件不存在 |
| 5001 | `BUSINESS_FILE_UPLOAD_NOT_EMPTY` | 文件内容不能为空 |
| 5002 | `BUSINESS_FILE_UPLOAD_TOO_LARGE` | 文件大小超出限制 |
| 5003 | `BUSINESS_FILE_UPLOAD_TASK_EXPIRED` | 上传任务不存在或已过期 |
| 5004 | `BUSINESS_FILE_CHUNK_INCOMPLETE` | 分片未上传完整 |
| 5005 | `BUSINESS_FILE_HASH_MISMATCH` | 文件校验失败，MD5 不匹配 |
| 5006 | `BUSINESS_STREAM_NOT_STARTED` | 流未开启 |
| 5007 | `BUSINESS_SRS_ERROR` | SRS 流媒体服务调用失败 |

### 用户管理（7000-7999）

| code | 常量名 | 说明 |
|---|---|---|
| 7000 | `BUSINESS_USER_NOT_EXISTED` | 用户不存在 |
| 7001 | `BUSINESS_USER_EXISTED` | 用户名已存在 |
| 7002 | `BUSINESS_ROLE_NOT_EXISTED` | 角色不存在 |
| 7003 | `BUSINESS_ROLE_CODE_EXISTED` | 角色编码已存在 |
| 7004 | `BUSINESS_ROLE_IN_USE` | 角色正在被用户使用，无法删除 |

### API 客户端管理（8000-8999）

| code | 常量名 | 说明 |
|---|---|---|
| 8000 | `BUSINESS_API_CLIENT_NOT_EXISTED` | API 客户端不存在 |

### 权限管理（9000-9999）

| code | 常量名 | 说明 |
|---|---|---|
| 9000 | `BUSINESS_PERMISSION_NOT_EXISTED` | 权限不存在 |
| 9001 | `BUSINESS_PERMISSION_CODE_EXISTED` | 权限标识已存在 |
| 9002 | `BUSINESS_PERMISSION_IN_USE` | 权限正在被角色使用，无法删除 |
