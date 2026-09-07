# NeuroCast 中文文档

NeuroCast 开源项目的官方中文文档。

## 服务端（Server）

NeuroCast 服务端是基于 Spring Boot 3 / Java 21 构建的 IoT 设备管理后端。

### 架构文档

- [系统架构](server/architecture.md) — 系统概述、功能模块、技术栈、认证机制、响应格式

### API 接口文档

| 文档 | 说明 |
|------|------|
| [API 概览](server/api/overview.md) | 认证方式、统一响应格式、错误码总表 |

### 设备协议文档

| 文档 | 说明 |
|------|------|
| [OSD 水印配置协议](server/protocol/osd_elements_config_api.md) | OSD 水印配置 API（云端→设备） |
| [事件触发器配置协议](server/protocol/triggers_config_api.md) | 触发器自动化配置（定时器、蓝牙、录像） |

---

## 固件端（Firmware）

NeuroCast 固件是运行在 Linux 设备上的嵌入式 C++ 框架。

### API 参考

| 文档 | 说明 |
|------|------|
| [mediad API](firmware/api/mediad-api.md) | 媒体守护进程 — 相机、录像、推流、OSD、触发器 |
| [iot_agent API](firmware/api/iot-agent-api.md) | 云端代理 — MQTT、配置同步、OTA、文件上传、RPC |
| [IPC 通信协议](firmware/api/ipc-protocol.md) | 进程间通信协议（ZeroMQ） |
| [OSD 水印配置](firmware/api/osd-elements-config-api.md) | OSD 水印配置 API（服务端/云端集成） |
| [触发器配置](firmware/api/triggers-config-api.md) | 触发器自动化配置（定时器、蓝牙、录像） |
| [推流协议](firmware/api/push-streaming-protocol.md) | P2P/SFU 信令协议规范 |

### 开发指南

| 文档 | 说明 |
|------|------|
| [WebRTC 推流](firmware/guides/webrtc-streaming.md) | P2P/SFU 模式、MQTT 信令、WHIP/WHEP 协议 |
| [配置参考](firmware/guides/config-reference.md) | 全部配置项说明（mediad.json、iot_agent.json） |
| [OSD 水印架构](firmware/guides/osd-watermark-architecture.md) | OSD 叠加引擎设计与跨分辨率支持 |

---

## 管理平台界面展示（Console）

Web 管理平台的功能界面截图，展示各模块的操作界面。

### 首页概览

![仪表盘](console/imgs/dashboard.png)

### 设备管理

| 功能 | 截图 |
|------|------|
| 设备列表 | ![设备列表](console/imgs/device_list.png) |
| 设备配置 | ![设备配置](console/imgs/device_list_config.png) |
| 设备操作 | ![设备操作](console/imgs/device_list_opt.png) |
| OSD 水印配置 | ![OSD 配置](console/imgs/device_list_osd.png) |
| 触发器配置 | ![触发器](console/imgs/device_list_plan.png) |
| 产品管理 | ![产品管理](console/imgs/product.png) |

### 实时直播与回放

| 功能 | 截图 |
|------|------|
| 实时监控 | ![实时监控](console/imgs/realtime.png) |
| 实时直播（续） | ![实时监控2](console/imgs/realtime2.png) |
| 录像回放 | ![录像回放](console/imgs/playback.png) |

### 媒体库

| 功能 | 截图 |
|------|------|
| 图片列表 | ![图片列表](console/imgs/lib_image.png) |
| 图片详情 | ![图片详情](console/imgs/lib_image2.png) |
| 录像列表 | ![录像列表](console/imgs/lib_video.png) |
| 录像详情 | ![录像详情](console/imgs/lib_video2.png) |

### 系统管理

| 功能 | 截图 |
|------|------|
| 用户管理 | ![用户管理](console/imgs/sys_user.png) |
| 角色管理 | ![角色管理](console/imgs/sys_role.png) |
| API 客户端管理 | ![API 客户端](console/imgs/sys_api_client.png) |
| 权限管理 | ![权限管理](console/imgs/sys_pem.png) |
