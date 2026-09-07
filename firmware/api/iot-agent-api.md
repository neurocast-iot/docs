# iot_agent 接口文档

iot_agent 是 NeuroCast 的云端通信代理进程，负责 MQTT 连接、配置同步、OTA 升级、文件上传和 RPC 处理。

## 进程信息

| 属性 | 值 |
|------|-----|
| 可执行文件 | `iot_agent` |
| 配置文件 | `/etc/config/iot_agent.json` |
| 日志文件 | `/mnt/emmc/logs/iot_agent.log` |
| 启动参数 | `iot_agent [配置文件路径]` |

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                       iot_agent                              │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │CloudService │  │ConfigRouter │  │    RpcHandler       │ │
│  │  (MQTT连接) │  │ (配置路由)  │  │   (RPC处理)         │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │OtaManager   │  │FileUploadSvc│  │    IpcHub           │ │
│  │  (OTA升级)  │  │ (文件上传)  │  │   (IPC中心节点)     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────┐  ┌─────────────────────────────────────┐  │
│  │DeviceService│  │         IpcEventHandler             │  │
│  │(Zigbee/GPS) │  │   (媒体事件处理 → 云端上报)         │  │
│  └─────────────┘  └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 配置文件

```json
{
    "platform": "thingsboard",
    "mqtt": {
        "url": "ssl://iot.example.com:8883",
        "username": "",
        "password": ""
    },
    "provision_device_key": "your-device-key",
    "provision_device_secret": "your-device-secret",
    "zigbee_port": "/dev/ttySAK1",
    "ota": {
        "base_dir": "/mnt/emmc/ota"
    }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `platform` | string | 是 | 云平台类型：`thingsboard` |
| `mqtt.url` | string | 是 | MQTT broker 地址 |
| `mqtt.username` | string | 否 | MQTT 用户名 |
| `mqtt.password` | string | 否 | MQTT 密码 |
| `provision_device_key` | string | 是 | 设备预配置密钥 |
| `provision_device_secret` | string | 是 | 设备预配置密钥 |
| `zigbee_port` | string | 否 | Zigbee 串口设备路径 |
| `ota.base_dir` | string | 是 | OTA 文件存储目录 |

## MQTT 接口

### 认证流程（ThingsBoard）

iot_agent 使用 ThingsBoard 的 Device Provisioning 协议获取访问令牌：

1. 使用 `provision_device_key` 和 `provision_device_secret` 向 ThingsBoard 请求 `accessToken`
2. 使用 `accessToken` 作为 MQTT 用户名连接
3. 连接成功后订阅设备主题

### 订阅主题

| 主题 | 用途 |
|------|------|
| `v1/devices/me/attributes` | 属性下发（配置更新） |
| `v1/devices/me/rpc/request/+` | RPC 请求 |
| `v1/devices/me/ota/config` | OTA 配置 |

### 发布主题

| 主题 | 用途 |
|------|------|
| `v1/devices/me/telemetry` | 遥测数据上报 |
| `v1/devices/me/attributes` | 属性上报 |
| `v1/devices/me/ota/status` | OTA 状态上报 |

## 遥测上报

### 系统遥测

定时上报系统状态：

```json
{
    "cpu_usage": 25,
    "memory_usage": 45,
    "disk_usage": 30,
    "uptime": 86400,
    "wifi_rssi": -65,
    "camera_status": "running",
    "record_status": "idle"
}
```

### 媒体事件遥测

拍照/录像完成后上报：

```json
{
    "event_type": "media_file_ready",
    "event_source": "mediad",
    "event_time": 1725696000000,
    "trigger_type": "timer",
    "file_type": "image",
    "file_name": "timer_20260907_120000.jpg",
    "file_size": 125000
}
```

## 属性上报

### 设备属性

启动时和状态变化时上报：

```json
{
    "device_id": "abc123",
    "firmware_version": "1.0.0",
    "hardware_version": "1.0",
    "mac_address": "AA:BB:CC:DD:EE:FF",
    "ip_address": "192.168.1.100"
}
```

## RPC 接口

### 云端下发的 RPC 命令

#### camera_stream.start

开始实时观看。

**请求：**
```json
{
    "method": "camera_stream.start",
    "params": {
        "accessToken": "platform-token"
    }
}
```

**响应：**
```json
{
    "success": true
}
```

#### camera_stream.stop

停止实时观看。

**请求：**
```json
{
    "method": "camera_stream.stop",
    "params": {
        "accessToken": "platform-token"
    }
}
```

#### device.restart

重启设备。

**请求：**
```json
{
    "method": "device.restart",
    "params": {}
}
```

#### device.getConfig

获取设备当前配置。

**请求：**
```json
{
    "method": "device.getConfig",
    "params": {}
}
```

**响应：**
```json
{
    "config": {
        "device_id": "abc123",
        "camera": {...},
        "osd": {...}
    }
}
```

#### device.setConfig

更新设备配置（局部更新）。

**请求：**
```json
{
    "method": "device.setConfig",
    "params": {
        "patch": {
            "snapshot": {
                "quality": 90
            }
        }
    }
}
```

## 配置同步

iot_agent 作为配置中心，管理所有服务的配置：

```
Cloud (MQTT)              iot_agent                    mediad
    │                         │                           │
    │  属性下发                │                           │
    │  {"snapshot":{...}}     │                           │
    │────────────────────────►│                           │
    │                         │  CONFIG_SYNC_UPDATE       │
    │                         │──────────────────────────►│
    │                         │                           │
    │                         │  CONFIG_SYNC_ACK          │
    │                         │◄──────────────────────────│
    │                         │                           │
    │                         │  持久化到文件             │
    │                         │  广播给其他服务           │
```

## OTA 升级

### 升级流程

1. 云端下发 OTA 配置（固件版本、下载地址）
2. iot_agent 下载固件包到 `/mnt/emmc/ota/`
3. 校验 SHA256
4. 通知设备执行升级
5. 上报升级状态

### OTA 状态上报

```json
{
    "ota_status": "downloading",
    "ota_progress": 45,
    "ota_version": "1.1.0",
    "ota_error": ""
}
```

| 状态 | 说明 |
|------|------|
| `idle` | 无升级任务 |
| `downloading` | 下载中 |
| `verifying` | 校验中 |
| `applying` | 应用升级 |
| `completed` | 升级完成 |
| `failed` | 升级失败 |

## 文件上传

### 上传流程

1. 收到 MEDIA_FILE_READY 事件
2. 将文件加入上传队列
3. 通过 HTTP POST 上传到服务器
4. 上传成功后上报云端

### 上传状态

```json
{
    "upload_status": "uploading",
    "upload_file": "timer_20260907_120000.jpg",
    "upload_progress": 60
}
```

## IPC 接口

### 作为中心节点

iot_agent 作为 IPC 中心节点，所有服务通过 DEALER 模式连接：

| 端点 | 模式 | 说明 |
|------|------|------|
| `ipc:///tmp/iot_agent.ipc` | ROUTER | 中心节点端点 |

### 接收的事件

| 事件类型 | 来源 | 处理 |
|----------|------|------|
| MEDIA_FILE_READY | mediad | 入队上传 + 遥测上报 |
| MEDIA_STATE_CHANGED | mediad | 属性上报 |
| MEDIA_CONFIG_APPLIED | mediad | 确认配置生效 |
| MEDIA_STARTED | mediad | 记录服务启动 |

### 发送的命令

| 命令类型 | 目标 | 说明 |
|----------|------|------|
| CONFIG_SYNC_UPDATE | mediad | 配置更新 |
| HEARTBEAT | mediad | 心跳检测 |
| SERVICE_RESTART | mediad | 服务重启 |

## 心跳与监控

iot_agent 定时向 mediad 发送心跳：

```json
{
    "type": 1000,
    "processId": 1234,
    "processName": "iot_agent",
    "uptime": 86400,
    "cpuUsage": 10,
    "memoryUsage": 25
}
```

如果 mediad 长时间无响应，iot_agent 会发送 SERVICE_RESTART 命令重启服务。

## 错误处理

- MQTT 断线：自动重连，指数退避（1s → 2s → 4s → ... → 60s）
- 配置下发失败：回复 ACK 失败原因，云端可重试
- OTA 下载失败：重试 3 次，失败后上报错误
- 文件上传失败：重试 3 次，失败后跳过该文件
