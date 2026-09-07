# NeuroCast Documentation

Official documentation for the NeuroCast open-source project.

## Firmware (Device)

Documentation for the embedded firmware (C++ framework running on device hardware).

### API Reference

| Document | Description |
|----------|-------------|
| [mediad API](firmware/api/mediad-api.md) | Media daemon — camera, recording, streaming, OSD, triggers |
| [iot_agent API](firmware/api/iot-agent-api.md) | Cloud agent — MQTT, config sync, OTA, file upload, RPC |
| [IPC Protocol](firmware/api/ipc-protocol.md) | Inter-process communication protocol (ZeroMQ) |
| [OSD Elements Config](firmware/api/osd-elements-config-api.md) | OSD watermark configuration API (server/cloud integration) |
| [Triggers Config](firmware/api/triggers-config-api.md) | Trigger automation configuration (timer, bluetooth, record) |
| [Push Streaming Protocol](firmware/api/push-streaming-protocol.md) | P2P/SFU signaling protocol specification |

### Guides

| Document | Description |
|----------|-------------|
| [WebRTC Streaming](firmware/guides/webrtc-streaming.md) | P2P/SFU modes, MQTT signaling, WHIP/WHEP protocol |
| [Configuration Reference](firmware/guides/config-reference.md) | All configuration options (mediad.json, iot_agent.json) |
| [OSD Watermark Architecture](firmware/guides/osd-watermark-architecture.md) | OSD overlay engine design and cross-resolution support |

## Platform (Server)

*Coming soon — Java backend documentation.*

## Shared

*Coming soon — Cross-cutting protocol specifications and conventions.*
