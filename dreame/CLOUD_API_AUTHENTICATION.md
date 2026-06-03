# Dreame Cloud API Authentication Flow

## Overview

The Dreame mower integration uses cloud-based REST authentication to obtain credentials for both REST API calls and MQTT connections. Authentication is implemented in the `DreameMowerCloudBase` class.

## Authentication Sequence

```
┌──────────────┐
│   Username   │
│  + Password  │
└──────┬───────┘
       │
       │ MD5 Hash + Salt
       │
       ▼
┌─────────────────────────────────┐
│  Prepare Login Payload          │
│  - account_type (dreame/mova)   │
│  - country (us/eu/cn/etc)       │
└──────┬──────────────────────────┘
       │
       │ REST POST
       │ https://{country}.dreame-cloud.net:9999/
       │ auth/auth/authenticate
       │
       ▼
┌─────────────────────────────────┐
│  Dreame Cloud Server            │
│  - Validates credentials        │
│  - Generates session            │
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│  Response Contains:             │
│  - token (Session token)        │
│  - refresh_token                │
│  - token_expire (seconds)       │
│  - uid (User ID)                │
│  - Location info                │
└──────┬──────────────────────────┘
       │
       │ Store Credentials
       │
       ▼
┌─────────────────────────────────┐
│  Ready for:                     │
│  - REST API calls               │
│  - MQTT connections             │
│  - Device commands              │
└─────────────────────────────────┘
```

## API Strings Obfuscation

API endpoints are obfuscated using Base64 + zlib compression:

```python
# Encoded strings
DREAME_STRINGS = "H4sICAAAAAAEAGNsb3VkX3N0cmluZ3MuanNvbgBdUltv..."
MOVA_STRINGS = "H4sIAAAAAAAAA11Sa2/aMBT9K6hS0SYtIQktYar..."

# Decoding function
def _decode_api_strings(encoded: str) -> List[str]:
    return json.loads(
        zlib.decompress(
            base64.b64decode(encoded), 
            zlib.MAX_WBITS | 32
        )
    )

# Result: Array of ~60 API endpoints and strings
api_strings[0]  = Host domain
api_strings[1]  = Port
api_strings[17] = /auth/authenticate endpoint
api_strings[18] = "token" key name
api_strings[23] = /api/v1 prefix
# ... and many more
```

## Connection Parameters

### Login Endpoint
```
URL: https://{country}.dreame-cloud.net:9999/auth/auth/authenticate
Method: POST
Content-Type: application/x-www-form-urlencoded
```

### Request Payload
```python
data = f"{salt}{separator}password_type{separator}{username}{separator}pwd_key{separator}{md5_hashed_password}"

# Example:
# "salt_value=accountpassword_md5_hash_with_salt"
```

### Response Format
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_expire": 86400,
    "uid": "12345678",
    "country": "us",
    "ti": "timestamp_identifier"
  }
}
```

## Token Management

### Token Lifecycle
```
┌─────────────────────────────────────────────┐
│  Token Obtained                             │
│  Expiry: current_time + token_expire - 120s │
│  (120s buffer for refresh)                  │
└──────────────┬────────────────────────────────┘
               │
        Time passes │
               │
               ▼
┌─────────────────────────────────────────────┐
│  120 seconds before expiry                  │
│  Automatic refresh triggered                │
└──────────────┬────────────────────────────────┘
               │
        REST POST │ /auth/auth/refresh
        with refresh_token
               │
               ▼
┌─────────────────────────────────────────────┐
│  New tokens received                        │
│  Continue using new token                   │
│  (120s buffer resets)                       │
└─────────────────────────────────────────────┘
```

### Token Refresh Implementation
```python
if self._key_expire and time.time() > self._key_expire:
    # Token expired or expiring soon
    self.connect()  # Re-authenticate
```

## Account Types

### Dreame Account
```
Endpoint: {country}.dreame-cloud.net
API Strings: DREAME_STRINGS (decoded)
Account Type: "dreame"
Apps: Dreamehome app
Devices: Dreame G10, Dreame D9, etc.
```

### MOVA Account
```
Endpoint: {country}.dreame-cloud.net (same)
API Strings: MOVA_STRINGS (decoded)
Account Type: "mova"
Apps: MOVAhome app
Devices: MOVA Z500, MOVA Y1000, etc.
```

## Country Codes

```python
VALID_COUNTRIES = [
    "us",   # United States
    "eu",   # Europe
    "cn",   # China
    # ... additional regional endpoints
]

# Base URL construction
api_url = f"https://{country}.dreame-cloud.net:9999/"
```

## REST API Headers

```python
headers = {
    "Accept": "*/*",
    "Content-Type": "application/x-www-form-urlencoded",
    "Accept-Language": "en-US;q=0.8",
    "Accept-Encoding": "gzip, deflate",
    "User-Agent": "dreame_cloud_api/1.0",
    "Authorization": f"Bearer {token}",
}

# Country-specific headers
if country == "cn":
    headers["X-Custom-Header"] = "cn_specific_value"
```

## Error Handling

### Common Error Codes
```
code=0      → Success
code=401    → Token expired (trigger refresh)
code=403    → Forbidden (auth failure)
code=500    → Server error (retry)
code=80001  → Device offline
```

### Error Recovery
```python
if response.status_code == 401:
    if self._secondary_key:
        # Try to refresh with secondary key
        self._secondary_key = None
        return self.connect()
    else:
        # Full re-authentication required
        raise AuthenticationError("Authentication failed")

elif response.status_code == 500:
    # Retry with backoff
    retry_count -= 1
    if retry_count > 0:
        time.sleep(1)
        # Retry the request
```

## Complete Authentication Example

```python
from dreame_mower_integration import DreameMowerCloudBase

# Initialize
cloud = DreameMowerCloudBase(
    username="user@example.com",
    password="secure_password",
    country="us",
    account_type="dreame"
)

# Authenticate
if cloud.connect():
    print(f"Authenticated as: {cloud._uuid}")
    print(f"Token expires at: {cloud._key_expire}")
    
    # Can now make API calls
    devices = cloud.get_devices()
    print(f"Devices: {devices}")
else:
    print("Authentication failed")
```

## Security Considerations

✓ **Credentials**: 
- Passwords hashed with MD5 + salt
- Never transmitted in plaintext
- Stored securely in Home Assistant config

✓ **Tokens**:
- Time-limited (24-48 hours)
- Automatically refreshed before expiry
- Included in Bearer authentication header

✓ **Transport**:
- HTTPS TLS 1.2+ mandatory
- Self-signed certificates ignored for MQTT (necessary for cloud broker)

⚠️ **Refresh Token**:
- Stored securely for token refresh
- Used only for re-authentication
- Never exposed to external services

## Integration with MQTT

```python
# After REST authentication, credentials used for MQTT
mqtt_username = cloud._uuid      # From REST auth
mqtt_password = cloud._key       # From REST auth
mqtt_host = device_info["host"]  # From device API
mqtt_port = 8883                 # Standard TLS MQTT

# MQTT client setup
client.username_pw_set(mqtt_username, mqtt_password)
client.tls_set()
client.connect(mqtt_host, mqtt_port)
```

## Retry Strategy

```python
retries = 0
max_retries = 3
timeout = 20  # seconds

while retries < max_retries + 1:
    try:
        response = session.post(
            url,
            headers=headers,
            data=data,
            timeout=timeout
        )
        if response.status_code == 200:
            return json.loads(response.text)
        elif response.status_code == 401:
            # Token expired - refresh and retry
            self.connect()
            continue
    except requests.exceptions.Timeout:
        retries += 1
        if retries < max_retries:
            time.sleep(1)  # Wait before retry
    except Exception as ex:
        # Handle other errors
        break

if retries >= max_retries:
    raise ConnectionError("Max retries exceeded")
```

## Testing Authentication

```bash
# Using curl to test login endpoint
curl -X POST https://us.dreame-cloud.net:9999/auth/auth/authenticate \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=user@example.com&password=hashed_password"

# Expected response
{
  "code": 0,
  "data": {
    "token": "...",
    "refresh_token": "...",
    "token_expire": 86400,
    "uid": "..."
  }
}
```

## Troubleshooting

### "Login failed: Read timed out"
- Cloud server unreachable
- Network connectivity issue
- Firewall blocking port 9999

### "Login failed: error_description contains 'refresh token'"
- Refresh token is invalid or expired
- Secondary key not available
- Full re-authentication required

### "Invalid account type selected"
- account_type must be "dreame" or "mova"
- Wrong app credentials for device type
- Check device manual for correct account type

## Summary

✓ Username/Password → MD5 Hash + Salt  
✓ REST POST to cloud auth endpoint  
✓ Receive session token + refresh token  
✓ Token auto-refresh 120s before expiry  
✓ Tokens used for both REST API and MQTT  
✓ Country-specific endpoints (us/eu/cn)  
✓ Account types: dreame or mova  
