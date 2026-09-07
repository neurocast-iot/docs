# NeuroCast Server 架构文档

## 1. 系统概述

NeuroCast 是一个 IoT 设备管理平台，提供设备管理、实时直播、录像回放、媒体文件管理等能力。实时视频以 **WebRTC** 为核心，设备端通过 WebRTC P2P/SFU 推流，服务端通过 SRS 进行信令与转发，前端浏览器原生解码播放，实现全链路亚秒级低延迟。设备通过 MQTT 接入 ThingsBoard IoT 平台，由规则引擎转发事件至 NeuroCast 服务端。

**技术栈**：Spring Boot 3.5.9 / Java 21 / PostgreSQL / Redis / ThingsBoard / SRS（WebRTC SFU） / MyBatis-Plus

---

## 2. 功能模块总览

```
NeuroCast Server
├── 系统管理（system）
│   ├── 认证与令牌（登录/登出/刷新令牌/当前用户）
│   ├── 用户管理（CRUD/重置密码/角色关联）
│   ├── 角色与权限管理（RBAC / API 客户端 scope）
│   └── API 客户端管理（CRUD/密钥生成）
│
├── 设备管理（device）
│   ├── 产品管理（设备类型 CRUD，关联 ThingsBoard Device Profile）
│   ├── 设备 CRUD（创建/批量创建/查询/更新/删除，设备归属产品）
│   ├── 设备配置（业务配置/系统配置/OSD 水印配置/触发器配置）
│   ├── 设备指令（FRP 隧道/SSH 隧道/复位设备/重启/指令审计日志）
│   └── 设备事件（上下线状态同步）
│
├── 媒体服务（media）
│   ├── 相机控制（运行模式/参数配置）
│   ├── 直播与回放（实时直播/录像回放）
│   ├── 媒体库（抓拍图片列表/录像列表/请求设备上传）
│   └── 文件管理（分片上传/断点续传/秒传/批量上传/流式下载）
│
└── 定时任务（job）
    ├── HLS 分片缓存清理
    ├── 媒体文件定期清理
    └── 推流超时自动停止
```

---

## 3. 模块详细说明

### 3.1 系统管理

| 功能 | 说明 |
|---|---|
| 认证 | 用户名密码登录，返回 accessToken + refreshToken；accessToken 用于 API 鉴权（Bearer Token），refreshToken 用于无感续期 |
| 用户管理 | 管理员专属，支持创建/编辑/删除用户、重置密码、分页查询 |
| 角色与权限 | RBAC 模型，权限码人机共用（type 字段区分用户权限/API 客户端 scope） |
| API 客户端 | 管理员专属，为内部系统/设备签发 X-API-KEY 凭证，创建时返回完整 Key（仅此一次） |
| 系统配置 | 服务端 KV 配置（如 MQTT 地址等），仅服务端读取，不推送到设备 |

### 3.2 设备管理

| 功能 | 说明 |
|---|---|
| 产品管理 | 产品即"设备类型"，每个产品关联一个 ThingsBoard Device Profile。支持 CRUD、分页查询、型号唯一性校验。产品下有设备时禁止删除 |
| 设备 CRUD | 创建设备时必须指定产品，根据产品关联的 TB Profile 自动在 ThingsBoard 注册设备，返回设备接入 accessToken（仅创建时可见）；支持批量创建 |
| 设备配置 | 配置通过 ThingsBoard SHARED_SCOPE 属性下发到设备，设备自动接收变更。包含视频参数、抓拍参数、OSD 水印等 |
| 触发器配置 | 定时抓拍、录像、蓝牙标签等触发类配置通过独立的 triggers 接口下发 |
| 设备指令 | FRP 内网穿透（启动/停止）、SSH 隧道（启动/停止）、复位设备（清配置 + 自动重启）、远程重启、指令审计日志 |
| 设备事件 | 接收 ThingsBoard 规则引擎推送的设备上下线事件，更新本地状态与时间戳 |

**设备配置字段一览**：

| 分类 | 字段 | 类型 | 校验规则 | 说明 |
|---|---|---|---|---|
| 主通道 | mainCodec | String | `h264` \| `h265` | 编码格式 |
| | mainFrameRate | Integer | 5-30 | 帧率（fps） |
| | mainResolutionWidth/Height | Integer | 见分辨率表 | 分辨率 |
| | mainBitrate | Integer | — | 码率（Kbps） |
| | mainBrMode | String | `VBR` \| `CBR` | 码率模式 |
| 子通道 | subCodec | String | `h264` \| `h265` | 编码格式 |
| | subFrameRate | Integer | 5-30 | 帧率（fps） |
| | subResolutionWidth/Height | Integer | 见分辨率表 | 分辨率 |
| | subBitrate | Integer | — | 码率（Kbps） |
| | subBrMode | String | `VBR` \| `CBR` | 码率模式 |
| 拍照 | snapshotQuality | Integer | 20-100 | 图片质量 |
| 录像 | recordSegmentSec | Integer | 1-3600 | 录像分段时长（秒） |

**主通道支持分辨率**：320×176、640×360、768×432、1024×576、1280×720、1920×1080、2560×1440

**子通道支持分辨率**：320×176、640×360、768×432、1024×576、1280×720

> **OSD 水印配置**：通过独立的 OSD 配置接口管理，支持 time/label/rect/circle/ellipse/polygon/bitmap 七种元素类型，坐标使用千分比（0~1000）。每路画面最多同时显示 4 个有效元素。详见 [OSD 配置协议](protocol/osd_elements_config_api.md)

> **事件触发器配置**：通过统一的 triggers 配置接口下发，支持 timer/record/bluetooth 三种类型。详见 [触发器配置协议](protocol/triggers_config_api.md)

### 3.3 媒体服务

| 功能 | 说明 |
|---|---|
| 相机控制 | 设置运行模式、下发/查询相机基础参数 |
| 实时直播 | 开启设备实时推流，返回 SRS 播放地址（FLV/WebRTC），支持观众复用 |
| 录像回放 | 基于 HLS 协议的文件回放，支持时间范围 seek |
| 媒体库 | 按时间范围查询抓拍图片/录像列表，支持请求设备上传未上传的文件 |
| 文件上传 | 分片上传（支持断点续传/秒传）、批量小文件上传 |
| 文件下载 | 流式下载，支持 HTTP Range 断点续传 |

### 3.4 定时任务

| 任务 | 说明 |
|---|---|
| HLS 缓存清理 | 定期清理过期的 HLS 分片文件 |
| 媒体文件清理 | 定期清理过期的临时媒体文件 |
| 推流超时停止 | 检测超时的推流流并自动停止 |

---

## 4. 系统架构

```
┌─────────────┐     MQTT      ┌──────────────┐
│   设备端     │◄─────────────►│ ThingsBoard  │
│  (firmware)  │               │  (IoT 平台)   │
└──────┬───────┘               └──────┬───────┘
       │                              │
       │ RTMP/WebRTC                  │ 规则引擎
       ▼                              │ HTTP 回调
┌──────────────┐                      ▼
│     SRS      │              ┌───────────────┐
│  (流媒体)    │              │ NeuroCast     │
│              │◄────────────►│   Server      │
└──────┬───────┘   REST API   │  (Spring Boot)│
       │                      └───────┬───────┘
       │                              │
       │ HTTP-FLV/HLS/WebRTC         │ PostgreSQL
       ▼                              │ Redis
┌──────────────┐                      ▼
│   前端/客户端  │              ┌───────────────┐
│  (platform)  │              │  数据存储       │
└──────────────┘              └───────────────┘
```

---

## 5. 认证机制

| 认证方式 | 适用场景 | 传递方式 |
|---|---|---|
| JWT Bearer Token | 前端用户操作 | `Authorization: Bearer <accessToken>` |
| X-API-KEY | 内部系统/设备事件推送 | `X-API-KEY: <apiKey>` |

**令牌生命周期**：
- accessToken：短期有效（默认 2 小时），用于 API 鉴权
- refreshToken：长期有效（默认 7 天），用于无感续期
- 登出时两个令牌同时失效

---

## 6. 统一响应格式

```json
{
  "code": 0,
  "message": null,
  "data": { ... }
}
```

- `code = 0`：成功
- `code != 0`：失败，`message` 包含错误描述

**分页响应**：

```json
{
  "code": 0,
  "data": {
    "total": 100,
    "items": [ ... ]
  }
}
```

**分页查询参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| pageNum | int | 页码，从 1 开始 |
| pageSize | int | 每页大小，默认 10，最大 100 |
| orderByName | String | 排序字段 |
| orderByDesc | Boolean | 是否倒序 |
