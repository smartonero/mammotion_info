# Dreame Lawn Mower Integration Documentation

## Overview

This folder contains comprehensive technical documentation about Dreame and MOVA lawn mower communication protocols, authentication, and integration with Home Assistant and other automation platforms.

## Documentation Files

### 1. **COMMUNICATION_FLOW.md**
Comprehensive guide to the complete communication architecture:
- High-level system architecture diagrams
- Detailed message flow sequences
- Connection state diagrams
- Network topology
- Protocol specifications (REST API and MQTT)
- Timeline and latency analysis
- Message type references

**Use when**: Understanding how data flows between user, cloud, and device

### 2. **BLUETOOTH_PROTOCOL.md**
Detailed specification of Bluetooth LE functionality:
- Bluetooth connection status monitoring
- Property identifiers (siid/piid)
- MQTT payload structure
- Home Assistant sensor integration
- Why Bluetooth is NOT used for commands
- Troubleshooting Bluetooth issues

**Use when**: Implementing Bluetooth monitoring or troubleshooting connectivity

### 3. **CLOUD_API_AUTHENTICATION.md**
Complete authentication and API gateway documentation:
- Authentication sequence flow
- API string obfuscation (Base64 + zlib)
- Token lifecycle and refresh
- Account types (Dreame vs MOVA)
- Error handling and recovery
- Security considerations
- Retry strategies

**Use when**: Implementing cloud authentication or API integration

## Quick Reference

### Communication Layers

```
User/App
   ↓ REST API (commands)
Dreame Cloud Server
   ↓ MQTT TLS (command relay)
MQTT Broker
   ↓ MQTT (subscription)
Home Assistant / Device
   ↑ MQTT (telemetry)
MQTT Broker
   ↑ MQTT (relay)
Dreame Cloud Server
   ↑ REST API (status)
User/App
```

### Key Facts

✓ **All control commands** go through cloud REST API → MQTT → Device  
✓ **All status updates** come via MQTT from device → HA → User  
✓ **Bluetooth** monitors local connection status only (NOT for control)  
✓ **Real-time latency** is typically 800ms end-to-end  
✓ **MQTT runs on port 8883** with TLS 1.2 encryption  
✓ **REST API port is 9999** with token-based authentication  

### Protocols Used

| Protocol | Use | Port | Encryption | Details |
|----------|-----|------|------------|----------|
| HTTPS | Authentication & Control | 9999 | TLS 1.2 | REST API |
| MQTT | Real-time Updates | 8883 | TLS 1.2 | Pub/Sub messaging |
| BLE | Status Monitor | N/A | BLE 5.0 | Local connection only |

### Device Properties (siid/piid)

```
Service 1: Device Information
  piid=1   - Telemetry data (20-byte array)
  piid=53  - Bluetooth connected status (bool)
  piid=50-55 - Firmware & session info

Service 2: Mowing Control
  piid=1   - Status code (int: 1=mowing, 2=charging, ...)
  piid=2   - Device code (errors/warnings)
  piid=50  - Scheduling/task payload
  piid=57  - Power/battery state

Services 3-6: Additional subsystems
  - Map data
  - Navigation
  - Consumables tracking
  - OTA updates
```

## Account Types

### Dreame Account
- App: **Dreamehome**
- Devices: Dreame G10, D9, A1, A2, etc.
- Endpoint: `us/eu/cn.dreame-cloud.net:9999`

### MOVA Account
- App: **MOVAhome**
- Devices: MOVA Z500, Y1000, A1 Pro, etc.
- Endpoint: `us/eu/cn.dreame-cloud.net:9999` (same)
- API Strings: Different encoding

## Country Codes

```
us → United States
eu → Europe  
cn → China
... (other regional endpoints available)
```

## Message Examples

### Start Mowing Command (REST)
```json
POST https://us.dreame-cloud.net:9999/rpc/action
Authorization: Bearer {token}

{
  "did": "device_123456",
  "id": "1001",
  "method": "action",
  "params": {
    "did": "device_123456",
    "siid": 2,
    "aiid": 1,
    "in": []
  }
}
```

### Status Update (MQTT)
```json
Topic: /dreame/device_123456/uid/model/us/
QoS: 1

{
  "method": "properties_changed",
  "params": [
    {"siid": 2, "piid": 1, "value": 1},
    {"siid": 2, "piid": 57, "value": 85},
    {"siid": 1, "piid": 53, "value": true}
  ]
}
```

## Integration Points

### Home Assistant
- MQTT client subscribes to device topic
- REST client sends commands to cloud
- Coordinator handles device state
- Entities: sensor, button, select, lawn_mower

### Mobile Apps
- Dreamehome app (Dreame devices)
- MOVAhome app (MOVA devices)
- Also use cloud infrastructure for commands

### Third-party Integrations
- Must go through cloud REST API
- Cannot directly access MQTT or Bluetooth
- Requires valid cloud credentials

## Troubleshooting

### Device Offline
1. Check device WiFi connection
2. Check internet connectivity
3. Verify Dreame cloud server status
4. Check firewall (port 8883 for MQTT, 9999 for REST)
5. Restart device and Home Assistant

### Commands Not Working
1. Verify cloud connection (REST API test)
2. Check MQTT connection status
3. Verify device is not in error state
4. Check Home Assistant logs for API errors
5. Verify account credentials are correct

### Bluetooth Disconnected
1. Move mobile device closer to mower
2. Check for WiFi interference
3. This does NOT affect cloud control - still works normally
4. Bluetooth is optional for control

## Related Repositories

- **[antondaubert/dreame-mower](https://github.com/antondaubert/dreame-mower)** - Most active Home Assistant integration ⭐ 81
- **[bhuebschen/dreame-mower](https://github.com/bhuebschen/dreame-mower)** - Original integration ⭐ 63
- **[nicolasglg/dreame-mova-mower](https://github.com/nicolasglg/dreame-mova-mower)** - MOVA-specific integration
- **[EvotecIT/homeassistant-dreamelawnmower](https://github.com/EvotecIT/homeassistant-dreamelawnmower)** - HACS integration

## API Discovery

The API strings are obfuscated in source code and decoded at runtime:

```python
# Encoded
DREAME_STRINGS = "H4sICAAAAAAEAGNsb3VkX3N0cmluZ3MuanNvbgBd..."

# Decoded
import base64, zlib, json
strings = json.loads(
    zlib.decompress(
        base64.b64decode(DREAME_STRINGS),
        zlib.MAX_WBITS | 32
    )
)
# Result: ~60 API endpoints and configuration strings
```

## Security Notes

⚠️ **Never share**:
- Cloud credentials
- API tokens
- Device IDs
- Device MAC addresses
- Cloud URLs

✓ **Always**:
- Use HTTPS for API calls
- Store credentials securely
- Use token-based auth
- Verify SSL certificates (or explicitly ignore for self-signed)
- Keep firmware updated

## Performance Metrics

| Operation | Typical Latency | Max Latency |
|-----------|-----------------|-------------|
| REST API call | 50-100ms | 500ms |
| MQTT message | 100-200ms | 1000ms |
| Full command cycle | 800ms | 2000ms |
| Status update | 1-5 seconds | 30 seconds |
| Device connection | 2-5 seconds | 30 seconds |

## Version History

- **2024-2026**: Cloud-based integration (current)
- **2023-2024**: Local-first integration (deprecated)
- Earlier: Direct BLE control (no longer supported)

## Contributing

If you have improvements or corrections to this documentation:
1. Test your findings
2. Document the change
3. Create a pull request with details
4. Include device model and firmware version

## License

This documentation is provided as reference material for understanding Dreame mower integrations.
