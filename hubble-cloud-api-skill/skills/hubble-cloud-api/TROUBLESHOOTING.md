# Hubble Cloud API - Troubleshooting Guide

Common errors, their causes, and solutions when working with the Hubble Cloud API.

## Table of Contents

- [Authentication Errors](#authentication-errors)
- [Device Registration Errors](#device-registration-errors)
- [Packet Retrieval Issues](#packet-retrieval-issues)
- [Webhook Problems](#webhook-problems)
- [Rate Limiting](#rate-limiting)
- [Pagination Issues](#pagination-issues)
- [Base64 Encoding Errors](#base64-encoding-errors)
- [Network and Connectivity](#network-and-connectivity)

---

## Authentication Errors

### 401 Unauthorized - Invalid Token

**Symptom**: All API requests return `401 Unauthorized`

**Possible Causes**:
1. Missing `Authorization` header
2. Invalid token format
3. Expired API token
4. Revoked API token

**Solutions**:

**Check header format**:
```python
# ❌ Wrong
headers = {"Authorization": "your-token"}

# ✓ Correct
headers = {"Authorization": "Bearer your-token"}
```

**Verify token is valid**:
```bash
curl -X GET "https://api.hubble.com/api/org/YOUR_ORG_ID/check" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Generate new token** if expired:
1. Log in to Hubble Dashboard
2. Go to Developer → API Tokens
3. Generate new token
4. Update your application

### 403 Forbidden - Insufficient Permissions

**Symptom**: Specific endpoints return `403 Forbidden`

**Possible Causes**:
1. API key lacks required scope
2. Attempting to access resources from different organization
3. Account subscription limitations

**Solutions**:

**Check API key scopes**:
```bash
curl -X GET "https://api.hubble.com/api/org/YOUR_ORG_ID/key_scopes" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Required scopes by operation**:
- Register device: `devices:write`
- List devices: `devices:read`
- Query packets: `packets:read`
- Create webhook: `webhooks:write`
- View metrics: `metrics:read`

**Solution**: Create new API key with required scopes

**Verify organization ID**:
```python
# Make sure ORG_ID matches your actual organization
response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}",
    headers={"Authorization": f"Bearer {TOKEN}"}
)
```

---

## Device Registration Errors

### 400 Bad Request - Invalid device_key format

**Symptom**: Device registration fails with "Invalid deviceKey format"

**Cause**: Device key not Base64-encoded

**Solution**:

```python
import base64
import os

# ❌ Wrong - raw binary
device_key = os.urandom(32)

# ✓ Correct - Base64-encoded
device_key_binary = os.urandom(32)
device_key = base64.b64encode(device_key_binary).decode('utf-8')

# Register device
payload = {
    "name": "Sensor 001",
    "dev_eui": "70B3D54996C4C5A7",
    "device_key": device_key  # ✓ Base64 string
}
```

### 409 Conflict - Device already exists

**Symptom**: `409 Conflict` when registering device

**Cause**: Device with same `dev_eui` already registered

**Solutions**:

**Option 1: Retrieve existing device**:
```python
# List devices and find by dev_eui
response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}/devices",
    headers={"Authorization": f"Bearer {TOKEN}"}
)
devices = response.json()['devices']
existing = next((d for d in devices if d['dev_eui'] == dev_eui), None)
```

**Option 2: Update existing device**:
```python
# Update device instead of creating
response = requests.patch(
    f"https://api.hubble.com/api/org/{ORG_ID}/devices/{device_id}",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={"name": "Updated Name"}
)
```

### Missing Required Fields

**Symptom**: `400 Bad Request` with "Missing required field" message

**Required fields for device registration**:
- `name` (string)
- `dev_eui` (string, 16 hex characters)
- `device_key` (string, Base64-encoded)

**Solution**:
```python
# Validate before sending
def validate_device_payload(payload):
    required = ['name', 'dev_eui', 'device_key']
    missing = [f for f in required if f not in payload]
    if missing:
        raise ValueError(f"Missing required fields: {missing}")
    return True

validate_device_payload(payload)
```

---

## Packet Retrieval Issues

### No Packets Returned

**Symptom**: API returns empty `packets` array

**Possible Causes**:
1. Devices haven't transmitted yet
2. Time range doesn't include packets
3. Incorrect device ID filter
4. Wrong organization ID

**Solutions**:

**Check device connectivity**:
```python
# Verify device is registered and has sent packets
response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}/devices/{DEVICE_ID}",
    headers={"Authorization": f"Bearer {TOKEN}"}
)
device = response.json()['device']
print(f"Last seen: {device.get('last_seen_at')}")
print(f"Packet count: {device.get('packet_count')}")
```

**Widen time range**:
```python
from datetime import datetime, timedelta

# Try wider time window
start_time = datetime.utcnow() - timedelta(days=30)
end_time = datetime.utcnow()

response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}/packets",
    headers={"Authorization": f"Bearer {TOKEN}"},
    params={
        "start_time": start_time.isoformat(),
        "end_time": end_time.isoformat()
    }
)
```

**Remove filters temporarily**:
```python
# Query without device_id filter to see if any packets exist
response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}/packets",
    headers={"Authorization": f"Bearer {TOKEN}"}
)
```

### Payload Decoding Errors

**Symptom**: `binascii.Error: Invalid base64-encoded string`

**Cause**: Attempting to decode invalid Base64 string

**Solution**:
```python
import base64

def safe_decode_payload(payload_b64):
    """Safely decode Base64 payload"""
    try:
        # Add padding if missing
        missing_padding = len(payload_b64) % 4
        if missing_padding:
            payload_b64 += '=' * (4 - missing_padding)

        payload_binary = base64.b64decode(payload_b64)
        return payload_binary
    except Exception as e:
        print(f"Failed to decode payload: {e}")
        return None

# Usage
payload = safe_decode_payload(packet['payload'])
```

---

## Webhook Problems

### Webhook Not Receiving Packets

**Symptom**: Webhook endpoint never receives POST requests

**Possible Causes**:
1. Webhook URL not publicly accessible
2. Firewall blocking requests
3. HTTPS certificate issues
4. No devices transmitting
5. Webhook disabled/deleted

**Diagnostic Steps**:

**1. Verify webhook is active**:
```bash
curl -X GET "https://api.hubble.com/api/org/YOUR_ORG_ID/webhooks" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**2. Check webhook metrics**:
```python
response = requests.get(
    f"https://api.hubble.com/api/org/{ORG_ID}/webhook_metrics",
    headers={"Authorization": f"Bearer {TOKEN}"},
    params={"webhook_id": WEBHOOK_ID, "days_back": 1}
)
metrics = response.json()
print(f"Delivery attempts: {metrics['summary']['total_deliveries']}")
print(f"Success rate: {metrics['summary']['success_rate']}%")
```

**3. Test endpoint accessibility**:
```bash
# Test from external location
curl -X POST "https://your-domain.com/webhook/hubble" \
  -H "Content-Type: application/json" \
  -H "HTTP-X-HUBBLE-TOKEN: your-secret" \
  -d '{"packets": []}'
```

**4. Check HTTPS certificate**:
```bash
curl -v https://your-domain.com/webhook/hubble
# Look for SSL/TLS errors
```

### Webhook Receiving 403 Errors

**Symptom**: Webhook metrics show high failure rate, endpoint returns 403

**Cause**: Token validation failing

**Solution**:
```python
# Ensure correct header name
@app.route('/webhook/hubble', methods=['POST'])
def webhook():
    # ✓ Correct header name
    token = request.headers.get('HTTP-X-HUBBLE-TOKEN')

    # ❌ Wrong
    # token = request.headers.get('X-HUBBLE-TOKEN')

    if token != WEBHOOK_SECRET:
        abort(403)

    # Process packets
    return '', 200
```

### Webhook Timeouts

**Symptom**: High failure rate, timeout errors in metrics

**Cause**: Endpoint taking too long to respond (>30 seconds)

**Solution**: Process packets asynchronously

```python
from queue import Queue
from threading import Thread

packet_queue = Queue()

def process_packets_worker():
    """Background worker to process packets"""
    while True:
        packet = packet_queue.get()
        try:
            process_packet(packet)  # Slow processing
        finally:
            packet_queue.task_done()

# Start worker thread
worker = Thread(target=process_packets_worker, daemon=True)
worker.start()

@app.route('/webhook/hubble', methods=['POST'])
def webhook():
    # Validate quickly
    token = request.headers.get('HTTP-X-HUBBLE-TOKEN')
    if token != WEBHOOK_SECRET:
        abort(403)

    # Queue packets for background processing
    packets = request.json.get('packets', [])
    for packet in packets:
        packet_queue.put(packet)

    # Return 200 immediately (< 1 second)
    return '', 200
```

---

## Rate Limiting

### 429 Too Many Requests

**Symptom**: `429 Too Many Requests` response

**Rate Limits**:
- Per-endpoint: 3 requests/second
- Organization-wide: 15 requests/second

**Solutions**:

**Implement exponential backoff**:
```python
import time

def make_request_with_retry(url, headers, max_retries=5):
    """Make request with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)

        if response.status_code != 429:
            return response

        # Rate limited - get retry delay
        retry_after = int(response.headers.get('Retry-After', 60))
        wait_time = min(retry_after, 2 ** attempt)

        print(f"Rate limited. Waiting {wait_time}s... (attempt {attempt + 1}/{max_retries})")
        time.sleep(wait_time)

    raise Exception("Max retries exceeded")
```

**Use batch operations**:
```python
# ❌ Bad - 1000 individual requests
for device_id in device_ids:
    update_device(device_id)

# ✓ Good - 1 batch request
batch_update_devices(device_ids)  # Max 1000 per batch
```

**Spread requests over time**:
```python
import time

def safe_batch_process(items, process_func, delay=0.4):
    """Process items with delay to avoid rate limits"""
    for item in items:
        process_func(item)
        time.sleep(delay)  # ~2.5 requests/sec
```

---

## Pagination Issues

### Missing Continuation Token

**Symptom**: Only getting first 1000 results

**Cause**: Not implementing continuation token pagination

**Solution**:
```python
def fetch_all_results(url, headers):
    """Fetch all results using continuation tokens"""
    all_results = []
    continuation_token = None

    while True:
        # Set continuation token if present
        if continuation_token:
            headers['Continuation-Token'] = continuation_token

        response = requests.get(url, headers=headers)
        response.raise_for_status()

        data = response.json()
        all_results.extend(data.get('items', []))

        # Check for more pages
        continuation_token = response.headers.get('Continuation-Token')
        if not continuation_token:
            break  # No more pages

        # Clean up headers for next request
        headers = dict(headers)  # Create new dict

    return all_results
```

### Continuation Token Expired

**Symptom**: `400 Bad Request` with "Invalid continuation token"

**Cause**: Token expired (valid for limited time)

**Solution**: Restart query from beginning
```python
# Don't reuse continuation tokens after long delays
# If processing takes >5 minutes, start new query
```

---

## Base64 Encoding Errors

### Invalid Base64 String

**Symptom**: `binascii.Error: Invalid base64-encoded string`

**Solutions**:

**Add padding if missing**:
```python
def fix_base64_padding(b64_string):
    """Add missing padding to Base64 string"""
    missing_padding = len(b64_string) % 4
    if missing_padding:
        b64_string += '=' * (4 - missing_padding)
    return b64_string

# Usage
payload_b64 = fix_base64_padding(packet['payload'])
payload = base64.b64decode(payload_b64)
```

**Validate before decoding**:
```python
import base64
import binascii

def is_valid_base64(s):
    """Check if string is valid Base64"""
    try:
        # Try to decode
        base64.b64decode(s, validate=True)
        return True
    except binascii.Error:
        return False

if is_valid_base64(payload_b64):
    payload = base64.b64decode(payload_b64)
```

### String vs Bytes Confusion

**Symptom**: `TypeError: expected bytes-like object`

**Solution**:
```python
# Encoding (binary → Base64 string)
binary_data = b'Hello'
b64_string = base64.b64encode(binary_data).decode('utf-8')  # ✓ String

# Decoding (Base64 string → binary)
b64_string = "SGVsbG8="
binary_data = base64.b64decode(b64_string)  # ✓ Bytes

# ❌ Wrong - encoding string directly
# base64.b64encode("Hello")  # TypeError
```

---

## Network and Connectivity

### Connection Timeout

**Symptom**: `requests.exceptions.ConnectTimeout`

**Solutions**:

**Increase timeout**:
```python
response = requests.get(
    url,
    headers=headers,
    timeout=30  # 30 seconds
)
```

**Check network connectivity**:
```bash
# Test DNS resolution
nslookup api.hubble.com

# Test connectivity
ping api.hubble.com

# Test HTTPS
curl -v https://api.hubble.com
```

### SSL Certificate Errors

**Symptom**: `SSLError: certificate verify failed`

**Cause**: Certificate verification issues

**Solutions**:

**Update CA certificates**:
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install ca-certificates

# macOS
brew install openssl
```

**Temporary workaround (not recommended for production)**:
```python
# ⚠ Disable SSL verification (development only!)
response = requests.get(url, headers=headers, verify=False)
```

---

## Debugging Checklist

When troubleshooting issues, follow this checklist:

### API Calls

- [ ] Authorization header includes "Bearer " prefix
- [ ] Organization ID is correct
- [ ] API key has required scopes
- [ ] Request body is valid JSON
- [ ] Content-Type header set to "application/json"
- [ ] Binary data is Base64-encoded
- [ ] Timestamps are in ISO-8601 format (UTC)
- [ ] Rate limits not exceeded
- [ ] Continuation tokens used for pagination

### Device Registration

- [ ] Device key is 32 bytes, Base64-encoded
- [ ] dev_eui is unique
- [ ] All required fields present (name, dev_eui, device_key)
- [ ] Tags are valid key-value pairs

### Webhooks

- [ ] URL is publicly accessible (not localhost)
- [ ] HTTPS with valid certificate
- [ ] Endpoint returns 200 within 30 seconds
- [ ] HTTP-X-HUBBLE-TOKEN header validated
- [ ] Endpoint handles POST requests
- [ ] Endpoint accepts application/json

### Packets

- [ ] Devices are registered
- [ ] Devices have transmitted packets
- [ ] Time range includes packet timestamps
- [ ] Payloads decoded from Base64 correctly

---

## Getting Help

If issues persist after trying these solutions:

1. **Check API Status**: [status.hubble.com](https://status.hubble.com)

2. **Review Logs**: Include `X-Request-ID` from error responses

3. **Contact Support**:
   - Email: support@hubble.com
   - Include: Request ID, timestamp, API endpoint, error message

4. **GitHub Issues**: [github.com/hubble-network/claude-hubble-cloud-api-skill/issues](https://github.com/hubble-network/claude-hubble-cloud-api-skill/issues)

5. **Documentation**: [docs.hubble.com](https://docs.hubble.com)

---

## Additional Resources

- [API_REFERENCE.md](./API_REFERENCE.md) - Complete endpoint documentation
- [WORKFLOWS.md](./WORKFLOWS.md) - Step-by-step implementation guides
- [EXAMPLES.md](./EXAMPLES.md) - Working code examples
- [SKILL.md](./SKILL.md) - Core skill overview
