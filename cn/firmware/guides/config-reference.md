# 配置参考文档

本文档描述 NeuroCast 的所有配置项，包括 `mediad.json` 和 `iot_agent.json`。

## 配置文件位置

| 进程 | 配置文件 | 说明 |
|------|----------|------|
| mediad | `/etc/config/mediad.json` | 媒体服务配置 |
| iot_agent | `/etc/config/iot_agent.json` | 云端代理配置 |

配置文件支持运行时热更新，无需重启服务。

## mediad.json

### 完整示例

```json
{
    "device_id": "abc123",
    "camera": {
        "auto_start": true,
        "sensor_config": "/etc/isp_gc4653_mipi_2lane_av100.conf",
        "main": {
            "width": 1920,
            "height": 1080,
            "fps": 25,
            "bitrate_kbps": 2048,
            "codec": "h264",
            "br_mode": "vbr"
        },
        "sub": {
            "width": 640,
            "height": 480,
            "fps": 15,
            "bitrate_kbps": 512
        }
    },
    "osd": {
        "enabled": true,
        "font_file": "/usr/fonts/ak_font_16.bin",
        "elements": [
            {
                "id": "time_main",
                "type": "time",
                "enabled": true,
                "x": 12,
                "y": 22,
                "size": "medium",
                "format": "YYYY-MM-DD",
                "show_week": false
            }
        ]
    },
    "snapshot": {
        "output_dir": "/mnt/emmc/media/photos",
        "quality": 80
    },
    "thumbnail": {
        "enabled": true,
        "width": 320,
        "height": 176,
        "quality": 60
    },
    "record": {
        "segment_sec": 300,
        "output_dir": "/mnt/emmc/media/records"
    },
    "triggers": [
        {
            "id": "timer_snapshot",
            "type": "timer",
            "enabled": true,
            "action": "snapshot",
            "priority": 10,
            "interval_sec": 300,
            "burst_count": 1,
            "burst_interval_ms": 0,
            "schedule": {
                "start_time": "08:00",
                "end_time": "18:00",
                "days": [1, 2, 3, 4, 5]
            }
        }
    ],
    "ipc": {
        "iot_agent_endpoint": "ipc:///tmp/iot_agent.ipc"
    },
    "live": {
        "enabled": true,
        "mqtt_signaling": {
            "url": "tcp://iot.example.com:1883",
            "username": "",
            "password": ""
        },
        "ice": {
            "host": "turn.example.com",
            "port": 3478,
            "username": "turnuser",
            "password": "turnpass"
        },
        "srs": {
            "whip_url_template": "http://srs.example.com:1985/rtc/v1/whip/?app=live&stream={deviceUid}&accessToken={accessToken}"
        },
        "p2p_timeout_sec": 15,
        "relay_idle_sec": 30
    }
}
```

### 配置项详解

#### 顶层配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `device_id` | string | `""` | 设备唯一标识，由 iot_agent 通过 IPC 下发 |

---

#### camera — 相机配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `auto_start` | bool | `true` | 启动后自动开流 |
| `sensor_config` | string | `/etc/isp_gc4653_mipi_2lane_av100.conf` | ISP 配置文件路径 |
| `main` | object | — | 主通道配置（录像/推流） |
| `sub` | object | — | 子通道配置（拍照隔离通道） |

**通道配置（main/sub）：**

| 字段 | 类型 | 默认值 | 约束 | 说明 |
|------|------|--------|------|------|
| `width` | int | 1920/640 | 32 的倍数，32-1920 | 宽度（像素） |
| `height` | int | 1080/480 | 8 的倍数，8-1080 | 高度（像素） |
| `fps` | int | 25/15 | 1-30 | 帧率 |
| `bitrate_kbps` | int | 2048/512 | 100-8192 | 码率（kbps） |
| `codec` | string | `h264` | `h264`/`h265` | 编码格式 |
| `br_mode` | string | `vbr` | `vbr`/`cbr` | 码率控制模式 |

**注意：** 修改分辨率后相机会重启，OSD 水印会重新创建。

---

#### osd — OSD 水印配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | `true` | OSD 总开关 |
| `font_file` | string | `/usr/fonts/ak_font_16.bin` | 点阵字库路径 |
| `elements` | array | `[]` | 水印元素列表 |

**水印元素配置：**

| 字段 | 类型 | 适用类型 | 说明 |
|------|------|----------|------|
| `id` | string | 所有 | 元素唯一标识 |
| `type` | string | 所有 | `time`/`label`/`rect`/`circle`/`ellipse`/`polygon`/`bitmap` |
| `enabled` | bool | 所有 | 元素开关（默认 `true`） |
| `x` | int | 所有 | 千分比 X 坐标（0-1000） |
| `y` | int | 所有 | 千分比 Y 坐标（0-1000） |
| `text` | string | `label` | 固定文本内容 |
| `size` | string | `time`/`label` | 字号档位：`small`/`medium`/`large` |
| `format` | string | `time` | 日期格式：`YYYY-MM-DD`/`MM-DD-YYYY`/`Chinese` |
| `show_week` | bool | `time` | 是否显示星期（默认 `false`） |
| `w` | int | `rect`/`ellipse`/`bitmap` | 宽度（千分比） |
| `h` | int | `rect`/`ellipse`/`bitmap` | 高度（千分比） |
| `r` | int | `circle` | 半径（千分比） |
| `color` | string | 图形类 | 颜色：`black`/`white`/`red`/`green`/`blue`/`yellow` |
| `opacity` | int | 图形类 | 透明度 0-100（0=不透明，-1=使用默认值） |
| `points` | array | `polygon` | 顶点坐标数组（千分比），至少 3 个，最多 16 个 |
| `image_path` | string | `bitmap` | BMP 文件路径（24 位无压缩） |

**千分比说明：** 坐标和尺寸使用千分比（0-1000），表示占画面宽/高的千分之几。例如 `x: 12, y: 22` 表示左上角位于画面宽度的 1.2%、高度的 2.2% 处。

---

#### snapshot — 拍照配置

| 字段 | 类型 | 默认值 | 约束 | 说明 |
|------|------|--------|------|------|
| `output_dir` | string | `/mnt/emmc/media/photos` | — | 照片输出目录 |
| `quality` | int | `80` | 1-100 | JPEG 质量 |

---

#### thumbnail — 缩略图配置

| 字段 | 类型 | 默认值 | 约束 | 说明 |
|------|------|--------|------|------|
| `enabled` | bool | `true` | — | 缩略图总开关 |
| `width` | int | `320` | 160-640 | 缩略图宽度 |
| `height` | int | `176` | 90-360 | 缩略图高度 |
| `quality` | int | `60` | 30-90 | JPEG 质量 |

---

#### record — 录像配置

| 字段 | 类型 | 默认值 | 约束 | 说明 |
|------|------|--------|------|------|
| `segment_sec` | int | `300` | 30-3600 | 单段 MP4 时长（秒） |
| `output_dir` | string | `/mnt/emmc/media/records` | — | 录像输出目录 |

**注意：** 分段时长过短会导致文件切换频繁，过长会导致单文件过大。

---

#### triggers — 触发源配置

触发源是数组格式，支持多个触发源同时存在。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | string | — | 触发源唯一标识 |
| `type` | string | — | `timer`/`bluetooth`/`sos`/`record` |
| `enabled` | bool | `false` | 开关 |
| `action` | string | — | `snapshot`（拍照）或 `record`（录像） |
| `priority` | int | 按类型 | 优先级（数字越大越高） |
| `interval_sec` | int | `300` | 触发间隔（秒，仅 timer） |
| `burst_count` | int | `1` | 连拍次数 |
| `burst_interval_ms` | int | `0` | 连拍间隔（毫秒） |
| `schedule` | object | — | 时间表配置 |

**默认优先级：**

| 类型 | 默认优先级 |
|------|------------|
| `timer` | 10 |
| `bluetooth` | 30 |
| `record` | 50 |
| `sos` | 100 |

**时间表配置（schedule）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `start_time` | string | 起始时间（如 `08:00`） |
| `end_time` | string | 结束时间（如 `18:00`） |
| `days` | array | 生效的星期几（0=周日, 1=周一, ...） |

---

#### ipc — IPC 端点配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `iot_agent_endpoint` | string | `ipc:///tmp/iot_agent.ipc` | iot_agent 的 ROUTER 端点 |

---

#### live — 实时视频配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | `true` | 实时视频总开关 |
| `mqtt_signaling` | object | — | MQTT 信令配置 |
| `ice` | object | — | STUN/TURN 服务器配置 |
| `srs` | object | — | SRS 推流配置 |
| `p2p_timeout_sec` | int | `15` | P2P 协商超时（秒） |
| `relay_idle_sec` | int | `30` | Relay 模式空闲超时（秒） |

**MQTT 信令配置：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | string | EMQX broker 地址 |
| `username` | string | MQTT 用户名 |
| `password` | string | MQTT 密码 |

**ICE 服务器配置：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `host` | string | coturn 域名 |
| `port` | int | 端口（STUN/TURN 同端口） |
| `username` | string | TURN 用户名 |
| `password` | string | TURN 密码 |

**SRS 配置：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `whip_url_template` | string | WHIP 推流地址模板，支持 `{deviceUid}` 和 `{accessToken}` 占位符 |

---

## iot_agent.json

### 完整示例

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

### 配置项详解

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `platform` | string | 是 | 云平台类型，目前只支持 `thingsboard` |
| `mqtt.url` | string | 是 | MQTT broker 地址（支持 `tcp://` 和 `ssl://`） |
| `mqtt.username` | string | 否 | MQTT 用户名（ThingsBoard 使用 accessToken） |
| `mqtt.password` | string | 否 | MQTT 密码 |
| `provision_device_key` | string | 是 | 设备预配置密钥（ThingsBoard 创建） |
| `provision_device_secret` | string | 是 | 设备预配置密钥 |
| `zigbee_port` | string | 否 | Zigbee 串口设备路径 |
| `ota.base_dir` | string | 是 | OTA 文件存储目录 |

---

## 配置更新

### 通过 mediactl 更新

```bash
# 更新单个字段
mediactl config set '{"snapshot":{"quality":90}}'

# 更新多个字段
mediactl config set '{"snapshot":{"quality":90},"record":{"segment_sec":600}}'

# 更新数组元素（通过 id 匹配）
mediactl config set '{"triggers":[{"id":"timer_snapshot","interval_sec":60}]}'
```

### 通过云端更新

云端通过 MQTT 下发属性，iot_agent 收到后转发给 mediad：

```json
// 云端下发
{"snapshot": {"quality": 90}}

// iot_agent 转发 CONFIG_SYNC_UPDATE
// mediad 应用配置并回复 CONFIG_SYNC_ACK
```

### 配置持久化

- 配置更新后立即生效
- 自动持久化到 `/etc/config/mediad.json`
- 使用原子写操作，断电安全
- 下次启动自动加载

---

## 配置校验

配置加载时会进行校验，不合法的字段会使用默认值：

| 校验项 | 规则 | 失败处理 |
|--------|------|----------|
| 分辨率 | 宽 32 对齐、高 8 对齐、范围限制 | 使用默认值 |
| 帧率 | 1-30 | 使用默认值 |
| 码率 | 100-8192 kbps | 使用默认值 |
| JPEG 质量 | 1-100 | 使用默认值 |
| 分段时长 | 30-3600 秒 | 使用默认值 |
| 缩略图尺寸 | 宽 160-640、高 90-360 | 使用默认值 |

校验失败会在日志中输出警告：

```
[WARN] ConfigStore: camera.main.width=33 invalid, using default 1920
```

---

## 默认配置

如果配置文件不存在或解析失败，mediad 会使用内置默认配置继续运行，并生成一份默认配置文件方便修改。

默认配置特点：

- 相机：1080p@25fps 主通道 + 480p@15fps 子通道
- OSD：启用时间水印，位置左上角
- 拍照：质量 80，输出到 `/mnt/emmc/media/photos`
- 录像：分段 300 秒，输出到 `/mnt/emmc/media/records`
- 触发源：默认禁用
- 实时视频：启用，P2P 优先
