# mediad 接口文档

mediad 是 NeuroCast 的媒体服务守护进程，负责相机控制、录像、拍照、OSD 水印、实时推流和触发源管理。

## 进程信息

| 属性 | 值 |
|------|-----|
| 可执行文件 | `mediad` |
| 配置文件 | `/etc/config/mediad.json` |
| 日志文件 | `/mnt/emmc/logs/mediad.log` |
| 启动参数 | `mediad [配置文件路径]` |

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                        mediad                                │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │CameraService│  │ OsdService  │  │  TriggerManager     │ │
│  │  (相机控制)  │  │  (OSD水印)  │  │  ├─ TimerTrigger    │ │
│  └─────────────┘  └─────────────┘  │  ├─ BluetoothTrigger│ │
│  ┌─────────────┐  ┌─────────────┐  │  └─ RecordTrigger   │ │
│  │RecordService│  │SnapshotSvc  │  └─────────────────────┘ │
│  │   (录像)    │  │   (拍照)    │                          │
│  └─────────────┘  └─────────────┘  ┌─────────────────────┐ │
│  ┌─────────────┐                   │   ConfigStore       │ │
│  │LiveStreamSvc│  ┌─────────────┐  │   (配置中心)        │ │
│  │  (WebRTC)   │  │ IpcClient   │  └─────────────────────┘ │
│  └─────────────┘  │  (IPC通信)  │                          │
│                   └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

## WebRTC 实时视频

mediad 内置完整的 WebRTC 实时视频能力，支持两种模式：

- **P2P 直连** — 浏览器通过 MQTT 信令与设备建立 WebRTC 直连，Full-ICE 支持 NAT 穿透，延迟最低
- **SFU 转发** — 设备通过 WHIP 推流到 SRS 服务器，观看端通过 WHEP 拉流，支持多人观看

### 快速开始

```bash
# 查看当前推流状态
mediactl status
# 输出: {"live_mode":"idle","live_viewers":0,"live_manual_push":false}

# 手动起 SFU 推流
mediactl stream start <accessToken>

# 停止推流
mediactl stream stop <accessToken>
```

详细协议说明见 [WebRTC Streaming Guide](webrtc-streaming.md)。

## IPC 端点

| 端点 | 模式 | 用途 |
|------|------|------|
| `ipc:///tmp/iot_agent.ipc` | DEALER | 连接 iot_agent（接收命令、发送事件） |
| `ipc:///tmp/nc_mediad_cmd.ipc` | REP | 调试命令通道（mediactl 直连） |
| `ipc:///tmp/nc_mediad_evt.ipc` | PUB | 事件广播通道 |

## 命令接口

### MEDIA_SNAPSHOT (12000)

立即拍照。

**请求：**
```json
{
    "type": 12000,
    "payload": {
        "quality": 80
    }
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `quality` | int | 否 | JPEG 质量 1-100，默认使用配置值 |

**应答：**
```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "path": "/mnt/emmc/media/photos/manual_20260907_120000.jpg"
    }
}
```

**错误码：**
- `-4` CameraNotReady — 相机未运行
- `-5` Internal — 拍照失败

---

### MEDIA_STREAM_START (12010)

开始实时推流（WHIP 推送到 SRS）。

**请求：**
```json
{
    "type": 12010,
    "payload": {
        "accessToken": "platform-issued-token"
    }
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `accessToken` | string | 否 | 平台下发的认证令牌 |

**应答：**
```json
{
    "code": 0,
    "message": "ok"
}
```

**错误码：**
- `-1` BadRequest — 推流功能已禁用
- `-3` Busy — 正在推流中
- `-5` Internal — 启动失败

---

### MEDIA_STREAM_STOP (12011)

停止手动推流。

**请求：**
```json
{
    "type": 12011,
    "payload": {
        "accessToken": "platform-issued-token"
    }
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `accessToken` | string | 否 | 必须与启动时的 token 一致 |

**应答：**
```json
{
    "code": 0,
    "message": "ok"
}
```

**错误码：**
- `-1` BadRequest — token 不匹配
- `-5` Internal — 未在推流

---

### MEDIA_GET_STATUS (12030)

查询各服务的运行状态。

**请求：**
```json
{
    "type": 12030
}
```

**应答：**
```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "camera": "running",
        "osd_active": true,
        "record_mode": "idle",
        "record_file": "",
        "live_mode": "idle",
        "live_viewers": 0,
        "live_manual_push": false
    }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `camera` | string | `running` 或 `stopped` |
| `osd_active` | bool | OSD 水印是否激活 |
| `record_mode` | string | `idle`/`command` |
| `record_file` | string | 当前录像文件路径 |
| `live_mode` | string | `idle`/`p2p`/`relay` |
| `live_viewers` | int | 当前观看人数 |
| `live_manual_push` | bool | 是否手动推流中 |

---

### MEDIA_GET_CONFIG (12031)

查询当前生效的全量配置。

**请求：**
```json
{
    "type": 12031
}
```

**应答：**
```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "device_id": "abc123",
        "camera": {...},
        "osd": {...},
        "snapshot": {...},
        "record": {...},
        "triggers": [...],
        "live": {...}
    }
}
```

`data` 字段内容与 `mediad.json` 配置文件格式一致。

## 事件广播

mediad 通过 PUB 通道广播以下事件：

### MEDIA_FILE_READY (12100)

媒体文件生成完成（拍照/录像）。

```json
{
    "trigger_type": "timer",
    "file_type": "image",
    "file_name": "timer_20260907_120000.jpg",
    "file_path": "/mnt/emmc/media/photos/timer_20260907_120000.jpg",
    "file_size": 125000,
    "thumb_path": "/mnt/emmc/media/photos/.thumbs/timer_20260907_120000.jpg",
    "thumb_size": 8500,
    "timestamp": 1725696000000
}
```

### MEDIA_STATE_CHANGED (12101)

服务状态变化。

```json
{
    "camera": "running",
    "osd_active": true,
    "record_mode": "idle"
}
```

### MEDIA_CONFIG_APPLIED (12102)

配置已生效。

```json
{
    "restarted": false
}
```

| 字段 | 说明 |
|------|------|
| `restarted` | 是否需要重启相机（分辨率变更时为 true） |

### MEDIA_STARTED (12103)

mediad 启动完成，所有服务初始化完毕。

## 触发源系统

触发源是拍照/录像的自动化驱动机制。

### 触发源类型

| 类型 | 说明 | 优先级 |
|------|------|--------|
| `timer` | 定时触发 | 10 |
| `bluetooth` | 蓝牙触发 | 30 |
| `sos` | 紧急触发 | 100 |
| `record` | 时间段录像 | 50 |

### 触发源配置

```json
{
    "id": "timer_snapshot",
    "type": "timer",
    "enabled": true,
    "action": "snapshot",
    "interval_sec": 300,
    "burst_count": 1,
    "burst_interval_ms": 0,
    "schedule": {
        "start_time": "08:00",
        "end_time": "18:00",
        "days": [1, 2, 3, 4, 5]
    }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 触发源唯一标识 |
| `type` | string | `timer`/`bluetooth`/`sos`/`record` |
| `enabled` | bool | 开关 |
| `action` | string | `snapshot`（拍照）或 `record`（录像） |
| `interval_sec` | int | 触发间隔（秒，仅 timer） |
| `burst_count` | int | 连拍次数 |
| `burst_interval_ms` | int | 连拍间隔（毫秒） |
| `schedule` | object | 时间表配置 |

## 线程模型

| 线程 | 职责 |
|------|------|
| 主线程 | 启动初始化、信号处理 |
| IPC 命令线程 | 处理下行命令（REQ/REP 串行） |
| 相机取帧线程 | 从 VI 获取视频帧 |
| 录像线程 | MP4 封装和写盘 |
| 推流线程 | WHIP 推流到 SRS |
| 触发源线程 | 每个启用的触发源一个线程 |
| 缩略图线程 | 延迟生成缩略图 |

## 错误处理

- 相机 SDK 失败：自动重试 3 次，间隔 1 秒
- 磁盘满：停止录像/拍照，广播 MEDIA_STATE_CHANGED 事件
- 网络断开：推流自动重连
- 配置错误：使用默认值继续运行，日志告警
