# Hubble Cloud API Workflows

Step-by-step implementation guides for common Hubble Cloud API tasks. Each workflow includes prerequisites, detailed steps, code examples, error handling, and verification procedures.

## Table of Contents

- [Device Onboarding](#device-onboarding)
- [Packet Streaming Setup](#packet-streaming-setup)
- [Webhook Configuration](#webhook-configuration)
- [API Key Management](#api-key-management)
- [Device Batch Operations](#device-batch-operations)

---

## Device Onboarding

### Overview

Complete workflow for registering new IoT devices with Hubble Network before deployment.

### Prerequisites

- Active Hubble organization account
- API key with `devices:write` scope
- Device hardware with Bluetooth capability
- Device EUI (unique identifier for the hardware)

### Step 1: Generate Device Credentials

Each device needs unique encryption keys for secure communication.

**Python Implementation**:
```python
import os
import base64

def generate_device_credentials(device_name):
    """Generate unique credentials for a new device"""
    # Generate unique device EUI (16 hex characters)
    # In production, use actual hardware EUI
    device_eui = os.urandom(8).hex().upper()

    # Generate 32-byte encryption key
    device_key_binary = os.urandom(32)

    # Base64-encode for API transmission
    device_key_b64 = base64.b64encode(device_key_binary).decode('utf-8')

    return {
        'name': device_name,
        'dev_eui': device_eui,
        'device_key': device_key_b64,
        'device_key_binary': device_key_binary  # Store securely for firmware
    }

# Example usage
credentials = generate_device_credentials("Temperature Sensor #001")
print(f"Device EUI: {credentials['dev_eui']}")
print(f"Device Key (Base64): {credentials['device_key']}")
```

**Important**:
- Store the **binary** device key securely (needed for firmware flashing)
- Never commit device keys to version control
- Use a secure key management system for production

### Step 2: Register Device via API

Send device registration request to Hubble Platform.

**Python Implementation**:
```python
import requests

def register_device(api_token, org_id, credentials, tags=None):
    """Register a new device with Hubble Network"""
    url = f"https://api.hubble.com/api/v2/org/{org_id}/devices"

    headers = {
        "Authorization": f"Bearer {api_token}",
        "Content-Type": "application/json"
    }

    payload = {
        "name": credentials['name'],
        "dev_eui": credentials['dev_eui'],
        "device_key": credentials['device_key']
    }

    # Add custom tags if provided
    if tags:
        payload["tags"] = tags

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 201:
        device = response.json()['device']
        print(f"✓ Device registered successfully: {device['id']}")
        return device
    else:
        print(f"✗ Registration failed: {response.status_code}")
        print(f"  Error: {response.text}")
        return None

# Example usage
device = register_device(
    api_token="your-api-token",
    org_id="7db20441-a04c-49ba-8089-1197b3a66a4b",
    credentials=credentials,
    tags={
        "location": "warehouse-a",
        "sensor_type": "temperature",
        "deployment": "phase-1"
    }
)
```

**curl Alternative**:
```bash
curl -X POST "https://api.hubble.com/api/v2/org/YOUR_ORG_ID/devices" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Temperature Sensor #001",
    "dev_eui": "70B3D54996C4C5A7",
    "device_key": "YmFzZTY0X2VuY29kZWRfa2V5X2hlcmU=",
    "tags": {
      "location": "warehouse-a",
      "sensor_type": "temperature"
    }
  }'
```

### Step 3: Verify Registration

Confirm device appears in your device list.

**Python Implementation**:
```python
def verify_device_registration(api_token, org_id, device_eui):
    """Verify device was registered successfully"""
    url = f"https://api.hubble.com/api/org/{org_id}/devices"

    headers = {"Authorization": f"Bearer {api_token}"}

    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        devices = response.json()['devices']
        matching_device = next(
            (d for d in devices if d['dev_eui'] == device_eui),
            None
        )

        if matching_device:
            print(f"✓ Device verified in system")
            print(f"  Device ID: {matching_device['id']}")
            print(f"  Status: {matching_device['status']}")
            print(f"  Tags: {matching_device.get('tags', {})}")
            return True
        else:
            print(f"✗ Device not found in system")
            return False
    else:
        print(f"✗ Verification failed: {response.status_code}")
        return False

# Example usage
verify_device_registration(api_token, org_id, credentials['dev_eui'])
```

### Step 4: Flash Firmware

Program device with credentials using Hubble Device SDK.

**Conceptual Steps** (refer to Hubble Device SDK documentation):
1. Connect to device via programming interface
2. Load Hubble firmware
3. Write device EUI and encryption key to device memory
4. Set network configuration (region, frequency plan)
5. Verify firmware and credentials
6. Test connectivity with test packet

**Example pseudo-code**:
```python
# Using Hubble Device SDK (pseudo-code)
from hubble_device_sdk import DeviceProgrammer

programmer = DeviceProgrammer()
programmer.connect(port="/dev/ttyUSB0")
programmer.flash_firmware("hubble_firmware_v2.3.bin")
programmer.write_credentials(
    dev_eui=credentials['dev_eui'],
    device_key=credentials['device_key_binary']
)
programmer.set_region("US915")
programmer.verify()
programmer.disconnect()
```

### Step 5: Test Connectivity

Send test packet and verify receipt.

**Device-side** (using Hubble SDK):
```python
# Device sends test packet
device.send_packet(b"Hello Hubble")
```

**Server-side verification**:
```python
import time

def wait_for_first_packet(api_token, org_id, device_id, timeout=300):
    """Wait for device's first packet (up to 5 minutes)"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"
    headers = {"Authorization": f"Bearer {api_token}"}
    params = {"device_id": device_id, "limit": 1}

    start_time = time.time()
    while time.time() - start_time < timeout:
        response = requests.get(url, headers=headers, params=params)
        if response.status_code == 200:
            packets = response.json().get('packets', [])
            if packets:
                print(f"✓ First packet received!")
                print(f"  Payload: {packets[0]['payload']}")
                print(f"  RSSI: {packets[0].get('rssi')}")
                return True

        time.sleep(10)  # Check every 10 seconds

    print(f"✗ No packet received within {timeout}s")
    return False
```

### Error Handling

**400 Bad Request - Invalid device key format**:
```python
# Problem: Device key not Base64-encoded
device_key = b'raw_binary_key'  # ❌ Wrong

# Solution: Base64-encode first
device_key = base64.b64encode(b'raw_binary_key').decode('utf-8')  # ✓ Correct
```

**409 Conflict - Device already exists**:
```python
def register_or_update_device(api_token, org_id, credentials):
    """Register device, or update if already exists"""
    # Try registration first
    device = register_device(api_token, org_id, credentials)

    if device is None:
        # Device might already exist, try to find it
        url = f"https://api.hubble.com/api/org/{org_id}/devices"
        response = requests.get(url, headers={"Authorization": f"Bearer {api_token}"})

        if response.status_code == 200:
            devices = response.json()['devices']
            existing = next((d for d in devices if d['dev_eui'] == credentials['dev_eui']), None)

            if existing:
                print(f"Device already registered: {existing['id']}")
                return existing

    return device
```

### Complete Workflow Script

```python
#!/usr/bin/env python3
"""Complete device onboarding workflow"""

import os
import base64
import requests
import time

API_TOKEN = os.getenv("HUBBLE_API_TOKEN")
ORG_ID = os.getenv("HUBBLE_ORG_ID")

def onboard_device(device_name, tags=None):
    """Complete onboarding workflow for a single device"""
    print(f"\n=== Onboarding: {device_name} ===\n")

    # Step 1: Generate credentials
    print("Step 1: Generating credentials...")
    credentials = generate_device_credentials(device_name)
    print(f"  Generated EUI: {credentials['dev_eui']}")

    # Step 2: Register device
    print("\nStep 2: Registering with Hubble Platform...")
    device = register_device(API_TOKEN, ORG_ID, credentials, tags)
    if not device:
        return False

    # Step 3: Verify registration
    print("\nStep 3: Verifying registration...")
    if not verify_device_registration(API_TOKEN, ORG_ID, credentials['dev_eui']):
        return False

    # Step 4: Flash firmware (manual step)
    print("\nStep 4: Flash firmware to device")
    print(f"  Device EUI: {credentials['dev_eui']}")
    print(f"  Device Key: {credentials['device_key_binary'].hex()}")
    input("  Press Enter after flashing firmware...")

    # Step 5: Wait for first packet
    print("\nStep 5: Waiting for first packet...")
    if wait_for_first_packet(API_TOKEN, ORG_ID, device['id']):
        print("\n✓ Onboarding complete!")
        return True
    else:
        print("\n✗ Onboarding incomplete (no packets received)")
        return False

# Onboard a batch of devices
if __name__ == "__main__":
    devices_to_onboard = [
        ("Temperature Sensor #001", {"location": "warehouse-a", "type": "temp"}),
        ("Temperature Sensor #002", {"location": "warehouse-a", "type": "temp"}),
        ("Humidity Sensor #001", {"location": "warehouse-a", "type": "humidity"})
    ]

    for name, tags in devices_to_onboard:
        onboard_device(name, tags)
        time.sleep(2)  # Rate limiting
```

---

## Packet Streaming Setup

### Overview

Configure efficient packet retrieval using continuation tokens for large datasets.

### Prerequisites

- API key with `packets:read` scope
- At least one device sending packets
- Storage/database for retrieved packets

### Step 1: Initial Packet Query

Retrieve first batch of packets with time range filter.

**Python Implementation**:
```python
import requests
from datetime import datetime, timedelta

def fetch_packet_batch(api_token, org_id, start_time=None, end_time=None, limit=1000):
    """Fetch a single batch of packets"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"

    headers = {"Authorization": f"Bearer {api_token}"}

    params = {"limit": limit}

    # Default to last 7 days if not specified
    if start_time:
        params['start_time'] = start_time.isoformat()
    if end_time:
        params['end_time'] = end_time.isoformat()

    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()

    data = response.json()
    continuation_token = response.headers.get('Continuation-Token')

    return data['packets'], continuation_token

# Example: Get packets from last 24 hours
end_time = datetime.utcnow()
start_time = end_time - timedelta(days=1)

packets, token = fetch_packet_batch(
    api_token="your-token",
    org_id="your-org-id",
    start_time=start_time,
    end_time=end_time
)

print(f"Retrieved {len(packets)} packets")
print(f"Continuation token: {token is not None}")
```

### Step 2: Implement Continuation Token Streaming

Iterate through all pages using continuation tokens.

**Python Implementation**:
```python
def stream_all_packets(api_token, org_id, start_time=None, end_time=None, callback=None):
    """Stream all packets using continuation tokens"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"

    headers = {"Authorization": f"Bearer {api_token}"}

    params = {}
    if start_time:
        params['start_time'] = start_time.isoformat()
    if end_time:
        params['end_time'] = end_time.isoformat()

    total_packets = 0
    page_count = 0

    while True:
        response = requests.get(url, headers=headers, params=params)
        response.raise_for_status()

        data = response.json()
        packets = data['packets']

        # Process packets (call callback if provided)
        if callback:
            callback(packets)

        total_packets += len(packets)
        page_count += 1
        print(f"  Page {page_count}: {len(packets)} packets (total: {total_packets})")

        # Check for continuation token
        continuation_token = response.headers.get('Continuation-Token')
        if not continuation_token:
            break  # No more pages

        # Set token for next request
        headers['Continuation-Token'] = continuation_token
        params = {}  # Clear other params when using continuation token

    return total_packets

# Example usage with callback
def process_packet_batch(packets):
    """Process a batch of packets"""
    for packet in packets:
        # Decode payload
        payload = base64.b64decode(packet['payload'])
        # Store in database
        save_to_database(packet)

total = stream_all_packets(
    api_token="your-token",
    org_id="your-org-id",
    start_time=datetime.utcnow() - timedelta(days=7),
    callback=process_packet_batch
)
print(f"\nTotal packets streamed: {total}")
```

### Step 3: Filter by Device or Tags

Retrieve packets for specific devices or device groups.

**Filter by Single Device**:
```python
def stream_device_packets(api_token, org_id, device_id, start_time, end_time):
    """Stream packets for a specific device"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"

    headers = {"Authorization": f"Bearer {api_token}"}
    params = {
        "device_id": device_id,
        "start_time": start_time.isoformat(),
        "end_time": end_time.isoformat()
    }

    packets = []

    while True:
        response = requests.get(url, headers=headers, params=params)
        response.raise_for_status()

        data = response.json()
        packets.extend(data['packets'])

        continuation_token = response.headers.get('Continuation-Token')
        if not continuation_token:
            break

        headers['Continuation-Token'] = continuation_token
        params = {}

    return packets
```

**Filter by Platform Tag**:
```python
def stream_packets_by_platform(api_token, org_id, platform, start_time, end_time):
    """Stream packets for devices on specific platform (e.g., LoRaWAN)"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"

    headers = {"Authorization": f"Bearer {api_token}"}
    params = {
        "platform_tag": f"hubnet.platform={platform}",
        "start_time": start_time.isoformat(),
        "end_time": end_time.isoformat()
    }

    # Same streaming logic as above
    # ...
```

### Step 4: Decode and Process Packets

Extract meaningful data from Base64-encoded payloads.

**Python Implementation**:
```python
import base64
import struct

def decode_packet_payload(packet):
    """Decode packet payload based on your data format"""
    # Payload is Base64-encoded
    payload_b64 = packet['payload']
    payload_binary = base64.b64decode(payload_b64)

    # Example: Decode temperature/humidity sensor data
    # Format: 2 bytes temperature (int16), 2 bytes humidity (uint16)
    if len(payload_binary) >= 4:
        temperature_raw, humidity_raw = struct.unpack('>hH', payload_binary[:4])

        # Convert to actual values (example scaling)
        temperature_c = temperature_raw / 100.0
        humidity_percent = humidity_raw / 100.0

        return {
            'device_id': packet['device_id'],
            'temperature_c': temperature_c,
            'humidity_percent': humidity_percent,
            'timestamp': packet['received_at'],
            'rssi': packet.get('rssi'),
            'snr': packet.get('snr')
        }

    return None

# Process and store packets
def process_and_store(packets):
    """Decode and store packets in database"""
    for packet in packets:
        decoded = decode_packet_payload(packet)
        if decoded:
            print(f"Device {decoded['device_id']}: "
                  f"{decoded['temperature_c']:.1f}°C, "
                  f"{decoded['humidity_percent']:.1f}%")
            # Save to database
            db.insert('sensor_readings', decoded)
```

### Step 5: Schedule Regular Polling

Set up periodic packet retrieval for ongoing monitoring.

**Python Implementation using APScheduler**:
```python
from apscheduler.schedulers.blocking import BlockingScheduler
from datetime import datetime, timedelta

class PacketPoller:
    def __init__(self, api_token, org_id):
        self.api_token = api_token
        self.org_id = org_id
        self.last_poll_time = datetime.utcnow()

    def poll_new_packets(self):
        """Poll for packets since last check"""
        end_time = datetime.utcnow()
        start_time = self.last_poll_time

        print(f"\nPolling packets from {start_time} to {end_time}")

        total = stream_all_packets(
            self.api_token,
            self.org_id,
            start_time=start_time,
            end_time=end_time,
            callback=process_and_store
        )

        print(f"Processed {total} new packets")
        self.last_poll_time = end_time

# Set up scheduler
poller = PacketPoller("your-token", "your-org-id")

scheduler = BlockingScheduler()
scheduler.add_job(poller.poll_new_packets, 'interval', minutes=5)

print("Starting packet poller (every 5 minutes)...")
scheduler.start()
```

### Best Practices

**Rate Limiting**:
```python
import time

def stream_with_rate_limiting(api_token, org_id, start_time, end_time):
    """Stream packets with rate limit handling"""
    url = f"https://api.hubble.com/api/org/{org_id}/packets"
    headers = {"Authorization": f"Bearer {api_token}"}
    params = {"start_time": start_time.isoformat(), "end_time": end_time.isoformat()}

    max_retries = 5
    retry_count = 0

    while True:
        try:
            response = requests.get(url, headers=headers, params=params)

            if response.status_code == 429:
                # Rate limited
                retry_after = int(response.headers.get('Retry-After', 60))
                print(f"  Rate limited. Waiting {retry_after}s...")
                time.sleep(retry_after)
                retry_count += 1
                if retry_count >= max_retries:
                    print("  Max retries exceeded")
                    break
                continue

            response.raise_for_status()
            data = response.json()

            # Process packets
            process_packet_batch(data['packets'])

            # Check continuation
            continuation_token = response.headers.get('Continuation-Token')
            if not continuation_token:
                break

            headers['Continuation-Token'] = continuation_token
            params = {}
            retry_count = 0  # Reset on success

            # Small delay to avoid hitting rate limits
            time.sleep(0.5)

        except requests.exceptions.RequestException as e:
            print(f"  Error: {e}")
            retry_count += 1
            if retry_count >= max_retries:
                break
            time.sleep(2 ** retry_count)  # Exponential backoff
```

---

## Webhook Configuration

### Overview

Set up real-time packet delivery to your HTTP endpoint via webhooks.

### Prerequisites

- API key with `webhooks:write` scope
- Publicly accessible HTTPS endpoint
- Web server capable of handling POST requests

### Step 1: Implement Webhook Endpoint

Create HTTP endpoint to receive packet batches.

**Flask Example**:
```python
from flask import Flask, request, abort
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-secret-token"

@app.route('/webhook/hubble', methods=['POST'])
def hubble_webhook():
    # Verify request is from Hubble
    token = request.headers.get('HTTP-X-HUBBLE-TOKEN')
    if token != WEBHOOK_SECRET:
        abort(403, "Invalid webhook token")

    # Parse packet data
    data = request.json
    packets = data.get('packets', [])

    print(f"Received {len(packets)} packets")

    # Process each packet
    for packet in packets:
        process_packet(packet)

    # Return 200 to acknowledge receipt
    return '', 200

def process_packet(packet):
    """Process a single packet"""
    device_id = packet['device_id']
    payload = base64.b64decode(packet['payload'])
    timestamp = packet['received_at']

    # Your processing logic
    print(f"  Device {device_id}: {len(payload)} bytes at {timestamp}")
    # Store in database, trigger alerts, etc.

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**FastAPI Example**:
```python
from fastapi import FastAPI, Header, HTTPException, Request

app = FastAPI()
WEBHOOK_SECRET = "your-secret-token"

@app.post("/webhook/hubble")
async def hubble_webhook(
    request: Request,
    http_x_hubble_token: str = Header(None)
):
    # Verify token
    if http_x_hubble_token != WEBHOOK_SECRET:
        raise HTTPException(status_code=403, detail="Invalid token")

    # Parse body
    data = await request.json()
    packets = data.get('packets', [])

    # Process packets asynchronously
    for packet in packets:
        await process_packet_async(packet)

    return {"status": "ok"}
```

### Step 2: Register Webhook with Hubble

Register your endpoint to start receiving packets.

**Python Implementation**:
```python
def register_webhook(api_token, org_id, webhook_url, secret, batch_size=100):
    """Register webhook endpoint with Hubble"""
    url = f"https://api.hubble.com/api/org/{org_id}/webhooks"

    headers = {
        "Authorization": f"Bearer {api_token}",
        "Content-Type": "application/json"
    }

    payload = {
        "url": webhook_url,
        "name": "Production Webhook",
        "max_batch_size": batch_size,
        "secret": secret
    }

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 201:
        webhook = response.json()['webhook']
        print(f"✓ Webhook registered successfully")
        print(f"  Webhook ID: {webhook['webhook_id']}")
        print(f"  URL: {webhook['url']}")
        print(f"  Batch size: {webhook['max_batch_size']}")
        return webhook
    else:
        print(f"✗ Webhook registration failed: {response.status_code}")
        print(f"  Error: {response.text}")
        return None

# Example usage
webhook = register_webhook(
    api_token="your-token",
    org_id="your-org-id",
    webhook_url="https://your-domain.com/webhook/hubble",
    secret="your-secret-token",
    batch_size=100
)
```

**curl Alternative**:
```bash
curl -X POST "https://api.hubble.com/api/org/YOUR_ORG_ID/webhooks" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/webhook/hubble",
    "name": "Production Webhook",
    "max_batch_size": 100,
    "secret": "your-secret-token"
  }'
```

### Step 3: Test Webhook Delivery

Send test packets and verify webhook receives them.

**Manual Test**:
```bash
# Use your test device to send a packet
# Or use Hubble Dashboard to send test packet
```

**Monitor Webhook**:
```python
import time

def monitor_webhook_metrics(api_token, org_id, webhook_id, duration=300):
    """Monitor webhook delivery for specified duration (seconds)"""
    url = f"https://api.hubble.com/api/org/{org_id}/webhook_metrics"
    headers = {"Authorization": f"Bearer {api_token}"}
    params = {"webhook_id": webhook_id, "days_back": 1}

    print(f"Monitoring webhook for {duration}s...")
    start_time = time.time()

    while time.time() - start_time < duration:
        response = requests.get(url, headers=headers, params=params)
        if response.status_code == 200:
            data = response.json()
            if data['metrics']:
                latest = data['metrics'][-1]
                print(f"  Deliveries: {latest['deliveries_attempted']}, "
                      f"Success: {latest['deliveries_successful']}, "
                      f"Failed: {latest['deliveries_failed']}")

        time.sleep(30)  # Check every 30 seconds

monitor_webhook_metrics("your-token", "your-org-id", webhook['webhook_id'])
```

### Step 4: Implement Error Handling

Handle webhook delivery failures gracefully.

**Retry Logic on Server Side**:
```python
@app.route('/webhook/hubble', methods=['POST'])
def hubble_webhook():
    try:
        # Verify token
        token = request.headers.get('HTTP-X-HUBBLE-TOKEN')
        if token != WEBHOOK_SECRET:
            abort(403)

        # Parse data
        data = request.json
        packets = data.get('packets', [])

        # Process with error handling
        for packet in packets:
            try:
                process_packet(packet)
            except Exception as e:
                # Log error but continue processing other packets
                logger.error(f"Error processing packet {packet['packet_id']}: {e}")

        return '', 200

    except Exception as e:
        # Log error
        logger.error(f"Webhook error: {e}")
        # Return 500 to trigger Hubble retry
        return '', 500
```

**Dead Letter Queue for Failed Packets**:
```python
import redis

redis_client = redis.Redis()

def process_packet(packet):
    """Process packet with DLQ fallback"""
    try:
        # Main processing
        decode_and_store(packet)
    except Exception as e:
        # Failed to process, add to DLQ
        redis_client.lpush('packet_dlq', json.dumps(packet))
        logger.error(f"Packet {packet['packet_id']} moved to DLQ: {e}")

# Background worker to retry DLQ packets
def process_dlq():
    """Retry failed packets from DLQ"""
    while True:
        packet_json = redis_client.rpop('packet_dlq')
        if packet_json:
            packet = json.loads(packet_json)
            try:
                decode_and_store(packet)
                logger.info(f"DLQ packet {packet['packet_id']} processed")
            except Exception as e:
                # Re-add to DLQ
                redis_client.lpush('packet_dlq', packet_json)
                time.sleep(60)  # Back off before retry
        else:
            time.sleep(10)
```

### Step 5: Monitor Webhook Health

Set up continuous monitoring and alerting.

**Health Check Script**:
```python
def check_webhook_health(api_token, org_id, webhook_id, alert_threshold=95.0):
    """Check webhook delivery success rate"""
    url = f"https://api.hubble.com/api/org/{org_id}/webhook_metrics"
    headers = {"Authorization": f"Bearer {api_token}"}
    params = {"webhook_id": webhook_id, "days_back": 1}

    response = requests.get(url, headers=headers, params=params)
    if response.status_code != 200:
        send_alert(f"Failed to fetch webhook metrics: {response.status_code}")
        return False

    data = response.json()
    summary = data.get('summary', {})

    success_rate = summary.get('success_rate', 0.0)
    total_deliveries = summary.get('total_deliveries', 0)

    print(f"Webhook Health Check:")
    print(f"  Total deliveries: {total_deliveries}")
    print(f"  Success rate: {success_rate:.2f}%")

    if success_rate < alert_threshold:
        send_alert(f"Webhook success rate low: {success_rate:.2f}%")
        return False

    return True

# Run health check periodically
scheduler = BlockingScheduler()
scheduler.add_job(
    lambda: check_webhook_health("token", "org-id", "webhook-id"),
    'interval',
    minutes=15
)
scheduler.start()
```

### Best Practices

- **Use HTTPS**: Webhooks must use secure HTTPS endpoints
- **Validate tokens**: Always check `HTTP-X-HUBBLE-TOKEN` header
- **Return 200 quickly**: Process packets asynchronously, return 200 ASAP
- **Handle retries**: Implement idempotency (check packet IDs to avoid duplicates)
- **Monitor metrics**: Track delivery success rates
- **Set appropriate batch size**: Larger batches = fewer requests, but larger payloads
- **Timeout handling**: Respond within 30 seconds or Hubble will retry

---

## API Key Management

### Overview

Securely manage API keys including creation, rotation, and revocation.

### Step 1: Create API Key with Minimal Scopes

Generate key with only required permissions.

**Python Implementation**:
```python
def create_api_key(api_token, org_id, name, scopes):
    """Create new API key with specific scopes"""
    url = f"https://api.hubble.com/api/org/{org_id}/key"

    headers = {
        "Authorization": f"Bearer {api_token}",
        "Content-Type": "application/json"
    }

    payload = {
        "name": name,
        "scopes": scopes
    }

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 201:
        key_data = response.json()
        print(f"✓ API key created: {key_data['key_id']}")
        print(f"  Token: {key_data['token']}")
        print(f"  WARNING: Save this token now, it won't be shown again!")
        return key_data
    else:
        print(f"✗ Key creation failed: {response.status_code}")
        return None

# Example: Create read-only key
readonly_key = create_api_key(
    api_token="admin-token",
    org_id="your-org-id",
    name="Analytics Read-Only Key",
    scopes=["devices:read", "packets:read", "metrics:read"]
)

# Example: Create webhook management key
webhook_key = create_api_key(
    api_token="admin-token",
    org_id="your-org-id",
    name="Webhook Manager",
    scopes=["webhooks:read", "webhooks:write"]
)
```

### Step 2: Store Keys Securely

Never commit keys to version control.

**Environment Variables (.env file)**:
```bash
# .env (add to .gitignore!)
HUBBLE_API_TOKEN=your-secret-token-here
HUBBLE_ORG_ID=7db20441-a04c-49ba-8089-1197b3a66a4b
```

**Load in Python**:
```python
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')
```

**AWS Secrets Manager**:
```python
import boto3
import json

def get_hubble_credentials():
    """Retrieve credentials from AWS Secrets Manager"""
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId='hubble/api-credentials')
    secret = json.loads(response['SecretString'])

    return secret['api_token'], secret['org_id']

API_TOKEN, ORG_ID = get_hubble_credentials()
```

### Step 3: Implement Key Rotation

Rotate keys regularly without downtime.

**Rotation Workflow**:
```python
def rotate_api_key(old_token, org_id, key_name):
    """Rotate API key with zero downtime"""
    print(f"Rotating key: {key_name}")

    # Step 1: List current keys to get scopes
    url = f"https://api.hubble.com/api/org/{org_id}/key"
    headers = {"Authorization": f"Bearer {old_token}"}

    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        print(f"✗ Failed to list keys")
        return None

    keys = response.json()['keys']
    old_key = next((k for k in keys if k['name'] == key_name), None)

    if not old_key:
        print(f"✗ Key '{key_name}' not found")
        return None

    # Step 2: Create new key with same scopes
    new_key = create_api_key(old_token, org_id, key_name, old_key['scopes'])
    if not new_key:
        return None

    print(f"\n✓ New key created")
    print(f"  New Token: {new_key['token']}")
    print(f"\nAction required:")
    print(f"  1. Update applications with new token")
    print(f"  2. Verify new token works")
    print(f"  3. Run: delete_api_key('{old_key['key_id']}')")

    return new_key

# Example usage
rotate_api_key("current-token", "org-id", "Production API Key")
```

### Step 4: Revoke Compromised Keys

Immediately revoke keys if compromised.

**Python Implementation**:
```python
def revoke_api_key(api_token, org_id, key_id):
    """Immediately revoke an API key"""
    url = f"https://api.hubble.com/api/org/{org_id}/key/{key_id}"
    headers = {"Authorization": f"Bearer {api_token}"}

    response = requests.delete(url, headers=headers)

    if response.status_code == 200:
        print(f"✓ Key {key_id} revoked")
        return True
    else:
        print(f"✗ Revocation failed: {response.status_code}")
        return False

# Emergency revocation
revoke_api_key("admin-token", "org-id", "compromised-key-id")
```

### Step 5: Audit Key Usage

Monitor API key usage and identify unused keys.

**Python Implementation**:
```python
def audit_api_keys(api_token, org_id):
    """Audit all API keys and their usage"""
    url = f"https://api.hubble.com/api/org/{org_id}/key"
    headers = {"Authorization": f"Bearer {api_token}"}

    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        print(f"✗ Failed to fetch keys")
        return

    keys = response.json()['keys']

    print("\n=== API Key Audit ===\n")
    for key in keys:
        print(f"Key: {key['name']}")
        print(f"  ID: {key['key_id']}")
        print(f"  Created: {key['created_at']}")
        print(f"  Last Used: {key.get('last_used_at', 'Never')}")
        print(f"  Scopes: {', '.join(key['scopes'])}")

        # Flag unused keys
        if not key.get('last_used_at'):
            print(f"  ⚠ WARNING: Key never used, consider revoking")

        print()

audit_api_keys("admin-token", "org-id")
```

---

## Device Batch Operations

### Overview

Efficiently update multiple devices in a single API request (up to 1,000 devices).

### Step 1: List Devices to Update

Identify target devices using filters.

**Python Implementation**:
```python
def find_devices_by_tag(api_token, org_id, tag_key, tag_value):
    """Find all devices with a specific tag"""
    url = f"https://api.hubble.com/api/org/{org_id}/devices"
    headers = {"Authorization": f"Bearer {api_token}"}
    params = {"tag": f"{tag_key}={tag_value}"}

    response = requests.get(url, headers=headers, params=params)
    if response.status_code != 200:
        return []

    return response.json()['devices']

# Example: Find all devices in warehouse-a
devices = find_devices_by_tag("token", "org-id", "location", "warehouse-a")
print(f"Found {len(devices)} devices")
```

### Step 2: Prepare Batch Update

Create update payload for multiple devices.

**Python Implementation**:
```python
def prepare_batch_update(devices, new_tags=None, name_prefix=None):
    """Prepare batch update payload"""
    updates = []

    for device in devices:
        update = {"device_id": device['id']}

        # Add new tags (merge with existing)
        if new_tags:
            merged_tags = {**device.get('tags', {}), **new_tags}
            update['tags'] = merged_tags

        # Update name with prefix
        if name_prefix:
            update['name'] = f"{name_prefix}{device['name']}"

        updates.append(update)

    return updates

# Example: Add deployment tag to all devices
updates = prepare_batch_update(
    devices,
    new_tags={"deployment": "phase-2", "updated_at": "2024-01-21"}
)
```

### Step 3: Execute Batch Update

Send batch update request (max 1,000 devices).

**Python Implementation**:
```python
def batch_update_devices(api_token, org_id, updates):
    """Update multiple devices in a single request"""
    if len(updates) > 1000:
        print(f"⚠ Warning: {len(updates)} devices exceed 1,000 limit. Splitting...")
        return batch_update_devices_chunked(api_token, org_id, updates)

    url = f"https://api.hubble.com/api/org/{org_id}/devices"
    headers = {
        "Authorization": f"Bearer {api_token}",
        "Content-Type": "application/json"
    }

    payload = {"updates": updates}

    response = requests.patch(url, json=payload, headers=headers)

    if response.status_code == 200:
        results = response.json()['results']
        successful = sum(1 for r in results if r['status'] == 'success')
        failed = sum(1 for r in results if r['status'] == 'error')

        print(f"✓ Batch update complete")
        print(f"  Successful: {successful}")
        print(f"  Failed: {failed}")

        # Log failures
        for result in results:
            if result['status'] == 'error':
                print(f"  ✗ Device {result['device_id']}: {result['error']}")

        return results
    else:
        print(f"✗ Batch update failed: {response.status_code}")
        return None

# Execute update
results = batch_update_devices("token", "org-id", updates)
```

### Step 4: Handle Large Batches (>1,000 devices)

Split into chunks for batches exceeding 1,000 devices.

**Python Implementation**:
```python
def batch_update_devices_chunked(api_token, org_id, updates, chunk_size=1000):
    """Update devices in chunks of 1,000"""
    all_results = []

    for i in range(0, len(updates), chunk_size):
        chunk = updates[i:i + chunk_size]
        print(f"\nProcessing chunk {i // chunk_size + 1} ({len(chunk)} devices)")

        results = batch_update_devices(api_token, org_id, chunk)
        if results:
            all_results.extend(results)

        # Rate limiting: wait between chunks
        if i + chunk_size < len(updates):
            print("  Waiting 2s before next chunk...")
            time.sleep(2)

    return all_results

# Example: Update 5,000 devices
large_updates = prepare_batch_update(all_devices, new_tags={"status": "active"})
results = batch_update_devices_chunked("token", "org-id", large_updates)
```

### Step 5: Verify Updates

Confirm updates were applied correctly.

**Python Implementation**:
```python
def verify_batch_update(api_token, org_id, device_ids, expected_tag_key, expected_tag_value):
    """Verify devices have expected tag"""
    url = f"https://api.hubble.com/api/org/{org_id}/devices"
    headers = {"Authorization": f"Bearer {api_token}"}

    verified = 0
    failed = 0

    for device_id in device_ids:
        response = requests.get(f"{url}/{device_id}", headers=headers)
        if response.status_code == 200:
            device = response.json()['device']
            tags = device.get('tags', {})

            if tags.get(expected_tag_key) == expected_tag_value:
                verified += 1
            else:
                failed += 1
                print(f"✗ Device {device_id} missing expected tag")

    print(f"\nVerification complete:")
    print(f"  Verified: {verified}")
    print(f"  Failed: {failed}")

    return failed == 0

# Verify deployment tag was added
device_ids = [d['id'] for d in devices]
verify_batch_update("token", "org-id", device_ids, "deployment", "phase-2")
```

### Complete Batch Update Script

```python
#!/usr/bin/env python3
"""Complete batch device update workflow"""

def batch_update_workflow(api_token, org_id, filter_tag, new_tags):
    """Complete workflow for batch updating devices"""
    print(f"\n=== Batch Update Workflow ===\n")

    # Step 1: Find target devices
    print(f"Step 1: Finding devices with tag '{filter_tag}'...")
    key, value = filter_tag.split('=')
    devices = find_devices_by_tag(api_token, org_id, key, value)
    print(f"  Found {len(devices)} devices")

    if not devices:
        print("  No devices to update")
        return

    # Step 2: Prepare updates
    print(f"\nStep 2: Preparing batch update...")
    updates = prepare_batch_update(devices, new_tags=new_tags)
    print(f"  Prepared {len(updates)} updates")

    # Step 3: Execute batch update
    print(f"\nStep 3: Executing batch update...")
    results = batch_update_devices_chunked(api_token, org_id, updates)

    # Step 4: Verify updates
    print(f"\nStep 4: Verifying updates...")
    device_ids = [d['id'] for d in devices]
    verify_key = list(new_tags.keys())[0]
    verify_value = new_tags[verify_key]
    success = verify_batch_update(api_token, org_id, device_ids, verify_key, verify_value)

    if success:
        print(f"\n✓ Batch update workflow complete!")
    else:
        print(f"\n⚠ Batch update completed with errors")

# Example usage
if __name__ == "__main__":
    batch_update_workflow(
        api_token="your-token",
        org_id="your-org-id",
        filter_tag="location=warehouse-a",
        new_tags={
            "deployment": "phase-2",
            "updated_at": "2024-01-21",
            "status": "production"
        }
    )
```

---

## Summary

These workflows cover the most common Hubble Cloud API integration scenarios:

1. **Device Onboarding**: Complete process from credential generation to connectivity testing
2. **Packet Streaming**: Efficient retrieval of large packet datasets with continuation tokens
3. **Webhook Configuration**: Real-time packet delivery with proper security and error handling
4. **API Key Management**: Secure key creation, rotation, and auditing
5. **Device Batch Operations**: Efficient mass updates for fleet management

For additional examples and code snippets, see [EXAMPLES.md](./EXAMPLES.md).
For troubleshooting common issues, see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).
