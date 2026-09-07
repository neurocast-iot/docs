# NeuroCast Server 文档

本目录包含 NeuroCast 服务端（Java 后端）的架构文档和 API 接口文档。

## 文档目录

### 架构文档

- [architecture.md](architecture.md) — 系统架构、功能模块、技术栈、认证机制、响应格式

### API 接口文档

- [api/overview.md](api/overview.md) — API 概览（认证方式、响应格式、错误码总表）
- **完整 API 接口参考**：由于接口数量较多（20+ 个模块，100+ 个端点），完整的 API 接口文档请参考项目根目录的 `docs/API接口文档.md`（已去除 OTA 相关内容）

### 设备协议文档

- [protocol/osd_elements_config_api.md](protocol/osd_elements_config_api.md) — OSD 水印配置协议（云端→设备）
- [protocol/triggers_config_api.md](protocol/triggers_config_api.md) — 事件触发器配置协议（云端→设备）

## 快速导航

| 功能模块 | API 路径前缀 | 说明 |
|---|---|---|
| 认证 | `/api/auth` | 登录/登出/刷新令牌/当前用户 |
| 用户管理 | `/api/admin/system/users` | 用户 CRUD |
| 角色管理 | `/api/admin/system/roles` | 角色 CRUD |
| 权限管理 | `/api/admin/system/permissions` | 权限 CRUD |
| API 客户端 | `/api/admin/system/api-clients` | 客户端管理 |
| 系统配置 | `/api/admin/system/config` | 服务端 KV 配置 |
| 产品管理 | `/api/admin/product` | 设备类型管理 |
| 设备管理 | `/api/admin/device` | 设备 CRUD |
| 设备配置 | `/api/admin/device/config` | 设备参数/OSD/触发器 |
| 设备指令 | `/api/admin/device/command` | FRP/SSH/重启/复位 |
| 直播流 | `/api/admin/media/stream` | 实时直播控制 |
| 媒体库 | `/api/admin/media` | 图片/录像查询 |
| HLS 回放 | `/api/admin/media/playback` | 录像回放 |
| 文件上传 | `/api/admin/file/upload` | 分片/批量上传 |
| 文件下载 | `/api/file/download` | 流式下载 |
| C 端认证 | `/api/app/auth` | 会员登录 |
| 会员资料 | `/api/app/user` | 会员信息管理 |

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [neurocast-server](https://github.com/neurocast-iot/neurocast-server) | 服务端源码 |
| [neurocast-firmware](https://github.com/neurocast-iot/neurocast-firmware) | 设备端固件 |
| [neurocast-platform](https://github.com/neurocast-iot/neurocast-platform) | Web 管理平台 |
