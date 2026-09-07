# WebRTC 实时视频

NeuroCast 内置完整的 WebRTC 实时视频能力，支持 **P2P 直连**（单人观看）和 **SFU 服务器转发**（多人观看 / P2P 不通时兜底），设备端只做执行，不做自动降级。

## 架构总览

```
┌──────────────┐                          ┌──────────────┐
│   观看端     │                          │    设备端     │
│  (浏览器)    │                          │   (mediad)   │
│              │   MQTT 信令              │              │
│  offer ──────┼─────────────────────────►│ P2PSignaling │
│              │   p2p/{devId}/offer      │     │        │
│  answer ◄────┼──────────────────────────┤     ▼        │
│              │   p2p/{devId}/answer     │  P2PSession  │
│              │                          │  (metaRTC)   │
│  ICE cand ◄──┼──────────────────────────┤     │        │
│  ICE cand ──►┼─────────────────────────►│     ▼        │
│              │   p2p/{devId}/cand/*     │  FrameSender │
│              │                          │     │        │
│              │   WebRTC 媒体流          │     ▼        │
│              │◄═════════════════════════►│  CameraService│
│              │  (P2P 直连 / SRS 中转)   │              │
└──────────────┘                          └──────────────┘
                                                │
                                          ┌─────▼──────┐
                                          │  SRS 服务器 │
                                          │  (WHIP/WHEP)│
                                          └────────────┘
```

## 推流模式

| 模式 | 说明 | ICE 策略 | 适用场景 |
|------|------|---------|---------|
| **P2P** | WebRTC 直连 | 全候选（host + srflx + relay） | 跨网络，NAT 穿透 |
| **P2P_Host** | WebRTC 直连（仅局域网） | 仅 host 候选 | 同局域网 / 设备有公网 IP |
| **SFU** | 服务器转发（SRS） | WHIP 推到 SRS | 多人观看 / P2P 不通时兜底 |

### 模式选择

- 单人观看 → 优先 P2P（延迟最低）
- P2P 不通 → 前端控制切 SFU（多人 / 跨 NAT）
- 设备不做自动降级，由前端根据 `bye` 消息的 `reason` 字段决定下一步

## MQTT 信令协议

P2P 模式通过 MQTT 交换 SDP 和 ICE 候选。

### Topic

```
p2p/{deviceId}/offer       ← 观看端发，设备收
p2p/{deviceId}/answer      → 设备发，观看端收
p2p/{deviceId}/cand/device → 设备发 ICE 候选
p2p/{deviceId}/cand/viewer ← 观看端发 ICE 候选
p2p/{deviceId}/bye         → 设备发，会话结束通知
p2p/{deviceId}/relay       → 设备发，SFU 观看者接纳通知
p2p/{deviceId}/alive       → 心跳保活（双方互发）
```

### 信令消息格式

所有消息均为 JSON，包含公共字段 `sid`（会话 ID）和 `ts`（时间戳）：

#### offer（观看端 → 设备）

```json
{
    "sid": "session-abc-123",
    "ts": 1724659200000,
    "sdp": "v=0\r\no=- 123...",
    "mode": "p2p"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sid` | string | 是 | 会话 ID（观看端生成） |
| `ts` | number | 是 | 时间戳（毫秒） |
| `sdp` | string | 是 | SDP offer |
| `mode` | string | 否 | `"p2p"`（默认，全候选）或 `"p2p_host"`（仅局域网） |

#### answer（设备 → 观看端）

```json
{
    "sid": "session-abc-123",
    "ts": 1724659200001,
    "sdp": "v=0\r\no=- 456..."
}
```

#### candidate（双方互发）

```json
{
    "sid": "session-abc-123",
    "ts": 1724659200002,
    "candidate": "candidate:1 1 udp 2130706431 192.168.1.100 5000 typ host"
}
```

#### bye（设备 → 观看端）

```json
{
    "sid": "session-abc-123",
    "ts": 1724659230000,
    "reason": "timeout"
}
```

| reason | 含义 | 前端建议动作 |
|--------|------|-------------|
| `timeout` | P2P 连接超时（15s 无帧） | RPC `startLiveStream` 切 SFU |
| `negotiate_failed` | SDP 协商失败 | RPC `startLiveStream` 切 SFU |
| `no_camera` | 相机不可用 | 提示用户检查设备 |
| `camera_restarting` | 相机正在重启 | 等待几秒后重试 |
| `p2p_occupied` | P2P 已有观看者 | 提示用户或等待 |
| `sfu_active` | 当前是 SFU 模式 | 直接 WHEP 拉流 |
| `push_failed` | WHIP 推流失败 | 提示用户检查网络 |
| （空） | 正常结束 | 无需动作 |

#### alive（双方互发）

```json
{
    "sid": "session-abc-123",
    "ts": 1724659200000
}
```

心跳保活，防止长连接断开。SFU 模式下观看者心跳消失 30 秒后设备自动停推。

## P2P 连接流程

```
观看端                        设备 (mediad)                    SRS
  │                              │                              │
  │  offer (MQTT)                │                              │
  │─────────────────────────────►│                              │
  │                              │  handleOffer()               │
  │                              │  → 创建 PeerConnection       │
  │                              │  → setRemote(offer)          │
  │                              │  → createAnswer              │
  │                              │                              │
  │  answer (MQTT)               │                              │
  │◄─────────────────────────────│                              │
  │                              │                              │
  │  ICE candidate (MQTT)        │  ICE candidate (MQTT)        │
  │◄────────────────────────────►│                              │
  │                              │                              │
  │         WebRTC P2P 直连      │                              │
  │◄════════════════════════════►│                              │
  │         (H.264 视频流)       │                              │
```

## SFU 连接流程

```
前端 (RPC)                    设备 (mediad)                    SRS
  │                              │                              │
  │  startLiveStream             │                              │
  │  {accessToken: "xxx"}        │                              │
  │─────────────────────────────►│                              │
  │                              │  WHIP HTTP POST              │
  │                              │─────────────────────────────►│
  │                              │  ← SDP answer + ICE          │
  │                              │◄─────────────────────────────│
  │                              │                              │
  │                              │  WebRTC WHIP 推流            │
  │                              │═════════════════════════════►│
  │                              │                              │
  │  WHEP 拉流                   │                              │
  │◄────────────────────────────────────────────────────────────│
  │         (H.264 视频流)       │                              │
```

## RPC 接口

### startLiveStream

触发设备推流到 SRS（SFU 模式）。

**请求：**
```json
{
    "method": "startLiveStream",
    "params": {
        "accessToken": "platform-issued-token"
    }
}
```

**响应：**
```json
{
    "success": true,
    "message": ""
}
```

**失败：**
```json
{
    "success": false,
    "message": "whip push failed (srs unreachable?)"
}
```

### stopLiveStream

停止设备推流。

**请求：**
```json
{
    "method": "stopLiveStream",
    "params": {
        "accessToken": "platform-issued-token"
    }
}
```

`accessToken` 必须与 `startLiveStream` 时一致，否则返回 `token mismatch`。

## 状态机

设备不做自动模式切换，所有切换由前端控制。

```
         offer (P2P/P2P_Host)
    ┌───────────────────────────┐
    │                           ▼
┌───────┐                 ┌─────────┐
│ Idle  │                 │   P2P   │
│       │◄────────────────│         │
└───────┘   bye/timeout    └─────────┘
    │                           
    │ RPC startLiveStream       
    ▼                           
┌─────────┐                     
│   SFU   │                     
│         │                     
└─────────┘                     
    │                           
    │ 所有观看者心跳消失(30s)    
    │ 或 RPC stopLiveStream     
    ▼                           
  Idle                          
```

### 模式冲突处理

| 当前状态 | 新请求 | 设备行为 |
|---------|--------|--------|
| P2P | P2P（第 2 人） | 拒绝，发 bye `p2p_occupied` |
| P2P | startLiveStream RPC | 停 P2P，起 SFU |
| SFU | P2P offer | 拒绝，发 bye `sfu_active` |
| SFU | startLiveStream（同 token） | 跳过，不重复推流 |
| SFU | startLiveStream（不同 token） | 停旧推流，用新 token 重推 |

## 前端决策逻辑

设备不做 Auto 降级，前端自行实现：

```
1. 前端尝试 P2P 连接
2. 收到 bye 消息 → P2P 失败/超时
3. 根据 bye reason 决定下一步：
   - timeout / negotiate_failed → RPC startLiveStream 切 SFU
   - p2p_occupied → 当前有人在看，等待或提示用户
   - sfu_active → 已经是 SFU 模式，直接 WHEP 拉流
```

## 配置

### mediad.json 中的 live 段

```json
{
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

| 字段 | 说明 |
|------|------|
| `mqtt_signaling.url` | MQTT 信令 broker 地址 |
| `ice.host` | STUN/TURN 服务器域名 |
| `ice.port` | STUN/TURN 端口 |
| `ice.username` | TURN 长期凭证用户名 |
| `ice.password` | TURN 长期凭证密码 |
| `srs.whip_url_template` | WHIP 推流地址模板，支持 `{deviceUid}` 和 `{accessToken}` 占位符 |
| `p2p_timeout_sec` | P2P 协商超时（秒），超时后发 bye `timeout` |
| `relay_idle_sec` | SFU 模式全部心跳消失后停推窗口（秒） |

### 云端下发映射

| 云端字段 | mediad 路径 |
|---------|------------|
| `mqtt_signaling_url` | `live.mqtt_signaling.url` |
| `push_stream_url` | `live.srs.whip_url_template` |
| `ice_host` | `live.ice.host` |
| `ice_port` | `live.ice.port` |
| `ice_username` | `live.ice.username` |
| `ice_password` | `live.ice.password` |

## 组件架构

```
libs/rtc/
├── include/rtc/
│   ├── p2p_session.h      P2P 会话（answer 方，Full-ICE）
│   ├── p2p_signaling.h    MQTT 信令客户端
│   ├── whip_pusher.h      WHIP 推流器（offer 方，推 SRS）
│   ├── frame_sender.h     帧管线（P2P/SFU 共用）
│   └── ice_config.h       ICE 服务器配置
└── src/
    ├── p2p_session.cpp    metaRTC 8.0 PeerConnection 封装
    ├── p2p_signaling.cpp  MQTT 信令收发
    ├── whip_pusher.cpp    WHIP HTTP SDP 交换
    └── frame_sender.cpp   NALU 分帧与时间戳管理
```

### 关键设计

- **设备 = answer 方**（P2P）/ **offer 方**（SFU WHIP）
- **Full-ICE**：P2P 模式走完整 ICE gather + STUN/TURN；SFU 模式直连 SRS（ICE-Lite）
- **帧路由**：进入 P2P/SFU 时订阅主通道，回 Idle 时退订；帧数据直接 push 到 RTC 管线
- **端口隔离**：P2P 用 17100，WHIP 用 17000，支持共存
- **相机重启编排**：`pauseForRestart` 结束所有会话，观看端收 bye 后自行重连

## 依赖

WebRTC 基于 [metaRTC 8.0](https://github.com/metartc/metaRTC) 实现：

- `YangPeerConnection8` — WebRTC 连接管理
- `yang_whip_connectWhipWhepServer` — WHIP 推流
- Full-ICE / DTLS / SRTP / RTCP 全套支持

