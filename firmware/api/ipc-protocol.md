# IPC 协议文档

本文档描述 NeuroCast 各进程间的通信协议。

## 概述

NeuroCast 使用 ZeroMQ 作为进程间通信（IPC）的基础设施。系统中有两个主要进程：

| 进程 | 职责 | IPC 角色 |
|------|------|----------|
| `mediad` | 媒体服务（相机、录像、拍照、OSD、推流） | 命令响应方 + 事件发送方 |
| `iot_agent` | 云端代理（MQTT、配置同步、OTA、文件上传） | IPC 中心节点 |

## 通道架构

```
┌─────────────┐         DEALER/ROUTER         ┌─────────────┐
│   mediad    │ ◄─────────────────────────────►│  iot_agent  │
│             │   ipc:///tmp/iot_agent.ipc     │             │
│  (DEALER)   │                                │  (ROUTER)   │
└─────────────┘                                └─────────────┘
      │                                              │
      │ 事件广播                                      │ MQTT
      │ (PUB)                                        ▼
      ▼                                          ┌─────────┐
  mediactl                                       │  Cloud  │
  (SUB)                                          │  (TB)   │
                                                 └─────────┘
```

### 通道说明

| 通道 | 模式 | 端点 | 用途 |
|------|------|------|------|
| 命令通道 | DEALER/ROUTER | `ipc:///tmp/iot_agent.ipc` | iot_agent 向 mediad 发送命令 |
| 事件通道 | PUB/SUB | `ipc:///tmp/nc_mediad_evt.ipc` | mediad 广播媒体事件 |
| 调试通道 | REQ/REP | `ipc:///tmp/nc_mediad_cmd.ipc` | mediactl 直连 mediad |

## 消息格式

### 消息头（MessageHeader）

所有 IPC 消息共享统一的 64 字节消息头：

```c
struct MessageHeader {
    uint32_t domain;        // 消息域：0=INPROC, 1=IPC
    uint32_t type;          // 消息类型（见下方枚举）
    uint64_t timestamp;     // 创建时间戳（毫秒）
    uint32_t sequence;      // 序列号（单调递增）
    uint32_t payloadSize;   // 载荷大小（字节）
    char     from[32];      // 发送方名称（如 "mediad"）
    char     to[32];        // 接收方名称（空=发给中心节点）
};
```

### 请求信封（命令通道）

下行命令使用 JSON 信封：

```json
{
    "type": 12000,
    "payload": {
        "quality": 80
    }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | uint32 | 是 | 命令类型编号 |
| `payload` | object | 否 | 命令参数（部分命令无需此字段） |

### 应答信封

所有命令响应使用统一格式：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "path": "/mnt/emmc/media/photos/manual_20260907_120000.jpg"
    }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 0=成功，负数=失败 |
| `message` | string | 状态描述 |
| `data` | object | 可选，命令返回数据 |

### 错误码

| 错误码 | 名称 | 说明 |
|--------|------|------|
| 0 | Ok | 成功 |
| -1 | BadRequest | 参数不合法 |
| -2 | NotImplemented | 功能未实现 |
| -3 | Busy | 资源忙 |
| -4 | CameraNotReady | 相机未就绪 |
| -5 | Internal | 内部执行失败 |

## 消息类型枚举

### 系统控制（1000-2999）

| 编号 | 名称 | 方向 | 说明 |
|------|------|------|------|
| 1000 | HEARTBEAT | 双向 | 心跳消息 |
| 1001 | HEARTBEAT_ACK | 双向 | 心跳响应 |
| 2000 | SERVICE_STATUS | 双向 | 服务状态查询/上报 |
| 2001 | SERVICE_RESTART | iot_agent→mediad | 服务重启命令 |
| 2002 | SERVICE_STOP | iot_agent→mediad | 服务停止命令 |

### 配置同步（3000-3999, 12000-12099）

| 编号 | 名称 | 方向 | 载荷 |
|------|------|------|------|
| 3000 | CONFIG_UPDATE | iot_agent→mediad | 局部配置 JSON patch |
| 12000 | CONFIG_SYNC_STARTED | mediad→iot_agent | `{"serviceName":"mediad","configVersion":1}` |
| 12001 | CONFIG_SYNC_UPDATE | iot_agent→mediad | `{"version":2,"config":{...}}` |
| 12002 | CONFIG_SYNC_ACK | mediad→iot_agent | `{"version":2,"success":true}` |
| 12003 | CONFIG_SYNC_REQUEST | mediad→iot_agent | `{"serviceName":"mediad"}` |
| 12004 | CONFIG_SYNC_RESPONSE | iot_agent→mediad | `{"version":2,"config":{...}}` |

### 媒体事件（12100-12199）

| 编号 | 名称 | 方向 | 载荷 |
|------|------|------|------|
| 12100 | MEDIA_FILE_READY | mediad→iot_agent | 文件就绪通知（见下方） |
| 12101 | MEDIA_STATE_CHANGED | mediad→iot_agent | 状态变化通知 |
| 12102 | MEDIA_CONFIG_APPLIED | mediad→iot_agent | 配置已生效 |
| 12103 | MEDIA_STARTED | mediad→iot_agent | mediad 启动完成 |
| 12104 | MEDIA_HEARTBEAT_REQUEST | iot_agent→mediad | 心跳请求 |
| 12105 | MEDIA_HEARTBEAT_ACK | mediad→iot_agent | 心跳确认 |

### MEDIA_FILE_READY 载荷

拍照/录像完成后广播，通知 iot_agent 上传文件：

```json
{
    "trigger_type": "timer",
    "file_type": "image",
    "file_name": "timer_20260907_120000.jpg",
    "file_path": "/mnt/emmc/media/photos/timer_20260907_120000.jpg",
    "file_size": 125000,
    "thumb_name": "timer_20260907_120000_thumb.jpg",
    "thumb_path": "/mnt/emmc/media/photos/.thumbs/timer_20260907_120000_thumb.jpg",
    "thumb_size": 8500,
    "timestamp": 1725696000000,
    "start_time": 0,
    "duration": 0
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `trigger_type` | string | 是 | 触发源：`timer`/`bluetooth`/`manual`/`record` |
| `file_type` | string | 是 | `image` 或 `video` |
| `file_name` | string | 是 | 文件名 |
| `file_path` | string | 是 | 完整路径 |
| `file_size` | int64 | 是 | 文件大小（字节） |
| `thumb_name` | string | 否 | 缩略图文件名 |
| `thumb_path` | string | 否 | 缩略图路径 |
| `thumb_size` | int64 | 否 | 缩略图大小 |
| `timestamp` | uint64 | 是 | 生成时间戳（毫秒） |
| `start_time` | uint64 | 录像必填 | 录像起始时间（秒） |
| `duration` | uint64 | 录像必填 | 录像时长（秒） |

## 命令类型（mediad 接收）

| 编号 | 名称 | 说明 |
|------|------|------|
| 12000 | MEDIA_SNAPSHOT | 立即拍照 |
| 12010 | MEDIA_STREAM_START | 开始推流 |
| 12011 | MEDIA_STREAM_STOP | 停止推流 |
| 12030 | MEDIA_GET_STATUS | 查询服务状态 |
| 12031 | MEDIA_GET_CONFIG | 查询当前配置 |

## 交互时序

### 启动流程

```
iot_agent                    mediad
    │                           │
    │  CONFIG_SYNC_REQUEST      │
    │◄──────────────────────────│
    │                           │
    │  CONFIG_SYNC_RESPONSE     │
    │──────────────────────────►│
    │                           │
    │  CONFIG_SYNC_STARTED      │
    │◄──────────────────────────│
    │  (configVersion=1)        │
    │                           │
    │  MEDIA_STARTED            │
    │◄──────────────────────────│
    │                           │
```

### 配置更新流程

```
Cloud (MQTT)              iot_agent                    mediad
    │                         │                           │
    │  属性下发                │                           │
    │────────────────────────►│                           │
    │                         │  CONFIG_SYNC_UPDATE       │
    │                         │──────────────────────────►│
    │                         │                           │
    │                         │  CONFIG_SYNC_ACK          │
    │                         │◄──────────────────────────│
    │                         │                           │
```

### 拍照命令流程

```
mediactl/Cloud          mediad (CommandRouter)        SnapshotService
    │                           │                           │
    │  {"type":12000}           │                           │
    │──────────────────────────►│                           │
    │                           │  fireEvent()              │
    │                           │──────────────────────────►│
    │                           │                           │
    │                           │  SnapshotResult           │
    │                           │◄──────────────────────────│
    │                           │                           │
    │  {"code":0,"data":{...}}  │                           │
    │◄──────────────────────────│                           │
    │                           │                           │
    │                           │  MEDIA_FILE_READY         │
    │                           │  (PUB 广播)               │
    │                           │──────────────────────────►│
    │                           │                           │
```

## 调试

使用 `mediactl listen` 命令可以实时查看所有媒体事件：

```bash
mediactl listen
# 输出示例：
# [event 12100] {"trigger_type":"timer","file_type":"image",...}
# [event 12101] {"camera":"running","record_mode":"idle",...}
```
