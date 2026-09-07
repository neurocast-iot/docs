# NeuroCast Firmware 架构文档

本文档从全局视角描述 NeuroCast 固件的整体架构、模块职责、进程间通信方式和平台抽象策略。各模块的详细接口请查阅 `api/` 和 `guides/` 下的专题文档。

---

## 系统概述

NeuroCast 是一个嵌入式 IP 摄像头固件框架，运行在 Linux 设备上，提供：

- WebRTC 实时视频推流（P2P 直连 + SFU 转发）
- 媒体服务（录像、拍照、OSD 水印）
- 云端对接（MQTT 上报、配置同步、OTA 升级、文件上传）
- 触发源自动化（定时器、蓝牙、录像联动）

---

## 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│  apps/                  应用层                                │
│    mediad               媒体服务守护进程                       │
│    iot_agent            云端通信代理进程                       │
│    iot_monitor          系统监控进程（CPU/内存/磁盘）           │
├─────────────────────────────────────────────────────────────┤
│  libs/                  通用库层                              │
│    camera/              相机驱动抽象                           │
│    recorder/            录像驱动抽象                           │
│    osd/                 OSD 水印渲染引擎                      │
│    common/              公共工具（日志、文件操作等）             │
│    http/                HTTP 客户端与下载策略（基于 libcurl）   │
│    mqtt/                MQTT 客户端封装（基于 Paho）           │
│    mq/                  进程间消息队列（基于 ZeroMQ）           │
│    config_sync/         配置可靠下发协议                       │
│    sensor/              传感器驱动（蓝牙、Zigbee）             │
│    exporter/            指标导出                               │
│    rtc/                 WebRTC 引擎封装（基于 metaRTC）        │
├─────────────────────────────────────────────────────────────┤
│  pal/                   平台抽象层                            │
│    include/pal/         纯接口定义（零厂商依赖）                │
│    backends/            各平台的具体实现                        │
│      linux-generic/     通用 Linux 后端（软件渲染 OSD 等）     │
│      mock/              单元测试桩                             │
└─────────────────────────────────────────────────────────────┘
```

**依赖方向严格单向**：`apps → libs → pal 接口 ← backends`

铁律：`apps/` 和 `libs/` 中永远不出现厂商 SDK 头文件（如 `#include <ak_xxx.h>`）。所有平台特定代码收敛于 `pal/backends/` 目录。

---

## 两个核心进程

设备上运行两个主要进程，各司其职：

### mediad — 媒体服务守护进程

负责所有与摄像头和媒体相关的功能：

| 服务 | 职责 |
|------|------|
| CameraService | 相机通道控制（主/子通道、分辨率、帧率） |
| RecordService | MP4 录像（分段录制、可配时长） |
| SnapshotService | JPEG 拍照（手动触发 + 自动触发） |
| LiveStreamService | WebRTC 实时推流（P2P / SFU 两种模式） |
| OsdService | OSD 水印叠加（时间、文字、图形、位图） |
| TriggerManager | 触发源调度（定时器、蓝牙、录像联动，优先级排队） |
| ConfigStore | 配置中心，热更新无需重启 |
| IpcClient | IPC 通信客户端 |

### iot_agent — 云端通信代理进程

负责设备与云端之间的所有通信：

| 服务 | 职责 |
|------|------|
| CloudService | MQTT 连接管理（ThingsBoard 等平台） |
| IpcHub | IPC 中心节点（ROUTER 模式，等 mediad 等进程连上来） |
| ConfigRouter | 云端配置字段翻译与路由 |
| IpcEventHandler | 接收 mediad 事件 → 转发云端上报 |
| RpcHandler | 云端 RPC 命令处理（拍照、推流等转发给 mediad） |
| OtaManager | OTA 固件升级（下载 → 校验 → 烧写） |
| FileUploadService | 文件上传（拍照/录像文件传云端） |
| DeviceService | 设备信息采集（Zigbee、GPS 等） |

---

## 进程间通信（IPC）

两个进程通过 ZeroMQ 通信。iot_agent 是中心节点，mediad 连上来接命令、报事件：

```
                       ┌──────────────┐
                       │    Cloud     │
                       └──────┬───────┘
                              │ MQTT
                              ▼
                ┌─────────────────────────────┐
                │        iot_agent            │
                │       （中心节点）            │
                │                             │
                │  ROUTER  ← 命令下发/应答     │
                │  SUB     ← 事件订阅          │
                └─────────┬───────────────────┘
                          │
            ┌─────────────┼───────────────┐
            │                             │
            ▼                             │
  ┌───────────────────┐                   │
  │      mediad       │                   │
  │                   │                   │
  │  DEALER ──────────┼── 命令通道         │
  │  (连 iot_agent)   │   ipc:///tmp/     │
  │                   │   iot_agent.ipc   │
  │                   │                   │
  │  PUB ─────────────┼── 事件广播         │
  │  (广播给订阅者)    │   ipc:///tmp/     │
  │                   │   nc_mediad_      │
  │                   │   evt.ipc         │
  │                   │                   │
  │  REP ─────────────┼── 调试直连         │
  │  (本地调试用)      │   ipc:///tmp/     │
  │                   │   nc_mediad_      │
  │                   │   cmd.ipc         │
  └───────────────────┘                   │
```

两套拓扑并存：

- **命令通道（星型）**：iot_agent 居中（ROUTER），所有业务进程作为 spoke（DEALER）连上来。命令下发、配置同步、心跳监控都走这个星型结构。未来新增 monitor 等服务，只需再连一个 DEALER 上来
- **事件通道（发布-订阅）**：mediad 自己广播（PUB），谁想听谁订（SUB），不经过中心节点路由

```
            monitor (DEALER)
                 │
                 ▼
mediad ──── iot_agent ──── 未来其他服务
(DEALER)   (ROUTER)
```

### 为什么是两条主通道而不是一条？

两种通信模式的需求不同：

| 通道 | ZMQ 模式 | 用途 | 为什么需要 |
|------|----------|------|-----------|
| `iot_agent.ipc` | ROUTER/DEALER | 命令下发 + 应答 | 点对点，一问一答有明确对象（iot_agent 发拍照命令 → mediad 回结果） |
| `nc_mediad_evt.ipc` | PUB/SUB | 事件广播 | 广播模式，mediad 拍完照发事件，不需要知道谁在听——iot_agent 听到后上传文件，将来其他进程也能直接订阅 |

如果硬塞进一条通道，要么丢失广播能力，要么点对点应答没法正确路由。

### 为什么 iot_agent 是中心节点？

由系统角色决定：

- **iot_agent 是"管家"** — 连云、管配置同步、管文件上传、管 OTA、管心跳监控。它需要跟所有业务进程打交道，所以用 ROUTER 绑一个端点，谁要接入就连上来
- **mediad 是"干活的"** — 只管媒体，不需要主动协调其他进程。它只需要连上 iot_agent 接命令、开 PUB 广播事件

代码中已预留多服务接入：

```cpp
m_configSync->registerService("mediad");
// 未来新增服务只需加一行 registerService
```

---

## 平台抽象策略

NeuroCast 通过 PAL（Platform Abstraction Layer）实现多平台支持，核心思路：

### 接口与实现分离

`pal/include/pal/` 定义纯 C++ 接口，零厂商依赖：

| 接口 | 职责 |
|------|------|
| `IOsdBackend` | OSD 位图绘制、色表设置 |
| `IFwFlash` | 固件分区读写 |
| `ISystemInfo` | 平台信息探针（内存、CPU、磁盘等） |

`pal/backends/<platform>/` 放各平台的具体实现。当前有：

| 后端 | 说明 |
|------|------|
| `linux-generic/` | 通用 Linux 后端，软件渲染 OSD、文件模拟固件烧写，用于 x86 开发调试 |
| `mock/` | 单元测试桩，所有接口返回模拟数据 |
| `anyka-av100/` | （未开源）Anychip 平台后端，唯一允许引用厂商 SDK 头文件的目录 |

### 新增一个平台只需三步

1. 新建 `pal/backends/<platform>/`，实现 `pal/include/pal/*.h` 接口
2. 新建 `libs/camera/src/<platform>/` 和 `libs/recorder/src/<platform>/`，实现驱动
3. 在 `CMakePresets.json` 加一个构建预设

apps 和 libs 层代码零修改。

### 构建时平台选择

通过 CMake 变量 `NC_PLATFORM` 在构建时选择唯一后端，无运行时开销：

```bash
# x86 宿主机（开发 + 单元测试）
cmake --preset x86-debug && cmake --build --preset x86-debug

# 板端固件（以 anyka-av100 为例）
cmake --preset arm-ak3918 && cmake --build --preset arm-ak3918
```

---

## 构建系统

| 组件 | 说明 |
|------|------|
| CMake 最低版本 | 3.20+ |
| C++ 标准 | C++14 |
| 依赖管理 | FetchContent 按版本拉取（源码不入库） |
| 平台预设 | `CMakePresets.json` 集中管理工具链 + `NC_PLATFORM` 变量 |
| 工具链 | `cmake/toolchains/` 集中管理，消除各模块重复维护 |

### 门控编译

部分库和应用依赖硬件驱动，按平台门控：

```cmake
# libs/CMakeLists.txt — 硬件相关库仅在平台可用时编译
if(NC_PLATFORM STREQUAL "anyka-av100")
    add_subdirectory(camera)
    add_subdirectory(recorder)
    add_subdirectory(rtc)
    add_subdirectory(mqtt)
endif()
```

`iot_monitor` 等不依赖硬件的模块在所有平台都编译。

---

## 数据流概览

### 拍照流程

```
触发源(Timer/蓝牙/手动)
    │
    ▼
TriggerManager → SnapshotService → CameraService(拍子通道)
                                       │
                                       ▼
                                  JPEG 文件落盘
                                       │
                    ┌──────────────────┤
                    ▼                  ▼
            MEDIA_FILE_READY      文件留在磁盘
            (PUB 广播)                  │
                    │                   │
                    ▼                   │
            iot_agent 收到事件          │
                    │                   │
                    ▼                   │
            FileUploadService ──────────┘
            (上传到云端)
```

### 配置下发流程

```
Cloud (MQTT 属性下发)
    │
    ▼
iot_agent / CloudService
    │
    ▼
ConfigRouter (翻译云端字段名为 mediad 配置结构)
    │
    ▼
IpcHub / ConfigSyncManager (可靠下发，跟踪服务在线状态)
    │
    ▼  ipc:///tmp/iot_agent.ipc (ROUTER → DEALER)
mediad 收到配置 → 热更新，无需重启
    │
    ▼
MEDIA_CONFIG_APPLIED (事件回报)
```

### WebRTC 推流流程

```
前端 / 云端 发推流命令 (MQTT RPC)
    │
    ▼
iot_agent / RpcHandler → IPC 转发 → mediad / LiveStreamService
                                          │
                              ┌───────────┴───────────┐
                              ▼                       ▼
                         P2P 模式                 SFU 模式
                              │                       │
                    浏览器 Full-ICE 直连      WHIP 推流到 SRS 服务器
                    (延迟最低，单人)          (多人观看，NAT 穿透)
```

---

## 开源版本说明

| 包含 | 不包含 |
|------|--------|
| 全部框架接口（`pal/include/`、`libs/*/include/`） | 厂商 SDK 后端实现（`anyka-av100/`） |
| `linux-generic` 通用后端 | 厂商专用工具链 |
| `mock` 测试桩 | 设备端部署配置 |
| 全部应用源码（mediad、iot_agent） | |
| 完整单元测试套件 | |
| 触发系统、OSD 引擎、配置同步协议 | |
