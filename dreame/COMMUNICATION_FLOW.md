# Dreame Mower Communication Flow - Simplified Schematic

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                          USER / HOME ASSISTANT                             │
│                                                                             │
└────────────┬────────────────────────────────────────────────────────────────┘
             │
             │ REST API Call
             │ (Control: Start/Stop/Dock)
             │
             ▼
  ┌────────────────────────────────┐
  │   DREAME CLOUD SERVER          │
  │   ◆ Authentication (REST)      │
  │   ◆ Command Routing            │
  │   ◆ API Strings Decoding       │
  │                                │
  │  Endpoints:                    │
  │  - /auth/authenticate          │
  │  - /rpc/action                 │
  │  - /device/getDeviceData       │
  └────────────┬────────────────────┘
               │
               │ MQTT Broker Connection
               │ (Port 8883 - TLS/SSL)
               │
  ┌────────────▼────────────┐
  │   MQTT MESSAGE BROKER   │
  │                         │
  │   Topic:                │
  │   /{svc_id}/{device_id}/│
  │   {uid}/{model}/{country}│
  │                         │
  └────┬────────────────┬───┘
       │                │
  MQTT │ PUBLISH        │ SUBSCRIBE
  PUSH │ (Real-time     │ (Commands)
  FROM │  updates)      │
  MOWER│                │
       │                │
  ┌────▼────────────────▼────┐
  │   HOME ASSISTANT          │
  │   (MQTT Client)           │
  │                           │
  │ ◆ Receives updates        │
  │ ◆ Sends commands          │
  │ ◆ Updates UI              │
  └────────────────────────────┘
       │                │
       │                └─────────────────────┐
       │                                      │
       │ Local Bluetooth                      │
       │ Status Monitoring                    │
       │                                      │
       ▼                                      ▼
  ┌──────────────────────────────────────────────┐
  │        DREAME LAWN MOWER DEVICE              │
  │                                              │
  │ ◆ Bluetooth LE Radio                         │
  │   └─ Connection status only                  │
  │                                              │
  │ ◆ WiFi + Cloud Connection                    │
  │   └─ MQTT publish (status updates)           │
  │   └─ Receives commands via cloud             │
  │                                              │
  │ ◆ Motor/Blade Control                        │
  │ ◆ Navigation & Mapping                       │
  │ ◆ Battery Management                         │
  └──────────────────────────────────────────────┘
```

---

## Detailed Message Flow - Single Command Sequence

```
┌──────────────┐                                        ┌─────────────┐
│     USER     │                                        │   MOWER     │
│   (Mobile    │                                        │   DEVICE    │
│    App or    │                                        │             │
│    HA UI)    │                                        └─────────────┘
└──────┬───────┘                                              ▲
       │                                                      │
       │ 1. TAP "START MOWING"                               │
       │                                                      │
       ▼                                                      │
┌──────────────────────────────┐                             │
│  HOME ASSISTANT              │                             │
│  ◆ Prepares REST command     │                             │
│  ◆ Adds authentication token │                             │
└──────┬───────────────────────┘                             │
       │                                                      │
       │ 2. REST API POST /rpc/action                        │
       │    {                                                │
       │      "did": "device_123",                          │
       │      "method": "action",                           │
       │      "params": {                                   │
       │        "siid": 2,                                  │
       │        "aiid": 1  ← Start Mowing Action           │
       │      }                                             │
       │    }                                               │
       │                                                      │
       ▼                                                      │
┌───────────────────────────────────────────────┐          │
│  DREAME CLOUD SERVER                          │          │
│  ◆ Validates request                          │          │
│  ◆ Checks device credentials                  │          │
│  ◆ Queues command for device                  │          │
└──────┬────────────────────────────────────────┘          │
       │                                                      │
       │ 3. MQTT PUBLISH on device topic                    │
       │    (Device is subscribed)                          │
       │    Payload:                                        │
       │    {                                               │
       │      "method": "action",                           │
       │      "params": {...}                              │
       │    }                                               │
       │                                                      │
       ▼                                                      │
┌─────────────────────┐                                     │
│  MQTT Broker        │                                     │
│  /dreame/device_123/│                                     │
│   uid/model/cn/     │                                     │
└────┬────────────────┘                                     │
     │                                                       │
     │ 4. MQTT DELIVER MESSAGE                              │
     │    (to subscribed clients)                            │
     │                                                       │
     └──────────────────────────────────────────────────────►
                                                            │
                                          5. EXECUTE COMMAND│
                                          ◆ Start motor      │
                                          ◆ Deploy mowing head│
                                          ◆ Begin navigation │
                                          │
     ┌─────────────────────────────────────┘
     │
     │ 6. MQTT PUBLISH STATUS UPDATES
     │    (every 1-5 seconds)
     │    {
     │      "method": "properties_changed",
     │      "params": [
     │        {"siid": 2, "piid": 1, "value": 1},  ← Mowing
     │        {"siid": 2, "piid": 57, "value": 92} ← Battery 92%
     │      ]
     │    }
     │
     ▼
┌──────────────────────────────┐
│  MQTT Broker                 │
│  (receives live updates)     │
└────┬─────────────────────────┘
     │
     │ 7. MQTT DELIVER UPDATES
     │    (to HA subscriber)
     │
     ▼
┌──────────────────────────────────┐
│  HOME ASSISTANT                  │
│  ◆ Receives status updates       │
│  ◆ Updates device entity state   │
│  ◆ Triggers automations          │
│  ◆ Refreshes UI display          │
└──────────────────────────────────┘
     │
     │ 8. DISPLAY TO USER
     │    "Mower Status: MOWING ✓"
     │    "Battery: 92%"
     │
     ▼
┌──────────────┐
│     USER     │
│   Sees live  │
│   updates    │
└──────────────┘
```

---

## Connection States Diagram

```
                    ┌─────────────────────┐
                    │  NOT CONNECTED      │
                    │  (Offline)          │
                    └────────┬────────────┘
                             │
                    REST Auth │ login()
                             │
                             ▼
                    ┌─────────────────────┐
                    │ REST CONNECTED      │
                    │ (Token obtained)    │
                    └────────┬────────────┘
                             │
                    MQTT Init │
                             │
                             ▼
                    ┌─────────────────────┐
                    │ MQTT CONNECTING     │
                    │ (handshake)         │
                    └���───────┬────────────┘
                             │
                    Connected│
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │        FULLY CONNECTED                             │
    │  ◆ Receiving real-time updates via MQTT           │
    │  ◆ Can send commands via REST                     │
    │  ◆ Device is reachable                            │
    └────────────────────────────────────────────────────┘
                             │
                    Device │ offline OR
                    Network │ error
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │        DISCONNECTED                                │
    │  ◆ MQTT connection lost                           │
    │  ◆ Attempting automatic reconnect                 │
    │  ◆ Backoff: 1s → 15s delay                       │
    └────────────────────────────────────────────────────┘
```

---

## Network Topology

```
                     INTERNET
                        │
          ┌─────────────┴─────────────┐
          │                           │
    ┌─────▼──────────┐        ┌──────▼─────────┐
    │  Dreame Cloud  │        │   MQTT Broker  │
    │  REST Server   │        │   (AWS/Aliyun) │
    │  (China/Global)│        │   Port 8883    │
    └────────────────┘        └────────────────┘
          ▲ │                         ▲ │
          │ │                         │ │
    REST  │ │ MQTT                    │ │ MQTT
    HTTPS │ │ Publish                 │ │ Subscribe
          │ └─────────────┬───────────┘ │
          │               │             │
          │        ┌──────▼─────────┐   │
          │        │ Home Network   │   │
          │        │ (WiFi Router)  │   │
          │        └──────┬─────────┘   │
          │               │             │
    ┌─────┴───────────────┴─────────────┴──────────┐
    │    Home Assistant (Automation Hub)           │
    │    - REST Client (sends commands)            │
    │    - MQTT Client (receives updates)          │
    │    - Local UI (Lovelace Dashboard)           │
    └──────────────────┬──────────────────────────┘
                       │
                  Local │ Bluetooth LE
                  Network│ Monitoring
                       │
                       ▼
            ┌────────────────────────┐
            │  Dreame Mower Device   │
            │  - MQTT Client         │
            │  - WiFi/4G Modem       │
            │  - Bluetooth LE Radio  │
            │  - Motor & Navigation  │
            └────────────────────────┘
```

---

## Data Types & Protocols

### REST API Communication
```
Protocol:     HTTPS (TLS 1.2)
Method:       POST
Port:         9999
Content-Type: application/x-www-form-urlencoded
Auth:         Token-based (Bearer token)
Timeout:      20 seconds
Retry:        Up to 3 attempts
```

### MQTT Communication
```
Protocol:     MQTT (Message Queuing Telemetry Transport)
Encryption:   TLS 1.2 with self-signed cert
Port:         8883 (standard MQTT + TLS)
QoS:          Level 1 (At least once delivery)
Keepalive:    50 seconds
Clean Session: True
Reconnect:    Exponential backoff (1s to 15s)
```

---

## Property Update Format (MQTT Payload)

```
┌─────────────────────────────────────────────────────────┐
│ MQTT Message Structure                                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  {                                                      │
│    "method": "properties_changed",                     │
│    "params": [                                         │
│      {                                                  │
│        "siid": 2,        ← Service ID (main system)   │
│        "piid": 1,        ← Property ID (status)       │
│        "value": 1        ← Value (1 = mowing)         │
│      },                                                │
│      {                                                  │
│        "siid": 2,        ← Service ID                 │
│        "piid": 57,       ← Property ID (battery)      │
│        "value": 85       ← Value (85% battery)        │
│      },                                                │
│      {                                                  │
│        "siid": 1,        ← Service ID (device info)   │
│        "piid": 53,       ← Property ID (bluetooth)    │
│        "value": true     ← Value (connected)          │
│      }                                                  │
│    ]                                                   │
│  }                                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘

Key Properties:
  siid=1, piid=53  → Bluetooth Connected (bool)
  siid=2, piid=1   → Status (int) 1=Mowing, 2=Charging, etc.
  siid=2, piid=57  → Battery Level (int 0-100)
  siid=2, piid=2   → Device Code (error/warning)
```

---

## Timeline: User Initiates "Start Mowing"

```
TIME    ACTOR              ACTION
────────────────────────────────────────────────────────────────
0ms     User              Taps "Start Mowing" button in app
│
├─ 10ms  HA Coordinator    Reads credentials from config entry
├─ 20ms  HA Entity         Sends REST POST /rpc/action
│
├─ 50ms  Cloud Server      ✓ Receives request
├─ 100ms Cloud Server      Validates auth token
├─ 150ms Cloud Server      Looks up device in registry
├─ 200ms Cloud Server      Queues command
│
├─ 250ms MQTT Broker       ✓ Message published on device topic
├─ 300ms MQTT Broker       Delivers to subscribers
│
├─ 350ms Mower Device      ✓ Receives MQTT message
├─ 400ms Mower Device      Validates command
├─ 450ms Mower Device      Executes: Start motor & navigation
│
├─ 500ms Mower Device      Publishes status: "Mowing"
│
├─ 550ms MQTT Broker       ✓ Status message published
├─ 600ms MQTT Broker       Delivers to HA subscriber
│
├─ 650ms HA Coordinator    ✓ Receives status update
├─ 700ms HA Entity         Updates: status="mowing"
├─ 750ms HA UI             Refreshes display
│
└─ 800ms User              ✓ Sees "Mower: MOWING" on screen

TOTAL LATENCY: ~800ms (typical)
```

---

## Summary Table

| Component | Role | Protocol | Details |
|-----------|------|----------|----------|
| **User/HA** | Controller | REST + MQTT | Sends commands, receives updates |
| **Cloud Server** | Gateway | REST API | Validates auth, routes commands |
| **MQTT Broker** | Message Hub | MQTT TLS | Pub/sub real-time messaging |
| **Mower Device** | Executor | MQTT + WiFi | Executes commands, sends telemetry |
| **Bluetooth** | Monitor | BLE | Local connection status only |

---

## Quick Reference: Message Types

### FROM USER → MOWER (Commands)
```
REST POST /rpc/action
└─ Start Mowing
└─ Pause / Resume
└─ Stop & Dock
└─ Set Schedule
└─ Configure Zones/Areas
```

### FROM MOWER → USER (Status Updates)
```
MQTT PUBLISH (real-time)
└─ Status: Mowing / Charging / Idle / Error
└─ Battery Level: 0-100%
└─ Position: X, Y coordinates
└─ Coverage: % area completed
└─ Error Codes: LiDAR blocked, wheel stuck, etc.
└─ Bluetooth: Connected / Disconnected
```

### KEEPALIVE
```
MQTT: Heartbeat every 50 seconds
REST: Token refresh before expiry (24-48 hours)
```

---

## Key Insights

✅ **Commands** flow through REST → Cloud → MQTT → Device  
✅ **Status updates** return via MQTT → Cloud → HA → UI  
✅ **Bluetooth** only monitors local connection, NOT used for control  
✅ **Real-time latency** is typically 800ms end-to-end  
✅ **Automatic reconnection** with exponential backoff (1-15 seconds)  
✅ **All control is cloud-mediated** - no local-only command possible
