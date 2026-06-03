# Mammotion Direct MQTT Protocol Schematic

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         HUMAN CLIENT                                     │
│                    (PyMammotion Library)                                │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ MammotionClient                                                  │  │
│  │  ├─ authenticate with Mammotion servers (OAuth)                │  │
│  │  ├─ get_mammotion_mqtt_credentials() → MQTTCredentials         │  │
│  │  └─ register_device() with product_key, device_name            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                  │                                       │
│                    ┌─────────────┼─────────────┐                        │
│                    │             │             │                        │
│           ┌────────▼──────┐  ┌──▼──────────┐  │                        │
│           │ TokenManager  │  │ DeviceHandle│  │                        │
│           │               │  │             │  │                        │
│           │ • HTTP OAuth  │  │ • MowerInfo │  │                        │
│           │ • MQTT JWT    │  │ • Map data  │  │                        │
│           │ • Aliyun IoT  │  │ • Status    │  │                        │
│           └────────┬──────┘  └──┬──────────┘  │                        │
│                    │             │             │                        │
│                    └─────────────┼─────────────┘                        │
│                                  │                                       │
│                    ┌─────────────▼─────────────┐                        │
│                    │    MQTTTransport          │                        │
│                    │  (aiomqtt wrapper)        │                        │
│                    │                           │                        │
│                    │  Config:                  │                        │
│                    │  • host: broker hostname  │                        │
│                    │  • port: 1883/8883        │                        │
│                    │  • username: account      │                        │
│                    │  • password: JWT token    │                        │
│                    │  • client_id: unique ID   │                        │
│                    │                           │                        │
│                    │  Methods:                 │                        │
│                    │  • connect()              │                        │
│                    │  • send(payload, iot_id)  │                        │
│                    │  • subscribe(topic)       │                        │
│                    │  • on_message callback    │                        │
│                    │  • on_device_notification │                        │
│                    └─────────────┬─────────────┘                        │
│                                  │                                       │
│                    ┌─────────────▼─────────────────────────┐            │
│                    │ HTTP Client (for send path)           │            │
│                    │ • mqtt_invoke() endpoint              │            │
│                    │ • Takes iot_id + base64 payload       │            │
│                    │ • Returns 200/401/50103/50104/20056   │            │
│                    └──────────────┬────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
         ┌──────────▼────────┐      │    ┌──────────▼────────┐
         │  Mammotion HTTP   │      │    │  MQTT Broker      │
         │  API Servers      │      │    │  (Mammotion Ops)  │
         │                   │      │    │                   │
         │  /mqtt_invoke     │      │    │  Port: 1883/8883  │
         │  /get_mqtt_creds  │      │    │  Protocol: MQTT 3 │
         │  /refresh_token   │      │    │                   │
         │  /login_v2        │      │    │  Topics:          │
         │                   │      │    │  • /sys/{pk}/{dn} │
         │  TLS: ✓           │      │    │    /thing/status  │
         └───────────────────┘      │    │  • /sys/{pk}/{dn} │
                                    │    │    /thing/event   │
                                    │    │  • /sys/{pk}/{dn} │
                                    │    │    /property/post │
                                    │    │                   │
                                    │    │  TLS: ✓ (8883)    │
                                    │    └───────────┬────────┘
                                    │                │
                        ┌───────────┴────────────────┘
                        │
                        │ Routes to Device
                        │ via backend systems
                        │
┌───────────────────────▼────────────────────────────────────────────────┐
│                    MAMMOTION LAWN MOWER DEVICE                         │
│                    (Luba, Yuka, RTK rover)                             │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Device Firmware                                                  │ │
│  │                                                                  │ │
│  │  Connected Modules:                                             │ │
│  │  • ESP32 (WiFi/BLE/4G comm)                                     │ │
│  │  • Main CPU (Spino OS)                                          │ │
│  │  • Navigation subsystem                                         │ │
│  │  • Cutter control                                               │ │
│  │  • Battery management                                           │ │
│  │  • Sensors (IMU, GPS, cameras)                                  │ │
│  │                                                                  │ │
│  │  Capabilities:                                                  │ │
│  │  • Subscribe to /sys/{pk}/{dn}/app/down/...                    │ │
│  │  • Process incoming LubaMsg protobuf commands                  │ │
│  │  • Execute work orders (mowing plans, navigation)              │ │
│  │  • Publish /sys/{pk}/{dn}/thing/status (state updates)         │ │
│  │  • Publish /sys/{pk}/{dn}/property/post (telemetry)            │ │
│  │  • Publish /sys/{pk}/{dn}/thing/event/... (notifications)      │ │
│  │                                                                  │ │
│  │  Built-in protobuf support:                                    │ │
│  │  • LubaMsg (wrapper)                                            │ │
│  │  • MctlSys (system commands)                                    │ │
│  │  • DevNet (network commands)                                    │ │
│  │  • report_info_t (device telemetry)                             │ │
│  │                                                                  │ │
│  │  Networking:                                                    │ │
│  │  • WiFi 802.11ac dual-band                                      │ │
│  │  • 4G LTE cellular (optional)                                   │ │
│  │  • GPS/RTK positioning                                          │ │
│  │  • MQTT connect via WiFi/4G to broker                           │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Message Flow Sequence Diagram

### Initial Connection & Authentication

```
┌──────────────┐                                        ┌──────────────┐
│ Human Client │                                        │ MQTT Broker  │
└──────┬───────┘                                        └──────┬───────┘
       │                                                        │
       │  1. POST /get_mqtt_credentials                        │
       │  (with HTTP Bearer token)                            │
       ├───────────────────────────────────────────────────────▶
       │  Mammotion HTTP API                                   │
       │                                                        │
       │  2. Returns MQTTCredentials:                          │
       │     {host, client_id, username, jwt, expires_at}     │
       │◀────────────────────────────────────────────────────┐ │
       │                                                       │  │
       │  3. Connect MQTT                                    │  │
       │     (client_id, username, password=jwt)             │  │
       ├──────────────────────────────────────────────────────▶ │
       │                                                        │
       │  4. MQTT CONNECT received                            │
       │     Validate JWT signature                           │
       │     Check exp claim                                  │
       │                                                        │
       │  5. MQTT CONNACK (success)                           │
       │◀────────────────────────────────────────────────────┐ │
       │                                                       │
       │  6. Subscribe to topics                              │
       │     /sys/{pk}/{dn}/app/down/thing/command            │
       │     /sys/{pk}/{dn}/app/down/thing/events             │
       │     (device publishes updates here)                  │
       ├──────────────────────────────────────────────────────▶ │
       │                                                        │
       │  7. SUBACK (subscribed)                              │
       │◀────────────────────────────────────────────────────┐ │
       │                                                       │
       ✓  Connected & ready to control
```

### Sending a Command (e.g., Start Mowing)

```
┌──────────────┐                      ┌──────────────┐                    ┌───────────┐
│ Human Client │                      │ HTTP Invoke  │                    │  Device   │
└──────┬───────┘                      │  Endpoint    │                    └─────┬─────┘
       │                              └──────┬───────┘                          │
       │                                      │                                 │
       │ 1. Build command (protobuf)         │                                 │
       │    LubaMsg with SEND_CONTROL        │                                 │
       │    (e.g., start mowing plan)        │                                 │
       │                                      │                                 │
       │ 2. send(payload, iot_id)            │                                 │
       │    • Base64 encode protobuf         │                                 │
       │    • POST to mqtt_invoke endpoint   │                                 │
       ├─────────────────────────────────────▶                                 │
       │                                      │                                 │
       │                                      │ 3. Server-side validation      │
       │                                      │    & routing to device         │
       │                                      │                                 │
       │                                      │ 4. Publish to device topic:   │
       │                                      │    /sys/{pk}/{dn}/app/down    │
       │                                      │    /thing/command              │
       │                                      │    (base64-wrapped protobuf)  │
       │                                      ├──────────────────────────────▶ │
       │                                      │                                 │
       │                                      │                                 │
       │ 5. HTTP 200 response                │                                 │
       │    (command queued/delivered)       │                                 │
       │◀─────────────────────────────────────┤                                 │
       │                                      │    5. Device receives msg     │
       │                                      │       Parses protobuf         │
       │                                      │       Executes command        │
       │                                      │       (starts mowing)         │
       │                                      │                                 │
       │                                      │    6. Publish status update   │
       │                                      │       to /sys/{pk}/{dn}       │
       │                                      │       /thing/status           │
       │                                      │       (JSON with device state)│
       │    7. Receive status message        │◀─────────────────────────────┤
       │       on subscribed topic           │                                 │
       │    (device now mowing)              │                                 │
       │◀─────────────────────────────────────┤                                 │
       │       via MQTT broker               │                                 │
       │                                      │                                 │
       ✓  Command executed
```

### Receiving Telemetry & Events

```
┌──────────────┐                              ┌──────────────┐
│ Human Client │                              │ MQTT Broker  │
└──────┬───────┘                              └──────┬───────┘
       │                                             │
       │  Subscribed to:                            │
       │  • /sys/{pk}/{dn}/thing/status             │
       │  • /sys/{pk}/{dn}/property/post            │
       │  • /sys/{pk}/{dn}/thing/event/+            │
       │                                             │
       │                            ┌─ Device publishes periodically
       │                            │
       │  1. PUBLISH received (status update)       │
       │     Topic: /sys/{pk}/{dn}/thing/status     │
       │     Payload:                               │
       │     {                                       │
       │       "action": "online",                  │
       │       "iotId": "device-123",               │
       │       "productKey": "pk-123",              │
       │       "timestamp": 1234567890              │
       │     }                                      │
       │◀─────────────────────────────────────────┐ │
       │                                           │  │
       │  2. on_device_status callback fired      │  │
       │     Update internal device state         │  │
       │                                           │  │
       │  3. PUBLISH received (telemetry)         │  │
       │     Topic: /sys/{pk}/{dn}/property/post  │  │
       │     Payload:                              │  │
       │     {                                      │  │
       │       "params": {                          │  │
       │         "content": "<base64-protobuf>"    │  │
       │       }                                    │  │
       │     }                                      │  │
       │◀─────────────────────────────────────────┐ │
       │                                           │
       │  4. on_device_mammotion_properties fired │
       │     Parse & extract:                      │
       │     • Battery: 85%                        │
       │     • Position: (123.45, 45.67)           │
       │     • Status: MOWING                      │
       │     • Errors: none                        │
       │                                           │
       │  5. PUBLISH received (event)              │
       │     Topic: /sys/{pk}/{dn}/thing/event    │
       │            /device_notification/post      │
       │     Payload:                               │
       │     {                                      │
       │       "params": {                          │
       │         "content": "obstacle_detected"    │
       │       }                                    │
       │     }                                      │
       │◀─────────────────────────────────────────┐ │
       │                                           │
       │  6. on_device_notification callback fired │
       │     Alert: "Obstacle detected!"           │
       │                                           │
       ✓  Telemetry flowing in
```

---

## Data Structure Transformations

### Command Sending Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. PYTHON OBJECT                                                    │
├─────────────────────────────────────────────────────────────────────┤
│ MammotionCommand.start_mowing(plan_id=5)                            │
│  └─ Returns bytes (LubaMsg protobuf bytes)                          │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. PROTOBUF BINARY                                                  │
├─────────────────────────────────────────────────────────────────────┤
│ LubaMsg {                                                            │
│   msgtype: MsgCmdType.ESP = 1                                       │
│   sender: MsgDevice.DEV_MOBILEAPP = 0x10                           │
│   rcver: MsgDevice.DEV_COMM_ESP = 0x20                             │
│   msgattr: MsgAttr.REQ = 1                                          │
│   seqs: 42                                                          │
│   version: 1                                                        │
│   timestamp: 1701234567890                                          │
│   sys: MctlSys {                                                    │
│     app_to_dev_set_plan_job: AppToDevPlanJobSet {                 │
│       plan_id: 5                                                    │
│       plan_action: DO                                               │
│     }                                                               │
│   }                                                                 │
│ }                                                                   │
│                                                                     │
│ Serialized to: b'\x08\x01\x10\x10...' (binary blob)               │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. BASE64 ENCODING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│ base64.b64encode(protobuf_bytes)                                   │
│ = "CAgQEGobCgUqBA0FAAAA..." (ASCII-safe string)                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. JSON ENVELOPE                                                    │
├─────────────────────────────────────────────────────────────────────┤
│ {                                                                   │
│   "params": {                                                       │
│     "content": "CAgQEGobCgUqBA0FAAAA..."                          │
│   }                                                                 │
│ }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. HTTP POST BODY                                                   │
├─────────────────────────────────────────────────────────────────────┤
│ POST /mqtt_invoke/{iot_id}                                          │
│ Content-Type: application/json                                      │
│ Authorization: Bearer {access_token}                                │
│                                                                     │
│ Body: JSON from step 4                                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. SERVER-SIDE PROCESSING                                           │
├─────────────────────────────────────────────────────────────────────┤
│ • Validate iot_id belongs to user's devices                         │
│ • Check command quota (24-hour limit)                               │
│ • Route payload to device backend                                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. MQTT PUBLISH                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ Topic: /sys/{product_key}/{device_name}/app/down/thing/command     │
│ Payload: JSON with base64 protobuf (same as step 4)               │
│ QoS: 1 (at least once)                                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 8. DEVICE RECEIVES & PARSES                                         │
├─────────────────────────────────────────────────────────────────────┤
│ • MQTT message arrives on subscribed topic                          │
│ • Extract base64 from JSON params.content                           │
│ • Decode base64 to binary                                           │
│ • Deserialize protobuf to LubaMsg                                   │
│ • Extract command (MctlSys.app_to_dev_set_plan_job)                │
│ • Execute: start mowing plan 5                                      │
│ • Motor control, navigation, etc.                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Credential Refresh Lifecycle

```
Client Startup
│
├─ HTTP Login (OAuth)
│  ├─ POST /login_v2 (email + password)
│  ├─ Returns: access_token, refresh_token, expires_in
│  └─ Store in HTTPCredentials
│
├─ Get MQTT Credentials
│  ├─ POST /get_mqtt_credentials (with Bearer token)
│  ├─ Returns: host, client_id, username, jwt, expires_at
│  └─ Store in MQTTCredentials
│
└─ Connect MQTT
   └─ Use jwt as password
      └─ Expires in ~24h
         │
         ├─ TokenManager proactive refresh (30-min window)
         │  ├─ Check JWT exp claim
         │  ├─ If expires in <30min: POST /get_mqtt_credentials
         │  └─ Update cached jwt
         │
         └─ On JWT rejection (MQTT rc=4 or 5)
            ├─ Force refresh (1 attempt)
            │  ├─ POST /refresh_token_v2 (refresh_token only)
            │  ├─ POST /get_mqtt_credentials (new Bearer)
            │  └─ Reconnect MQTT
            │
            └─ Still rejected?
               └─ Give up (require full re-login)
                  └─ Trigger on_fatal_auth_error callback
                     └─ Client must call login_v2() again
```

---

## Topic Hierarchy & Message Types

```
/sys/
├── {product_key}/
│   └── {device_name}/
│       │
│       ├── thing/
│       │   ├── status
│       │   │   ├─ Direction: Device → Client
│       │   │   ├─ Format: JSON
│       │   │   ├─ Content: {"action": "online", "iotId": "...", ...}
│       │   │   └─ Frequency: On connection/disconnection
│       │   │
│       │   ├── properties
│       │   │   ├─ Direction: Device → Client
│       │   │   ├─ Format: JSON with nested protobuf
│       │   │   ├─ Content: {"params": {"content": "<base64>", "iotId": "..."}}
│       │   │   └─ Frequency: Periodic updates
│       │   │
│       │   └── event/
│       │       ├── device_protobuf_msg_event/post
│       │       │   ├─ Direction: Device → Client
│       │       │   ├─ Format: JSON with base64 protobuf
│       │       │   ├─ Content: {"params": {"content": "<base64-LubaMsg>"}}
│       │       │   └─ Frequency: As needed (unsolicited messages)
│       │       │
│       │       ├── device_notification_event/post
│       │       │   ├─ Direction: Device → Client
│       │       │   ├─ Format: JSON
│       │       │   ├─ Content: {"params": {"content": "event_name"}}
│       │       │   └─ Examples: "obstacle_detected", "battery_low", etc.
│       │       │
│       │       └── [other event types]/post
│       │           └─ Generic notification handling
│       │
│       ├── property/post
│       │   ├─ Direction: Device → Client
│       │   ├─ Format: JSON with base64 protobuf
│       │   ├─ Content: Flat properties (alt format to thing/properties)
│       │   └─ Used in Mammotion direct-MQTT systems
│       │
│       └── app/down/  (inbound only, not visible to MQTT clients)
│           ├── thing/command
│           │   ├─ Direction: Broker → Device (Server-side routing)
│           │   ├─ Format: JSON with base64 protobuf
│           │   └─ Content: Commands from client via HTTP invoke
│           │
│           └── thing/events
│               └─ Alternative event delivery path
```

---

## Error Handling Flow

```
┌─────────────────────────────────────┐
│ send(payload, iot_id)                │
└────────────────┬────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ Rate Limited?      │
        └────────────┬───────┘
                     │
              ┌──────┴──────┐
              │ YES         │ NO
              │             │
              ▼             ▼
    TransportRateLimitedError
                       │
                       ▼
            ┌────────────────────┐
            │ POST /mqtt_invoke   │
            │ (base64 payload)    │
            └────────────┬────────┘
                         │
                    ┌────┴─────────────────────┬──────────────┐
                    │                          │              │
                    ▼                          ▼              ▼
            ┌──────────────┐          ┌──────────────┐  ┌──────────────┐
            │ HTTP 200/0   │          │ HTTP 401     │  │ HTTP 50103   │
            │ (Success)    │          │ (Auth fail)  │  │ (Offline)    │
            └──────────────┘          └──────┬───────┘  └──────────────┘
                   │                         │                │
                   ▼                         ▼                ▼
            Command sent           Refresh access token    Device offline
                             │               │           Exception
                             │               │
                             ▼               ▼
                    ┌──────────────────┐
                    │ Retry Invoke     │
                    │ (with new token) │
                    └────────┬─────────┘
                             │
                        ┌────┴─────┐
                        │ SUCCESS   │ FAIL
                        ▼           ▼
                    Success    401 after refresh?
                                    │
                                    ▼
                        ReLoginRequiredError
                                    │
                                    ▼
                        Give up (trigger re-login)
```

---

## Summary Table: Protocol Layers

| Layer | Technology | Encoding | Security |
|-------|-----------|----------|----------|
| **Application** | PyMammotion | Python objects (dataclasses) | N/A |
| **Command** | Protocol Buffers (protobuf) | Binary serialization | Field validation |
| **Encoding** | Base64 | ASCII-safe string | Data transport |
| **Message** | JSON | UTF-8 string | Structure validation |
| **Transport** | MQTT 3.1.1 | Binary over TCP | TLS 1.2+ (port 8883) |
| **Authentication** | JWT + OAuth 2.0 | Token-based | HMAC-SHA256 signature |
| **Network** | TCP/IP | Binary frames | Optional TLS/SSL |

---

## Key Design Decisions

1. **Base64 Encoding**: Protobuf is binary; needs to be ASCII-safe for JSON transport
2. **JSON Wrapper**: MQTT payloads must be machine-readable by cloud infrastructure
3. **HTTP Invoke Endpoint**: Server-side routing provides better control than direct MQTT pub
4. **JWT Authentication**: Stateless, scalable, no session storage needed
5. **Proactive Refresh**: Avoids auth failures in the middle of operations
6. **Topic Hierarchy**: Organizes messages by device and type (status/properties/events)
7. **Dual Format Support**: Both Aliyun and Mammotion direct formats for compatibility
