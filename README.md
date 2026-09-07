# NeuroCast 文档中心

NeuroCast 是一个开源的 IoT 设备管理平台，涵盖设备端固件、云端服务、Web 管理平台三大部分。

## 文档导航

### 中文版

| 模块 | 说明 |
|------|------|
| [服务端文档](cn/server/README.md) | 系统架构、API 接口、设备协议 |
| [固件端文档](cn/firmware/architecture.md) | 嵌入式固件架构、API 参考、开发指南 |
| [管理平台截图](cn/console/) | Web 管理界面功能展示 |

### 英文版

| 模块 | 说明 |
|------|------|
| [Server Documentation](server/README.md) | Architecture, API reference, device protocols |
| [Firmware Documentation](firmware/architecture.md) | Embedded firmware architecture, API reference, guides |

## 项目概览

```
NeuroCast IoT Platform
├── server/          # 云端服务（Java / Spring Boot）
│   ├── 设备管理      # 产品、设备 CRUD、配置下发
│   ├── 媒体服务      # 实时直播、录像回放、媒体库
│   ├── 系统管理      # 用户、角色、权限（RBAC）
│   └── 定时任务      # HLS 清理、媒体清理、推流超时
│
├── firmware/        # 设备端固件（C++ / Linux）
│   ├── 媒体服务      # 录像、拍照、OSD 水印、WebRTC 推流
│   ├── 云端通信      # MQTT、配置同步、OTA 升级、文件上传
│   └── 触发自动化    # 定时器、蓝牙、录像联动
│
└── console/         # Web 管理平台截图
    └── imgs/        # 界面截图资源
```

## 相关仓库

| 仓库 | 说明 |
|------|------|
| [neurocast-iot/server](https://github.com/neurocast-iot/server) | 服务端源码（Java） |
| [neurocast-iot/docs](https://github.com/neurocast-iot/docs) | 项目文档（本仓库） |

## 许可证

本项目采用 [Apache License 2.0](https://github.com/neurocast-iot/server/blob/main/LICENSE) 许可证。
