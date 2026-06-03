# Mammotion Information Repository

Comprehensive documentation about Mammotion lawn mower communication protocols, focusing on the **Direct MQTT Protocol** used for cloud-based device control.

## Contents

### 📊 MQTT Protocol Schematic

File: `MQTT_PROTOCOL_SCHEMATIC.md`

Contains detailed diagrams and flows for:

- **System Architecture** - Complete overview of all components (client, HTTP API, MQTT broker, device)
- **Message Flow Sequences** - Step-by-step connection, command sending, and telemetry receiving
- **Data Transformations** - How Python commands become protobuf, get base64-encoded, wrapped in JSON, and sent
- **Credential Refresh Lifecycle** - JWT token management and expiry handling
- **Topic Hierarchy** - Complete MQTT topic structure with message directions and content types
- **Error Handling Flow** - HTTP response code handling and recovery mechanisms
- **Protocol Layer Summary** - Table showing each OSI layer involved

## Quick Reference

### Mammotion Device Connection Flow

1. **Authenticate** → OAuth login to get HTTP bearer token
2. **Get MQTT Credentials** → POST `/get_mqtt_credentials` returns JWT
3. **Connect MQTT** → Use JWT as MQTT password (port 8883 TLS)
4. **Subscribe** → Listen on `/sys/{pk}/{dn}/thing/*` topics
5. **Send Commands** → POST `/mqtt_invoke/{iot_id}` with base64-encoded protobuf
6. **Receive Updates** → Device publishes status/properties/events to subscribed topics

### Key Protocols

- **Transport**: MQTT 3.1.1 over TCP/TLS (port 8883)
- **Serialization**: Protocol Buffers (protobuf)
- **Encoding**: Base64 for binary-safe JSON transport
- **Authentication**: JWT + OAuth 2.0
- **Message Wrapper**: JSON envelopes with `params.content`

### Command Sending

```
Python Object (MammotionCommand)
    ↓
Protobuf Binary (LubaMsg)
    ↓
Base64 Encoded String
    ↓
JSON Envelope {"params": {"content": "..."}}
    ↓
HTTP POST /mqtt_invoke/{iot_id}
    ↓
Server Routes to Device via MQTT
    ↓
Device Receives, Parses, Executes
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ Human Client (PyMammotion Library)                      │
│  - MammotionClient                                      │
│  - MQTTTransport (aiomqtt wrapper)                      │
│  - TokenManager (credential management)                 │
└─────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   HTTP API         MQTT Broker    Device
   (invoke)         (broker)       (Luba/Yuka)
```

## GitHub Repositories Referenced

This documentation is based on analysis of the following PyMammotion repositories:

- **[PyMammotion](https://github.com/mikey0000/PyMammotion)** - Core Python library
- **[pyluba](https://github.com/02JanDal/pyluba)** - Python package for Luba interaction
- **[MammotionProto](https://github.com/Nithanim/MammotionProto)** - Extracted protobuf definitions
- **[mammo](https://github.com/mikeymclellan/mammo)** - Go module implementation
- **[mammotion_manager](https://github.com/runlvl/mammotion_manager)** - Web-based management system

## Key Files in PyMammotion

### Transport Layer
- `pymammotion/transport/mqtt.py` - MQTTTransport implementation
- `pymammotion/transport/ble.py` - BLE transport (alternative local protocol)
- `pymammotion/auth/token_manager.py` - Credential and JWT management

### Protocol/Commands
- `pymammotion/mammotion/commands/messages/network.py` - Network command building
- `pymammotion/mammotion/commands/messages/system.py` - System command building
- `pymammotion/proto/` - Protobuf message definitions

### Data Models
- `pymammotion/data/mqtt/` - MQTT message parsing (status, properties, events)
- `pymammotion/data/model/` - Device state models

## Protocol Versions

- **MQTT**: Version 3.1.1 (not 5.0)
- **Protocol Buffers**: v3 (protoc3)
- **OAuth 2.0**: Standard flow with bearer tokens
- **TLS**: 1.2+ required

## Authentication

### Initial Login
```
POST /login_v2
  email, password
  ↓
  Returns: access_token, refresh_token, expires_in
```

### MQTT Credentials
```
POST /get_mqtt_credentials
  Authorization: Bearer {access_token}
  ↓
  Returns: host, client_id, username, jwt, expires_at
```

### Token Refresh
- **HTTP Bearer**: Refreshed when <5min remaining
- **MQTT JWT**: Refreshed when <30min remaining
- **On Auth Failure**: Force refresh (1 retry), then re-login required

## Topic Structure

```
/sys/{product_key}/{device_name}/
├── thing/status              # Device online/offline
├── thing/properties          # Device telemetry (periodic)
├── thing/event/*/post        # Device events/notifications
├── property/post             # Alternative telemetry format
└── app/down/*/               # Commands (server-routed, not visible to clients)
```

## Rate Limiting

- **24-hour quota**: Commands subject to rate limiting
- **Heartbeats**: Exempt from quota counting
- **Response Code 20056**: Gateway timeout (retry-able)
- **Response Code 50103/50104**: Device offline

## Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| 0/200 | Success | Continue |
| 401 | Auth failed | Refresh JWT |
| 460 | Token issue | Refresh JWT |
| 6205 | Device offline | Wait & retry |
| 50103 | Device offline | Wait & retry |
| 50104 | Device offline | Wait & retry |
| 20056 | Gateway timeout | Backoff & retry |

## License

This documentation is provided as-is for educational purposes.

## References

- PyMammotion GitHub Organization
- Mammotion Official Documentation
- Protocol Buffer Specification
- MQTT 3.1.1 Specification
