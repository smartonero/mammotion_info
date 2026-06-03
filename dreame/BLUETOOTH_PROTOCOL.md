# Dreame Mower Bluetooth Protocol Specification

## Overview

Bluetooth Low Energy (BLE) connection in Dreame mowers is primarily used for **local status monitoring** rather than command control. All actual control commands are routed through the cloud infrastructure.

## Bluetooth Connection Details

### Property Identifier
```python
BLUETOOTH_PROPERTY = PropertyIdentifier(
    siid=1,      # Service 1 (Device Information)
    piid=53,     # Property 53 (Bluetooth Connection Status)
    name="bluetooth_connected"
)
```

### Data Type
- **Type**: Boolean
- **Values**: 
  - `true` = Bluetooth connected to mobile app
  - `false` = Bluetooth disconnected

### MQTT Message Example
```json
{
  "method": "properties_changed",
  "params": [
    {
      "siid": 1,
      "piid": 53,
      "value": true
    }
  ]
}
```

## What Bluetooth Connection Status Means

✓ **When true**: Device is currently connected to a mobile app via Bluetooth LE  
✓ **When false**: Device is not connected via Bluetooth (typically when out of range)  
✗ **NOT used for**: Actual mower command execution  
✓ **Used for**: Diagnostics, monitoring, and local status awareness

## Cloud-Based Command Flow (NOT Bluetooth)

All mower control commands bypass local Bluetooth and route through the cloud:

```
User Action → REST API → Dreame Cloud → MQTT Broker → Device
```

### Why Not Direct Bluetooth?
1. **Cloud authentication required** - Commands must be validated through cloud
2. **Distributed control** - Multiple users/apps can control same device
3. **Reliability** - Cloud relay ensures command delivery (retry logic)
4. **Scalability** - Centralized command processing
5. **Security** - All commands pass through authenticated cloud gateway

## Bluetooth Property Updates

Bluetooth connection status is reported via MQTT as part of device telemetry:

```python
def _handle_mqtt_property_update(self, message: dict[str, Any]) -> bool:
    """Handle MQTT property updates including Bluetooth status."""
    try:
        siid = message["siid"]
        piid = message["piid"]
        
        if BLUETOOTH_PROPERTY.matches(siid, piid):
            bluetooth_value = bool(message["value"])
            old_bluetooth = self._bluetooth_connected
            self._bluetooth_connected = bluetooth_value
            
            if old_bluetooth != bluetooth_value:
                # Notify Home Assistant of Bluetooth status change
                self._notify_property_change(
                    BLUETOOTH_PROPERTY.name, 
                    bluetooth_value
                )
                return True
    except Exception as ex:
        _LOGGER.error("Failed to handle MQTT property update: %s", ex)
        return False
```

## Bluetooth Sensor in Home Assistant

```python
class DreameMowerBluetoothSensor(DreameMowerEntity, SensorEntity):
    """Bluetooth connection sensor for Dreame Mower."""

    def __init__(self, coordinator: DreameMowerCoordinator) -> None:
        super().__init__(coordinator, "bluetooth_connection")
        self._attr_icon = "mdi:bluetooth"
        self._attr_entity_category = EntityCategory.DIAGNOSTIC
        self._attr_translation_key = "bluetooth_connection"

    @property
    def native_value(self) -> bool | None:
        """Return the Bluetooth connection status."""
        return self.coordinator.device_bluetooth_connected

    @property
    def extra_state_attributes(self) -> dict[str, Any]:
        """Return extra state attributes."""
        return {
            "bluetooth_connected": self.coordinator.device_bluetooth_connected,
        }
```

## Monitoring Bluetooth Status

To check Bluetooth connection status in Home Assistant:

```yaml
# In Home Assistant templates
{{ state_attr('sensor.dreame_mower_bluetooth_connection', 'bluetooth_connected') }}

# In automations
automation:
  - alias: "Notify when Bluetooth disconnects"
    trigger:
      platform: state
      entity_id: sensor.dreame_mower_bluetooth_connection
      to: "off"
    action:
      service: persistent_notification.create
      data:
        title: "Mower Bluetooth Disconnected"
        message: "Dreame mower lost Bluetooth connection"
```

## Important Notes

⚠️ **Bluetooth Status is NOT Required**
- Device functions normally even if Bluetooth is disconnected
- Bluetooth only provides local status awareness
- All commands still work via cloud/MQTT regardless of Bluetooth state

⚠️ **Manual Mode Limitation**
- Manual mowing mode may require active Bluetooth connection
- This is the only exception where Bluetooth is directly involved in control
- See device logs for detailed error: "Manual mowing is not implemented yet; it appears to require Bluetooth"

⚠️ **Local-Only Control Not Supported**
- Even with Bluetooth connected, direct local commands are NOT supported
- All commands MUST go through cloud/MQTT infrastructure
- This is by design for security and authentication reasons

## Connectivity Chain

```
Bluetooth Status    MQTT Update     Home Assistant    User Sees
────────────────────────────────────────────────────────────────
Device BLE → Cloud → MQTT Broker → HA Integration → Dashboard
Connected                                              ✓ Connected
Disconnected                                          ✓ Disconnected
```

## Protocol Versions

| Version | Type | Details |
|---------|------|----------|
| BLE 5.0+ | Bluetooth LE | Standard for modern Dreame devices |
| MQTT 3.1.1 | MQTT | Status updates pushed via MQTT |
| REST API | HTTP/HTTPS | Command execution channel |

## Troubleshooting Bluetooth Disconnect

### Common Causes
1. Mobile app interfering with cloud MQTT connection
2. BLE radio interference (WiFi, other devices)
3. Device out of Bluetooth range (typically >50 meters)
4. Device in "away" mode (Bluetooth disabled for power saving)

### Solution
1. Ensure device is powered and connected to WiFi
2. Check cloud connection status separately from Bluetooth
3. Restart device if Bluetooth status doesn't update
4. Bluetooth disconnection does NOT prevent cloud control

## Related Properties

```python
# Other device information properties (Service 1)
siid=1, piid=1   → Property 1:1 (Telemetry data)
siid=1, piid=2   → Firmware install state
siid=1, piid=3   → Firmware download progress
siid=1, piid=4   → Pose coverage (mowing progress)
siid=1, piid=50  → Service 1 property 50
siid=1, piid=51  → Service 1 property 51
siid=1, piid=52  → Completion flag
siid=1, piid=53  → Bluetooth connected ← YOU ARE HERE
siid=1, piid=54  → Service 1 property 54
siid=1, piid=55  → Service 1 property 55
```

## Summary

- **Bluetooth**: Local status monitoring only
- **Cloud/MQTT**: All actual command control
- **Status**: Reported as boolean (true/false)
- **Update Interval**: Real-time via MQTT when status changes
- **Required for Control**: NO - Cloud is primary channel
- **User Benefit**: Local awareness of device connectivity
