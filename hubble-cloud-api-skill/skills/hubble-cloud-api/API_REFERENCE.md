# Hubble Cloud API Reference

Complete reference documentation for all Hubble Cloud API endpoints with enhanced descriptions, examples, and best practices.

## Table of Contents

- [Base URL & Authentication](#base-url--authentication)
- [API Key Management](#api-key-management)
- [Device Management](#device-management)
- [Packet Retrieval](#packet-retrieval)
- [Webhook Management](#webhook-management)
- [Metrics](#metrics)
- [Organization Management](#organization-management)
- [User Management](#user-management)
- [Invitation Management](#invitation-management)
- [Billing](#billing)

## Base URL & Authentication

**Base URL**: `https://api.hubble.com`

**Authentication**: All endpoints require Bearer token authentication:

```http
Authorization: Bearer <your-jwt-token>
```

**Organization ID**: Required in URL path for all endpoints. Find your organization ID at:
- Hubble Dashboard → Developer → API Tokens
- Or Organization Settings

## API Key Management

### Check API Key Validity

**Endpoint**: `GET /api/org/{org_id}/check`

**Purpose**: Validate that your current API key is active and has valid scopes.

**When to use**:
- Testing new API keys after generation
- Debugging authentication issues
- Health checks in monitoring systems

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/check" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "valid": true,
  "scopes": ["devices:read", "devices:write", "packets:read"],
  "organization_id": "7db20441-a04c-49ba-8089-1197b3a66a4b",
  "expires_at": "2025-12-31T23:59:59Z"
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid or expired token
- `403 Forbidden`: Token valid but lacks permissions

---

### Create API Key

**Endpoint**: `POST /api/org/{org_id}/key`

**Purpose**: Generate a new API key with specified authorization scopes.

**When to use**:
- Initial API integration setup
- Key rotation workflows
- Creating application-specific keys

**Request**:
```bash
curl -X POST "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/key" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production API Key",
    "scopes": ["devices:read", "devices:write", "packets:read", "webhooks:write"]
  }'
```

**Request Body**:
```json
{
  "name": "string",        // Human-readable key name
  "scopes": ["string"]     // Array of scope strings
}
```

**Available Scopes**:
- `api_keys:read`, `api_keys:write`
- `devices:read`, `devices:write`
- `packets:read`
- `webhooks:read`, `webhooks:write`
- `users:read`, `users:write`
- `invitations:read`, `invitations:write`
- `organization:read`, `organization:write`
- `metrics:read`
- `billing:read`

**Response** (201 Created):
```json
{
  "key_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production API Key",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "scopes": ["devices:read", "devices:write", "packets:read", "webhooks:write"],
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Important**: The `token` field is only returned once. Store it securely immediately.

**Error Responses**:
- `400 Bad Request`: Invalid scope requested
- `403 Forbidden`: Insufficient permissions to create keys

---

### List API Keys

**Endpoint**: `GET /api/org/{org_id}/key`

**Purpose**: List all API keys for your organization (tokens are never returned, only metadata).

**When to use**:
- Auditing active API keys
- Identifying keys to rotate or revoke
- Checking key scopes

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/key" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "keys": [
    {
      "key_id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Production API Key",
      "scopes": ["devices:read", "devices:write", "packets:read"],
      "created_at": "2024-01-15T10:30:00Z",
      "last_used_at": "2024-01-20T14:22:00Z"
    }
  ]
}
```

---

### Update API Key Metadata

**Endpoint**: `PATCH /api/org/{org_id}/key/{key_id}`

**Purpose**: Update API key name or other metadata (scopes cannot be changed; create new key instead).

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/key/550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Renamed Production Key"}'
```

**Response** (200 OK):
```json
{
  "key_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Renamed Production Key",
  "scopes": ["devices:read", "devices:write"],
  "updated_at": "2024-01-21T09:15:00Z"
}
```

---

### Delete API Key

**Endpoint**: `DELETE /api/org/{org_id}/key/{key_id}`

**Purpose**: Revoke an API key (immediate effect, cannot be undone).

**When to use**:
- Key rotation after creating replacement
- Security incident response
- Decommissioning applications

**Request**:
```bash
curl -X DELETE "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/key/550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "message": "API key successfully deleted",
  "key_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Warning**: This operation is immediate. Any systems using the deleted key will receive 401 errors.

---

### List Available Scopes

**Endpoint**: `GET /api/org/{org_id}/key_scopes`

**Purpose**: Get complete list of authorization scopes available for API keys.

**When to use**:
- Before creating new API keys
- Understanding permission model
- Documenting integrations

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/key_scopes" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "scopes": [
    {
      "scope": "devices:read",
      "description": "View device information and list devices"
    },
    {
      "scope": "devices:write",
      "description": "Register, update, and delete devices"
    },
    {
      "scope": "packets:read",
      "description": "Query and stream packet data"
    }
  ]
}
```

---

## Device Management

### Register Device

**Endpoint**: `POST /api/v2/org/{org_id}/devices`

**Purpose**: Register a new Bluetooth device with Hubble Network before it can transmit data.

**When to use**:
- Initial device onboarding
- Mass device provisioning
- Replacing failed devices

**Request**:
```bash
curl -X POST "https://api.hubble.com/api/v2/org/7db20441-a04c-49ba-8089-1197b3a66a4b/devices" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Temperature Sensor #1",
    "dev_eui": "70B3D54996C4C5A7",
    "device_key": "YmFzZTY0X2VuY29kZWRfa2V5X2hlcmU=",
    "tags": {
      "location": "warehouse-a",
      "sensor_type": "temperature"
    }
  }'
```

**Request Body**:
```json
{
  "name": "string",           // Human-readable device name
  "dev_eui": "string",        // Unique device EUI (16 hex characters)
  "device_key": "string",     // Base64-encoded encryption key (32 bytes)
  "tags": {                   // Optional custom tags
    "key": "value"
  }
}
```

**Important**: `device_key` must be a Base64-encoded 32-byte encryption key. Generate with:
```python
import base64, os
device_key = base64.b64encode(os.urandom(32)).decode('utf-8')
```

**Response** (201 Created):
```json
{
  "device": {
    "id": "uuid-string",
    "name": "Temperature Sensor #1",
    "dev_eui": "70B3D54996C4C5A7",
    "status": "active",
    "tags": {
      "location": "warehouse-a",
      "sensor_type": "temperature",
      "hubnet.platform": "BLE"
    },
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  }
}
```

**Platform Tags**: Hubble automatically adds `hubnet.platform` tag (e.g., `BLE`, `LoRaWAN`, `Sigfox`).

**Error Responses**:
- `400 Bad Request`: Invalid device_key format (not Base64), missing required fields
- `409 Conflict`: Device with this dev_eui already exists

---

### List Devices

**Endpoint**: `GET /api/org/{org_id}/devices`

**Purpose**: Retrieve all devices registered to your organization with filtering and sorting.

**When to use**:
- Device inventory management
- Finding devices by tags or name patterns
- Exporting device metadata
- Preparing for batch updates

**Query Parameters**:
- `platform_tag` (string): Filter by platform tag (e.g., `hubnet.platform=LoRaWAN`)
- `tag` (string): Filter by custom tag (e.g., `location=warehouse-a`)
- `name_pattern` (string): Filter devices by name (case-insensitive substring match)
- `created_after` (ISO-8601): Only devices created after this timestamp
- `created_before` (ISO-8601): Only devices created before this timestamp
- `sort` (string): Sort field (`name`, `created_at`, `updated_at`)
- `order` (string): Sort order (`asc`, `desc`)
- `limit` (integer): Max results per page (default 100, max 1000)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/devices?platform_tag=hubnet.platform%3DLoRaWAN&sort=created_at&order=desc&limit=50" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "devices": [
    {
      "id": "device-uuid-1",
      "name": "Temperature Sensor #1",
      "dev_eui": "70B3D54996C4C5A7",
      "status": "active",
      "tags": {
        "location": "warehouse-a",
        "hubnet.platform": "LoRaWAN"
      },
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-20T14:22:00Z"
    }
  ],
  "total_count": 127,
  "page_size": 50
}
```

**Pagination**: Use `Continuation-Token` header if `total_count > page_size`.

**Example: Find All Warehouse Devices**:
```bash
curl "https://api.hubble.com/api/org/YOUR_ORG_ID/devices?tag=location%3Dwarehouse-a"
```

---

### Get Device Details

**Endpoint**: `GET /api/org/{org_id}/devices/{device_id}`

**Purpose**: Retrieve complete information for a specific device.

**When to use**:
- Verifying device registration
- Checking device status and tags
- Retrieving device metadata before updates

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/devices/device-uuid-1" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "device": {
    "id": "device-uuid-1",
    "name": "Temperature Sensor #1",
    "dev_eui": "70B3D54996C4C5A7",
    "device_key": "YmFzZTY0X2VuY29kZWRfa2V5X2hlcmU=",
    "status": "active",
    "tags": {
      "location": "warehouse-a",
      "sensor_type": "temperature",
      "hubnet.platform": "LoRaWAN"
    },
    "last_seen_at": "2024-01-20T14:22:00Z",
    "packet_count": 1523,
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-20T14:22:00Z"
  }
}
```

**Error Responses**:
- `404 Not Found`: Device ID doesn't exist or belongs to different organization

---

### Update Device

**Endpoint**: `PATCH /api/org/{org_id}/devices/{device_id}`

**Purpose**: Update device metadata, name, or tags.

**When to use**:
- Renaming devices
- Adding or updating custom tags
- Changing device configuration

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/devices/device-uuid-1" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Temperature Sensor - Updated Name",
    "tags": {
      "location": "warehouse-b",
      "maintenance": "scheduled-2024-02"
    }
  }'
```

**Request Body** (all fields optional):
```json
{
  "name": "string",
  "tags": {
    "key": "value"
  }
}
```

**Response** (200 OK):
```json
{
  "device": {
    "id": "device-uuid-1",
    "name": "Temperature Sensor - Updated Name",
    "dev_eui": "70B3D54996C4C5A7",
    "tags": {
      "location": "warehouse-b",
      "maintenance": "scheduled-2024-02",
      "hubnet.platform": "LoRaWAN"
    },
    "updated_at": "2024-01-21T09:15:00Z"
  }
}
```

**Note**: Tags are merged, not replaced. To remove a tag, set its value to `null`.

---

### Batch Update Devices

**Endpoint**: `PATCH /api/org/{org_id}/devices`

**Purpose**: Update multiple devices in a single request (up to 1,000 devices).

**When to use**:
- Mass tag updates (e.g., deployment location changes)
- Batch renaming with patterns
- Fleet-wide configuration changes
- Optimizing API usage and rate limits

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/devices" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "updates": [
      {
        "device_id": "device-uuid-1",
        "name": "Sensor 001",
        "tags": {"deployment": "phase-2"}
      },
      {
        "device_id": "device-uuid-2",
        "name": "Sensor 002",
        "tags": {"deployment": "phase-2"}
      }
    ]
  }'
```

**Request Body**:
```json
{
  "updates": [
    {
      "device_id": "string",    // Required
      "name": "string",          // Optional
      "tags": {"key": "value"}   // Optional
    }
  ]
}
```

**Constraints**:
- Maximum 1,000 devices per request
- Each update is atomic (either succeeds or fails independently)
- Partial failures are reported in response

**Response** (200 OK):
```json
{
  "results": [
    {
      "device_id": "device-uuid-1",
      "status": "success",
      "device": { /* updated device object */ }
    },
    {
      "device_id": "device-uuid-2",
      "status": "error",
      "error": "Device not found"
    }
  ],
  "total_requested": 2,
  "successful": 1,
  "failed": 1
}
```

**Best Practice**: Check `results` array for per-device status. Don't assume all updates succeeded.

---

## Packet Retrieval

### Stream Packets

**Endpoint**: `GET /api/org/{org_id}/packets`

**Purpose**: Retrieve packets with streaming pagination using continuation tokens.

**When to use**:
- Historical packet analysis
- Batch data processing
- Backfilling databases
- Data export for analytics

**Query Parameters**:
- `start_time` (ISO-8601): Start of time range (default: 7 days ago)
- `end_time` (ISO-8601): End of time range (default: now)
- `device_id` (string): Filter by specific device UUID
- `platform_tag` (string): Filter by platform tag
- `limit` (integer): Results per page (default 100, max 1000)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/packets?start_time=2024-01-15T00:00:00Z&end_time=2024-01-16T00:00:00Z&limit=500" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "packets": [
    {
      "packet_id": "packet-uuid-1",
      "device_id": "device-uuid-1",
      "payload": "SGVsbG8gV29ybGQ=",
      "sequence_number": 1234,
      "received_at": "2024-01-15T10:30:00Z",
      "gateway_id": "gateway-abc123",
      "rssi": -95,
      "snr": 8.5,
      "metadata": {
        "latitude": 37.7749,
        "longitude": -122.4194,
        "accuracy": 50
      }
    }
  ],
  "count": 500,
  "has_more": true
}
```

**Response Headers**:
```
Continuation-Token: eyJsYXN0X3RpbWVzdGFtcCI6MTY0MDAwMDAwMH0=
```

**Pagination Pattern**:
1. Make initial request
2. Process packets
3. Check for `Continuation-Token` header
4. If present, repeat request with `Continuation-Token` header set
5. Continue until no token returned

**Example with Continuation**:
```bash
# Second request
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/packets?start_time=2024-01-15T00:00:00Z" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Continuation-Token: eyJsYXN0X3RpbWVzdGFtcCI6MTY0MDAwMDAwMH0="
```

**Payload Decoding**: Packet payloads are Base64-encoded:
```python
import base64
decoded_payload = base64.b64decode("SGVsbG8gV29ybGQ=")
# Result: b'Hello World'
```

**Time Range Best Practices**:
- Default 7-day window balances performance and completeness
- For large exports, use smaller time windows (1-day chunks)
- Timestamps are UTC (ISO-8601 format)

---

### Test Webhook (Example Endpoint)

**Endpoint**: `POST /api/webhook/testBatch`

**Purpose**: Example endpoint showing webhook request format (for testing your webhook implementation).

**When to use**:
- Testing webhook endpoint before registering
- Understanding webhook payload structure
- Debugging webhook integrations

**Example Webhook Request from Hubble**:
```http
POST /your/webhook/endpoint HTTP/1.1
Host: your-domain.com
Content-Type: application/json
HTTP-X-HUBBLE-TOKEN: your-webhook-secret

{
  "packets": [
    {
      "packet_id": "packet-uuid-1",
      "device_id": "device-uuid-1",
      "payload": "SGVsbG8gV29ybGQ=",
      "sequence_number": 1234,
      "received_at": "2024-01-15T10:30:00Z",
      "metadata": { ... }
    }
  ]
}
```

**Webhook Response Requirements**:
- Return `200 OK` for successful receipt
- Any 2xx status code accepted
- Non-2xx triggers automatic retry with exponential backoff
- Timeout: 30 seconds

---

## Webhook Management

### Create Webhook

**Endpoint**: `POST /api/org/{org_id}/webhooks`

**Purpose**: Register a webhook endpoint to receive real-time packet notifications.

**When to use**:
- Setting up real-time packet processing
- Event-driven architectures
- Avoiding API polling

**Request**:
```bash
curl -X POST "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/webhooks" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/webhook/hubble",
    "name": "Production Webhook",
    "max_batch_size": 100,
    "secret": "your-secret-token-for-validation"
  }'
```

**Request Body**:
```json
{
  "url": "string",              // HTTPS endpoint URL
  "name": "string",             // Human-readable name
  "max_batch_size": 100,        // Packets per request (10-1000)
  "secret": "string"            // Optional validation token
}
```

**Response** (201 Created):
```json
{
  "webhook": {
    "webhook_id": "webhook-uuid-1",
    "url": "https://your-domain.com/webhook/hubble",
    "name": "Production Webhook",
    "max_batch_size": 100,
    "secret": "your-secret-token-for-validation",
    "status": "active",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

**Webhook Behavior**:
- Hubble batches packets up to `max_batch_size`
- Sends POST request to your URL
- Includes `HTTP-X-HUBBLE-TOKEN` header with your secret
- Retries on failure with exponential backoff

**Security**: Always validate `HTTP-X-HUBBLE-TOKEN` header to prevent spoofing.

---

### List Webhooks

**Endpoint**: `GET /api/org/{org_id}/webhooks`

**Purpose**: List all registered webhooks for your organization.

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/webhooks" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "webhooks": [
    {
      "webhook_id": "webhook-uuid-1",
      "url": "https://your-domain.com/webhook/hubble",
      "name": "Production Webhook",
      "max_batch_size": 100,
      "status": "active",
      "last_delivery_at": "2024-01-20T14:22:00Z",
      "total_deliveries": 15234,
      "failed_deliveries": 12,
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

### Update Webhook

**Endpoint**: `PATCH /api/org/{org_id}/webhooks/{webhook_id}`

**Purpose**: Update webhook configuration (URL, batch size, secret).

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/webhooks/webhook-uuid-1" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "max_batch_size": 500,
    "name": "Updated Production Webhook"
  }'
```

**Request Body** (all fields optional):
```json
{
  "url": "string",
  "name": "string",
  "max_batch_size": 500,
  "secret": "string"
}
```

**Response** (200 OK):
```json
{
  "webhook": {
    "webhook_id": "webhook-uuid-1",
    "url": "https://your-domain.com/webhook/hubble",
    "name": "Updated Production Webhook",
    "max_batch_size": 500,
    "updated_at": "2024-01-21T09:15:00Z"
  }
}
```

---

### Delete Webhook

**Endpoint**: `DELETE /api/org/{org_id}/webhooks/{webhook_id}`

**Purpose**: Remove webhook registration (stops all packet deliveries to this endpoint).

**Request**:
```bash
curl -X DELETE "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/webhooks/webhook-uuid-1" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "message": "Webhook successfully deleted",
  "webhook_id": "webhook-uuid-1"
}
```

---

## Metrics

### API Metrics

**Endpoint**: `GET /api/org/{org_id}/api_metrics`

**Purpose**: Track API request volumes, success rates, and performance over time.

**Query Parameters**:
- `days_back` (integer): Number of days to query (1-365, default 7)
- `interval` (string): Aggregation interval (`hour`, `day`, `month`)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/api_metrics?days_back=30&interval=day" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "metrics": [
    {
      "timestamp": "2024-01-15T00:00:00Z",
      "total_requests": 15234,
      "successful_requests": 15102,
      "failed_requests": 132,
      "rate_limited_requests": 45,
      "avg_response_time_ms": 125,
      "endpoints": {
        "/devices": 8523,
        "/packets": 5234,
        "/webhooks": 1477
      }
    }
  ],
  "summary": {
    "total_requests": 456789,
    "success_rate": 99.1,
    "avg_response_time_ms": 130
  }
}
```

**Use Cases**:
- Monitoring API health
- Identifying rate limit patterns
- Performance optimization
- Usage forecasting

---

### Packet Metrics

**Endpoint**: `GET /api/org/{org_id}/packet_metrics`

**Purpose**: Track packet volumes, device activity, and data throughput.

**Query Parameters**:
- `days_back` (integer): Number of days (1-365, default 7)
- `interval` (string): Aggregation (`hour`, `day`, `month`)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/packet_metrics?days_back=7&interval=hour" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "metrics": [
    {
      "timestamp": "2024-01-15T10:00:00Z",
      "packet_count": 1523,
      "unique_devices": 87,
      "total_bytes": 152300,
      "avg_packets_per_device": 17.5,
      "platforms": {
        "LoRaWAN": 1200,
        "BLE": 323
      }
    }
  ],
  "summary": {
    "total_packets": 256789,
    "total_devices": 127,
    "total_bytes": 25678900
  }
}
```

---

### Webhook Metrics

**Endpoint**: `GET /api/org/{org_id}/webhook_metrics`

**Purpose**: Monitor webhook delivery success rates, failures, and performance.

**Query Parameters**:
- `days_back` (integer): Number of days (1-365, default 7)
- `interval` (string): Aggregation (`hour`, `day`, `month`)
- `webhook_id` (string): Filter by specific webhook

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/webhook_metrics?days_back=7" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "metrics": [
    {
      "webhook_id": "webhook-uuid-1",
      "webhook_name": "Production Webhook",
      "timestamp": "2024-01-15T10:00:00Z",
      "deliveries_attempted": 523,
      "deliveries_successful": 518,
      "deliveries_failed": 5,
      "avg_response_time_ms": 245,
      "retries": 15
    }
  ],
  "summary": {
    "total_deliveries": 15234,
    "success_rate": 99.7,
    "avg_response_time_ms": 250
  }
}
```

**Alerting Recommendations**:
- Success rate < 95%: Investigate endpoint health
- Avg response time > 5000ms: Performance issues
- Retry count increasing: Temporary failures

---

### Device Metrics

**Endpoint**: `GET /api/org/{org_id}/device_metrics`

**Purpose**: Track device registration, activity, and fleet health over time.

**Query Parameters**:
- `days_back` (integer): Number of days (1-365, default 7)
- `interval` (string): Aggregation (`day`, `month`)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/device_metrics?days_back=30&interval=day" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "metrics": [
    {
      "timestamp": "2024-01-15T00:00:00Z",
      "total_devices": 127,
      "active_devices": 119,
      "inactive_devices": 8,
      "new_registrations": 3
    }
  ],
  "summary": {
    "current_total": 127,
    "current_active": 119,
    "activity_rate": 93.7
  }
}
```

**Active Device Definition**: Device that sent at least one packet in the time period.

---

## Organization Management

### Get Organization

**Endpoint**: `GET /api/org/{org_id}`

**Purpose**: Retrieve organization details, settings, and configuration.

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "organization": {
    "id": "7db20441-a04c-49ba-8089-1197b3a66a4b",
    "name": "Acme IoT Solutions",
    "address": {
      "street": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94105",
      "country": "USA"
    },
    "contact": {
      "email": "[email protected]",
      "phone": "+1-555-0100"
    },
    "timezone": "America/Los_Angeles",
    "created_at": "2023-06-01T00:00:00Z",
    "plan": "enterprise"
  }
}
```

---

### Update Organization

**Endpoint**: `PATCH /api/org/{org_id}`

**Purpose**: Update organization name, address, contact info, or timezone.

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme IoT Solutions Inc.",
    "timezone": "America/New_York"
  }'
```

**Request Body** (all fields optional):
```json
{
  "name": "string",
  "address": {
    "street": "string",
    "city": "string",
    "state": "string",
    "postal_code": "string",
    "country": "string"
  },
  "contact": {
    "email": "string",
    "phone": "string"
  },
  "timezone": "string"
}
```

**Response** (200 OK):
```json
{
  "organization": { /* updated organization object */ }
}
```

---

## User Management

### List Users

**Endpoint**: `GET /api/org/{org_id}/users`

**Purpose**: List all users in your organization with roles and status.

**Query Parameters**:
- `page` (integer): Page number (default 1)
- `per_page` (integer): Results per page (default 50, max 100)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/users?per_page=50" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "users": [
    {
      "user_id": "user-uuid-1",
      "email": "[email protected]",
      "name": "John Doe",
      "role": "admin",
      "status": "active",
      "last_login_at": "2024-01-20T14:22:00Z",
      "created_at": "2023-06-01T00:00:00Z"
    }
  ],
  "total_count": 12,
  "page": 1,
  "per_page": 50
}
```

**Roles**:
- `admin`: Full access to organization
- `member`: Limited access (no billing, user management)

---

### Add User

**Endpoint**: `POST /api/org/{org_id}/users`

**Purpose**: Add an existing Hubble user directly to your organization (user must have Hubble account).

**Request**:
```bash
curl -X POST "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/users" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "email": "[email protected]",
    "role": "member"
  }'
```

**Response** (201 Created):
```json
{
  "user": {
    "user_id": "user-uuid-2",
    "email": "[email protected]",
    "role": "member",
    "status": "active",
    "added_at": "2024-01-21T09:15:00Z"
  }
}
```

**Error Responses**:
- `404 Not Found`: No Hubble account with this email (use Invitations instead)
- `409 Conflict`: User already in organization

---

### Update User

**Endpoint**: `PATCH /api/org/{org_id}/users/{user_id}`

**Purpose**: Update user role or name.

**Request**:
```bash
curl -X PATCH "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/users/user-uuid-2" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

**Response** (200 OK):
```json
{
  "user": {
    "user_id": "user-uuid-2",
    "email": "[email protected]",
    "role": "admin",
    "updated_at": "2024-01-21T09:15:00Z"
  }
}
```

---

### Remove User

**Endpoint**: `DELETE /api/org/{org_id}/users/{user_id}`

**Purpose**: Remove user from organization (revokes all access).

**Request**:
```bash
curl -X DELETE "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/users/user-uuid-2" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "message": "User successfully removed",
  "user_id": "user-uuid-2"
}
```

---

## Invitation Management

### List Invitations

**Endpoint**: `GET /api/org/{org_id}/invitations`

**Purpose**: List pending invitations (not yet accepted).

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/invitations" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "invitations": [
    {
      "invitation_id": "invite-uuid-1",
      "email": "[email protected]",
      "role": "member",
      "status": "pending",
      "invited_by": "user-uuid-1",
      "created_at": "2024-01-20T10:00:00Z",
      "expires_at": "2024-01-27T10:00:00Z"
    }
  ]
}
```

---

### Send Invitation

**Endpoint**: `POST /api/org/{org_id}/invitations`

**Purpose**: Invite new user to join organization via email.

**Request**:
```bash
curl -X POST "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/invitations" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "email": "[email protected]",
    "role": "member",
    "message": "Join our IoT platform team!"
  }'
```

**Response** (201 Created):
```json
{
  "invitation": {
    "invitation_id": "invite-uuid-1",
    "email": "[email protected]",
    "role": "member",
    "status": "pending",
    "expires_at": "2024-01-27T10:00:00Z"
  }
}
```

**Invitation Expiration**: Invitations expire after 7 days.

---

### Revoke Invitation

**Endpoint**: `DELETE /api/org/{org_id}/invitations/{invitation_id}`

**Purpose**: Cancel pending invitation before acceptance.

**Request**:
```bash
curl -X DELETE "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/invitations/invite-uuid-1" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "message": "Invitation successfully revoked",
  "invitation_id": "invite-uuid-1"
}
```

---

## Billing

### List Invoices

**Endpoint**: `GET /api/org/{org_id}/billing/invoices`

**Purpose**: Retrieve billing invoices for your organization.

**Query Parameters**:
- `limit` (integer): Max invoices to return (default 12)

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/billing/invoices" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "invoices": [
    {
      "invoice_id": "inv-uuid-1",
      "invoice_number": "INV-2024-001",
      "period_start": "2024-01-01T00:00:00Z",
      "period_end": "2024-01-31T23:59:59Z",
      "amount_due": 1250.00,
      "currency": "USD",
      "status": "paid",
      "due_date": "2024-02-15T00:00:00Z",
      "paid_at": "2024-02-10T14:30:00Z"
    }
  ]
}
```

---

### Download Invoice PDF

**Endpoint**: `GET /api/org/{org_id}/billing/invoices/{invoice_id}/pdf`

**Purpose**: Download PDF version of invoice.

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/billing/invoices/inv-uuid-1/pdf" \
  -H "Authorization: Bearer eyJhbGc..." \
  --output invoice-2024-001.pdf
```

**Response**: Binary PDF data with `Content-Type: application/pdf` header.

---

### Get Usage Data

**Endpoint**: `GET /api/org/{org_id}/billing/usage`

**Purpose**: Track daily and monthly active device usage for billing.

**Query Parameters**:
- `start_date` (YYYY-MM-DD): Start of date range
- `end_date` (YYYY-MM-DD): End of date range

**Request**:
```bash
curl -X GET "https://api.hubble.com/api/org/7db20441-a04c-49ba-8089-1197b3a66a4b/billing/usage?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer eyJhbGc..."
```

**Response** (200 OK):
```json
{
  "usage": [
    {
      "date": "2024-01-15",
      "active_devices": 119,
      "total_packets": 15234,
      "billable_devices": 119
    }
  ],
  "summary": {
    "period_start": "2024-01-01",
    "period_end": "2024-01-31",
    "avg_daily_active_devices": 117.5,
    "max_active_devices": 127,
    "total_packets": 456789
  }
}
```

**Billing Note**: Active device = device that sent at least one packet in the billing period.

---

## Common Patterns & Best Practices

### Pagination with Continuation Tokens

Many endpoints use continuation tokens for efficient streaming:

```python
def fetch_all_pages(url, headers):
    results = []
    continuation_token = None

    while True:
        if continuation_token:
            headers["Continuation-Token"] = continuation_token

        response = requests.get(url, headers=headers)
        data = response.json()
        results.extend(data.get("items", []))

        continuation_token = response.headers.get("Continuation-Token")
        if not continuation_token:
            break

    return results
```

### Error Handling Template

```python
def make_hubble_request(method, endpoint, **kwargs):
    url = f"https://api.hubble.com{endpoint}"
    headers = {
        "Authorization": f"Bearer {API_TOKEN}",
        **kwargs.get("headers", {})
    }

    response = requests.request(method, url, headers=headers, **kwargs)

    if response.status_code == 401:
        raise AuthenticationError("Invalid API token")
    elif response.status_code == 403:
        raise PermissionError("Insufficient scopes")
    elif response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", 60))
        raise RateLimitError(f"Rate limited. Retry after {retry_after}s")
    elif response.status_code >= 500:
        raise ServerError("Hubble API server error")

    response.raise_for_status()
    return response.json()
```

### Base64 Encoding Helper

```python
import base64

def encode_device_key(binary_key: bytes) -> str:
    """Convert binary device key to Base64 string for API"""
    return base64.b64encode(binary_key).decode('utf-8')

def decode_packet_payload(b64_payload: str) -> bytes:
    """Decode Base64 packet payload to binary"""
    return base64.b64decode(b64_payload)
```

---

## Rate Limiting Reference

| Limit Type | Value | Scope |
|------------|-------|-------|
| Per-endpoint | 3 req/sec | Individual endpoint |
| Organization-wide | 15 req/sec | All endpoints combined |
| Burst allowance | 10 requests | Short bursts before limiting |

**Headers on Every Response**:
```
X-RateLimit-Limit: 3
X-RateLimit-Remaining: 2
X-RateLimit-Reset: 1640000000
```

**429 Response**:
```json
{
  "error": "Rate limit exceeded",
  "retry_after": 60
}
```

---

## HTTP Status Code Reference

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| 200 | OK | Successful request |
| 201 | Created | Resource successfully created |
| 400 | Bad Request | Invalid JSON, missing fields, bad format |
| 401 | Unauthorized | Missing/invalid/expired token |
| 403 | Forbidden | Insufficient scopes |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource (e.g., device already exists) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error (retry with backoff) |
| 503 | Service Unavailable | Temporary outage (retry with backoff) |

---

## Further Reading

- **SKILL.md**: Quick reference and overview
- **WORKFLOWS.md**: Step-by-step implementation guides
- **EXAMPLES.md**: Complete, runnable code examples
- **TROUBLESHOOTING.md**: Common errors and solutions
- **OpenAPI Spec**: [resources/hubble-openapi.yaml](./resources/hubble-openapi.yaml)
