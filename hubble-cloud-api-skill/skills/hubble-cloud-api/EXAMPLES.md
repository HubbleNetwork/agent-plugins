# Hubble Cloud API - Code Examples

Complete, runnable code examples for common Hubble Cloud API operations. All examples include error handling and best practices.

## Table of Contents

- [Setup and Authentication](#setup-and-authentication)
- [Device Management Examples](#device-management-examples)
- [Packet Retrieval Examples](#packet-retrieval-examples)
- [Webhook Examples](#webhook-examples)
- [Metrics Examples](#metrics-examples)
- [Complete Applications](#complete-applications)

---

## Setup and Authentication

### Environment Setup

**requirements.txt**:
```
requests==2.31.0
python-dotenv==1.0.0
```

**.env file** (never commit!):
```bash
HUBBLE_API_TOKEN=your-jwt-token-here
HUBBLE_ORG_ID=7db20441-a04c-49ba-8089-1197b3a66a4b
```

### Basic Authentication Example

```python
#!/usr/bin/env python3
"""Basic Hubble API authentication example"""

import os
import requests
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')
BASE_URL = "https://api.hubble.com"

def make_hubble_request(method, endpoint, **kwargs):
    """Make authenticated request to Hubble API"""
    url = f"{BASE_URL}{endpoint}"

    headers = kwargs.pop('headers', {})
    headers['Authorization'] = f"Bearer {API_TOKEN}"

    response = requests.request(method, url, headers=headers, **kwargs)

    # Handle common errors
    if response.status_code == 401:
        raise Exception("Authentication failed - check API token")
    elif response.status_code == 403:
        raise Exception("Forbidden - insufficient permissions")
    elif response.status_code == 429:
        retry_after = response.headers.get('Retry-After', 60)
        raise Exception(f"Rate limited - retry after {retry_after}s")

    response.raise_for_status()
    return response

# Test authentication
try:
    response = make_hubble_request('GET', f'/api/org/{ORG_ID}/check')
    print("✓ Authentication successful")
    print(f"Token valid: {response.json()['valid']}")
except Exception as e:
    print(f"✗ Authentication failed: {e}")
```

---

## Device Management Examples

### Example 1: Register Single Device

```python
#!/usr/bin/env python3
"""Register a new device with Hubble Network"""

import os
import base64
import requests
from dotenv import load_dotenv

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def register_device(name, dev_eui, tags=None):
    """Register a new device"""
    # Generate device key (32 bytes, Base64-encoded)
    device_key_binary = os.urandom(32)
    device_key_b64 = base64.b64encode(device_key_binary).decode('utf-8')

    url = f"https://api.hubble.com/api/v2/org/{ORG_ID}/devices"
    headers = {
        "Authorization": f"Bearer {API_TOKEN}",
        "Content-Type": "application/json"
    }

    payload = {
        "name": name,
        "dev_eui": dev_eui,
        "device_key": device_key_b64
    }

    if tags:
        payload["tags"] = tags

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 201:
        device = response.json()['device']
        print(f"✓ Device registered: {device['id']}")
        print(f"  Name: {device['name']}")
        print(f"  EUI: {device['dev_eui']}")

        # Save binary key for firmware (in production, use secure storage)
        with open(f"device_{dev_eui}_key.bin", 'wb') as f:
            f.write(device_key_binary)
        print(f"  Key saved to: device_{dev_eui}_key.bin")

        return device
    else:
        print(f"✗ Registration failed: {response.status_code}")
        print(f"  Error: {response.text}")
        return None

# Example usage
if __name__ == "__main__":
    device = register_device(
        name="Temperature Sensor #001",
        dev_eui="70B3D54996C4C5A7",
        tags={
            "location": "warehouse-a",
            "sensor_type": "temperature",
            "deployment": "phase-1"
        }
    )
```

### Example 2: List and Filter Devices

```python
#!/usr/bin/env python3
"""List and filter devices"""

import requests
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def list_devices(platform_tag=None, custom_tag=None, name_pattern=None):
    """List devices with optional filters"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/devices"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    params = {}
    if platform_tag:
        params['platform_tag'] = platform_tag
    if custom_tag:
        params['tag'] = custom_tag
    if name_pattern:
        params['name_pattern'] = name_pattern

    response = requests.get(url, headers=headers, params=params)

    if response.status_code == 200:
        data = response.json()
        devices = data['devices']

        print(f"Found {len(devices)} devices (total: {data['total_count']})")

        for device in devices:
            print(f"\n  Device: {device['name']}")
            print(f"    ID: {device['id']}")
            print(f"    EUI: {device['dev_eui']}")
            print(f"    Status: {device['status']}")
            print(f"    Tags: {device.get('tags', {})}")

        return devices
    else:
        print(f"✗ Failed to list devices: {response.status_code}")
        return []

# Example usage
if __name__ == "__main__":
    # List all devices
    all_devices = list_devices()

    # Filter by platform
    lorawan_devices = list_devices(platform_tag="hubnet.platform=LoRaWAN")

    # Filter by custom tag
    warehouse_devices = list_devices(custom_tag="location=warehouse-a")

    # Search by name
    temp_sensors = list_devices(name_pattern="Temperature")
```

### Example 3: Batch Update Devices

```python
#!/usr/bin/env python3
"""Batch update multiple devices"""

import requests
import time
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def batch_update_devices(device_updates):
    """Update multiple devices (max 1000 per request)"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/devices"
    headers = {
        "Authorization": f"Bearer {API_TOKEN}",
        "Content-Type": "application/json"
    }

    # Split into chunks of 1000
    chunk_size = 1000
    all_results = []

    for i in range(0, len(device_updates), chunk_size):
        chunk = device_updates[i:i + chunk_size]

        print(f"Processing batch {i // chunk_size + 1} ({len(chunk)} devices)...")

        payload = {"updates": chunk}
        response = requests.patch(url, json=payload, headers=headers)

        if response.status_code == 200:
            results = response.json()['results']
            all_results.extend(results)

            successful = sum(1 for r in results if r['status'] == 'success')
            failed = sum(1 for r in results if r['status'] == 'error')

            print(f"  Successful: {successful}")
            print(f"  Failed: {failed}")

            # Show failures
            for result in results:
                if result['status'] == 'error':
                    print(f"    ✗ {result['device_id']}: {result['error']}")
        else:
            print(f"  ✗ Batch failed: {response.status_code}")

        # Rate limiting delay
        if i + chunk_size < len(device_updates):
            time.sleep(1)

    return all_results

# Example usage
if __name__ == "__main__":
    # Get devices to update
    response = requests.get(
        f"https://api.hubble.com/api/org/{ORG_ID}/devices",
        headers={"Authorization": f"Bearer {API_TOKEN}"},
        params={"tag": "location=warehouse-a"}
    )

    devices = response.json()['devices']

    # Prepare updates (add deployment tag to all)
    updates = [
        {
            "device_id": device['id'],
            "tags": {
                **device.get('tags', {}),
                "deployment": "phase-2",
                "updated_at": "2024-01-21"
            }
        }
        for device in devices
    ]

    print(f"Updating {len(updates)} devices...")
    results = batch_update_devices(updates)
```

---

## Packet Retrieval Examples

### Example 4: Stream Packets with Continuation Tokens

```python
#!/usr/bin/env python3
"""Stream all packets using continuation tokens"""

import requests
from datetime import datetime, timedelta
from dotenv import load_dotenv
import os
import base64

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def stream_packets(start_time, end_time, device_id=None, callback=None):
    """Stream packets with continuation token pagination"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/packets"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    params = {
        "start_time": start_time.isoformat(),
        "end_time": end_time.isoformat(),
        "limit": 1000
    }

    if device_id:
        params['device_id'] = device_id

    total_packets = 0
    page_count = 0

    while True:
        response = requests.get(url, headers=headers, params=params)

        if response.status_code != 200:
            print(f"✗ Error: {response.status_code} - {response.text}")
            break

        data = response.json()
        packets = data['packets']

        # Process packets
        if callback:
            callback(packets)

        total_packets += len(packets)
        page_count += 1

        print(f"  Page {page_count}: {len(packets)} packets (total: {total_packets})")

        # Check for more pages
        continuation_token = response.headers.get('Continuation-Token')
        if not continuation_token:
            print(f"\n✓ Streaming complete: {total_packets} packets")
            break

        # Set up next request
        headers['Continuation-Token'] = continuation_token
        params = {}  # Clear params when using continuation token

    return total_packets

def process_packet_batch(packets):
    """Process a batch of packets"""
    for packet in packets:
        # Decode Base64 payload
        try:
            payload = base64.b64decode(packet['payload'])
            print(f"    Device {packet['device_id']}: {len(payload)} bytes")

            # Your processing logic here
            # e.g., parse payload, save to database, trigger alerts

        except Exception as e:
            print(f"    ✗ Error processing packet {packet['packet_id']}: {e}")

# Example usage
if __name__ == "__main__":
    # Stream packets from last 24 hours
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=1)

    print(f"Streaming packets from {start_time} to {end_time}\n")

    total = stream_packets(
        start_time=start_time,
        end_time=end_time,
        callback=process_packet_batch
    )
```

### Example 5: Decode Temperature/Humidity Sensor Data

```python
#!/usr/bin/env python3
"""Decode sensor packet payloads"""

import base64
import struct
import requests
from datetime import datetime, timedelta
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def decode_sensor_payload(payload_b64):
    """Decode temperature/humidity sensor payload"""
    # Decode from Base64
    payload = base64.b64decode(payload_b64)

    if len(payload) < 4:
        return None

    # Parse binary data (example format)
    # Bytes 0-1: Temperature (int16, big-endian, in centidegrees C)
    # Bytes 2-3: Humidity (uint16, big-endian, in centipercent)
    temperature_raw, humidity_raw = struct.unpack('>hH', payload[:4])

    # Convert to actual values
    temperature_c = temperature_raw / 100.0
    humidity_percent = humidity_raw / 100.0

    return {
        'temperature_c': temperature_c,
        'temperature_f': temperature_c * 9/5 + 32,
        'humidity_percent': humidity_percent
    }

def fetch_and_decode_packets(device_id, hours=1):
    """Fetch recent packets and decode sensor data"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/packets"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=hours)

    params = {
        "device_id": device_id,
        "start_time": start_time.isoformat(),
        "end_time": end_time.isoformat()
    }

    response = requests.get(url, headers=headers, params=params)

    if response.status_code != 200:
        print(f"✗ Failed to fetch packets: {response.status_code}")
        return []

    packets = response.json()['packets']
    decoded_data = []

    for packet in packets:
        sensor_data = decode_sensor_payload(packet['payload'])

        if sensor_data:
            decoded_data.append({
                'timestamp': packet['received_at'],
                'device_id': packet['device_id'],
                'rssi': packet.get('rssi'),
                'snr': packet.get('snr'),
                **sensor_data
            })

            print(f"{packet['received_at']}: "
                  f"{sensor_data['temperature_c']:.1f}°C / "
                  f"{sensor_data['temperature_f']:.1f}°F, "
                  f"{sensor_data['humidity_percent']:.1f}% RH "
                  f"(RSSI: {packet.get('rssi')})")

    return decoded_data

# Example usage
if __name__ == "__main__":
    DEVICE_ID = "your-device-id"

    print(f"Fetching packets for device {DEVICE_ID}...\n")
    data = fetch_and_decode_packets(DEVICE_ID, hours=24)

    if data:
        # Calculate statistics
        temps = [d['temperature_c'] for d in data]
        humidities = [d['humidity_percent'] for d in data]

        print(f"\nStatistics ({len(data)} readings):")
        print(f"  Temperature: {min(temps):.1f}°C to {max(temps):.1f}°C "
              f"(avg: {sum(temps)/len(temps):.1f}°C)")
        print(f"  Humidity: {min(humidities):.1f}% to {max(humidities):.1f}% "
              f"(avg: {sum(humidities)/len(humidities):.1f}%)")
```

---

## Webhook Examples

### Example 6: Flask Webhook Server

```python
#!/usr/bin/env python3
"""Flask webhook server for receiving Hubble packets"""

from flask import Flask, request, abort
import base64
import json
from datetime import datetime
from dotenv import load_dotenv
import os

load_dotenv()

app = Flask(__name__)
WEBHOOK_SECRET = os.getenv('WEBHOOK_SECRET', 'your-secret-token')

# In production, use database
packet_log = []

@app.route('/webhook/hubble', methods=['POST'])
def hubble_webhook():
    """Receive packet batches from Hubble"""
    # Validate token
    token = request.headers.get('HTTP-X-HUBBLE-TOKEN')
    if token != WEBHOOK_SECRET:
        print(f"✗ Invalid webhook token received")
        abort(403, "Invalid token")

    # Parse packet data
    data = request.json
    packets = data.get('packets', [])

    print(f"\n[{datetime.utcnow().isoformat()}] Received {len(packets)} packets")

    # Process each packet
    for packet in packets:
        try:
            process_packet(packet)
        except Exception as e:
            print(f"  ✗ Error processing packet {packet['packet_id']}: {e}")

    # Acknowledge receipt
    return '', 200

def process_packet(packet):
    """Process a single packet"""
    device_id = packet['device_id']
    packet_id = packet['packet_id']
    payload_b64 = packet['payload']

    # Decode payload
    payload = base64.b64decode(payload_b64)

    # Example: Decode temperature/humidity
    if len(payload) >= 4:
        import struct
        temp_raw, humid_raw = struct.unpack('>hH', payload[:4])
        temp_c = temp_raw / 100.0
        humid_pct = humid_raw / 100.0

        print(f"  Device {device_id[:8]}...: "
              f"{temp_c:.1f}°C, {humid_pct:.1f}% RH "
              f"(RSSI: {packet.get('rssi')})")

        # Store (in production, save to database)
        packet_log.append({
            'timestamp': packet['received_at'],
            'device_id': device_id,
            'packet_id': packet_id,
            'temperature_c': temp_c,
            'humidity_percent': humid_pct,
            'rssi': packet.get('rssi')
        })

@app.route('/stats', methods=['GET'])
def stats():
    """View packet statistics"""
    if not packet_log:
        return {"message": "No packets received yet"}, 200

    return {
        "total_packets": len(packet_log),
        "first_packet": packet_log[0]['timestamp'],
        "last_packet": packet_log[-1]['timestamp'],
        "devices": len(set(p['device_id'] for p in packet_log))
    }, 200

if __name__ == '__main__':
    print(f"Starting webhook server...")
    print(f"Webhook URL: http://localhost:5000/webhook/hubble")
    print(f"Stats URL: http://localhost:5000/stats")
    print(f"Secret: {WEBHOOK_SECRET}\n")

    app.run(host='0.0.0.0', port=5000, debug=True)
```

### Example 7: Register Webhook

```python
#!/usr/bin/env python3
"""Register webhook with Hubble"""

import requests
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')
WEBHOOK_URL = os.getenv('WEBHOOK_URL', 'https://your-domain.com/webhook/hubble')
WEBHOOK_SECRET = os.getenv('WEBHOOK_SECRET', 'your-secret-token')

def register_webhook(url, secret, batch_size=100):
    """Register webhook endpoint"""
    api_url = f"https://api.hubble.com/api/org/{ORG_ID}/webhooks"

    headers = {
        "Authorization": f"Bearer {API_TOKEN}",
        "Content-Type": "application/json"
    }

    payload = {
        "url": url,
        "name": "Production Webhook",
        "max_batch_size": batch_size,
        "secret": secret
    }

    response = requests.post(api_url, json=payload, headers=headers)

    if response.status_code == 201:
        webhook = response.json()['webhook']
        print("✓ Webhook registered successfully")
        print(f"  Webhook ID: {webhook['webhook_id']}")
        print(f"  URL: {webhook['url']}")
        print(f"  Batch Size: {webhook['max_batch_size']}")
        print(f"  Status: {webhook['status']}")
        return webhook
    else:
        print(f"✗ Registration failed: {response.status_code}")
        print(f"  Error: {response.text}")
        return None

def list_webhooks():
    """List all webhooks"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/webhooks"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        webhooks = response.json()['webhooks']
        print(f"\nActive webhooks: {len(webhooks)}")

        for webhook in webhooks:
            print(f"\n  {webhook['name']}")
            print(f"    ID: {webhook['webhook_id']}")
            print(f"    URL: {webhook['url']}")
            print(f"    Deliveries: {webhook.get('total_deliveries', 0)}")
            print(f"    Failed: {webhook.get('failed_deliveries', 0)}")

        return webhooks
    else:
        print(f"✗ Failed to list webhooks: {response.status_code}")
        return []

# Example usage
if __name__ == "__main__":
    # Register new webhook
    webhook = register_webhook(
        url=WEBHOOK_URL,
        secret=WEBHOOK_SECRET,
        batch_size=500
    )

    # List all webhooks
    list_webhooks()
```

---

## Metrics Examples

### Example 8: Monitor API and Webhook Health

```python
#!/usr/bin/env python3
"""Monitor API and webhook metrics"""

import requests
from datetime import datetime
from dotenv import load_dotenv
import os

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

def get_api_metrics(days=7):
    """Get API request metrics"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/api_metrics"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    params = {"days_back": days, "interval": "day"}

    response = requests.get(url, headers=headers, params=params)

    if response.status_code == 200:
        data = response.json()
        summary = data.get('summary', {})

        print(f"\nAPI Metrics (last {days} days):")
        print(f"  Total Requests: {summary.get('total_requests', 0):,}")
        print(f"  Success Rate: {summary.get('success_rate', 0):.1f}%")
        print(f"  Avg Response Time: {summary.get('avg_response_time_ms', 0):.0f}ms")

        return data
    else:
        print(f"✗ Failed to fetch API metrics: {response.status_code}")
        return None

def get_webhook_metrics(days=7):
    """Get webhook delivery metrics"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/webhook_metrics"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    params = {"days_back": days}

    response = requests.get(url, headers=headers, params=params)

    if response.status_code == 200:
        data = response.json()
        summary = data.get('summary', {})

        print(f"\nWebhook Metrics (last {days} days):")
        print(f"  Total Deliveries: {summary.get('total_deliveries', 0):,}")
        print(f"  Success Rate: {summary.get('success_rate', 0):.1f}%")
        print(f"  Avg Response Time: {summary.get('avg_response_time_ms', 0):.0f}ms")

        if summary.get('success_rate', 100) < 95:
            print("  ⚠ WARNING: Success rate below 95%")

        return data
    else:
        print(f"✗ Failed to fetch webhook metrics: {response.status_code}")
        return None

def get_packet_metrics(days=7):
    """Get packet volume metrics"""
    url = f"https://api.hubble.com/api/org/{ORG_ID}/packet_metrics"
    headers = {"Authorization": f"Bearer {API_TOKEN}"}
    params = {"days_back": days, "interval": "day"}

    response = requests.get(url, headers=headers, params=params)

    if response.status_code == 200:
        data = response.json()
        summary = data.get('summary', {})

        print(f"\nPacket Metrics (last {days} days):")
        print(f"  Total Packets: {summary.get('total_packets', 0):,}")
        print(f"  Total Devices: {summary.get('total_devices', 0):,}")
        print(f"  Total Bytes: {summary.get('total_bytes', 0):,}")

        return data
    else:
        print(f"✗ Failed to fetch packet metrics: {response.status_code}")
        return None

# Example usage
if __name__ == "__main__":
    print(f"=== Hubble Platform Health Report ===")
    print(f"Generated: {datetime.utcnow().isoformat()}Z")

    # Fetch all metrics
    api_metrics = get_api_metrics(days=7)
    webhook_metrics = get_webhook_metrics(days=7)
    packet_metrics = get_packet_metrics(days=7)

    print("\n✓ Health check complete")
```

---

## Complete Applications

### Example 9: Complete Device Onboarding Script

```python
#!/usr/bin/env python3
"""Complete device onboarding automation"""

import os
import base64
import requests
import time
from datetime import datetime
from dotenv import load_dotenv

load_dotenv()

API_TOKEN = os.getenv('HUBBLE_API_TOKEN')
ORG_ID = os.getenv('HUBBLE_ORG_ID')

class DeviceOnboarder:
    def __init__(self, api_token, org_id):
        self.api_token = api_token
        self.org_id = org_id
        self.base_url = "https://api.hubble.com"

    def generate_credentials(self, device_name):
        """Generate device credentials"""
        device_eui = os.urandom(8).hex().upper()
        device_key_binary = os.urandom(32)
        device_key_b64 = base64.b64encode(device_key_binary).decode('utf-8')

        return {
            'name': device_name,
            'dev_eui': device_eui,
            'device_key': device_key_b64,
            'device_key_binary': device_key_binary
        }

    def register_device(self, credentials, tags=None):
        """Register device with Hubble"""
        url = f"{self.base_url}/api/v2/org/{self.org_id}/devices"
        headers = {
            "Authorization": f"Bearer {self.api_token}",
            "Content-Type": "application/json"
        }

        payload = {
            "name": credentials['name'],
            "dev_eui": credentials['dev_eui'],
            "device_key": credentials['device_key']
        }

        if tags:
            payload["tags"] = tags

        response = requests.post(url, json=payload, headers=headers)
        response.raise_for_status()

        return response.json()['device']

    def verify_registration(self, device_eui):
        """Verify device appears in list"""
        url = f"{self.base_url}/api/org/{self.org_id}/devices"
        headers = {"Authorization": f"Bearer {self.api_token}"}

        response = requests.get(url, headers=headers)
        response.raise_for_status()

        devices = response.json()['devices']
        return any(d['dev_eui'] == device_eui for d in devices)

    def wait_for_first_packet(self, device_id, timeout=300):
        """Wait for device's first packet"""
        url = f"{self.base_url}/api/org/{self.org_id}/packets"
        headers = {"Authorization": f"Bearer {self.api_token}"}
        params = {"device_id": device_id, "limit": 1}

        start_time = time.time()

        while time.time() - start_time < timeout:
            response = requests.get(url, headers=headers, params=params)

            if response.status_code == 200:
                packets = response.json().get('packets', [])
                if packets:
                    return packets[0]

            time.sleep(10)

        return None

    def onboard_device(self, device_name, tags=None):
        """Complete onboarding workflow"""
        print(f"\n=== Onboarding: {device_name} ===\n")

        # Step 1: Generate credentials
        print("Step 1: Generating credentials...")
        credentials = self.generate_credentials(device_name)
        print(f"  EUI: {credentials['dev_eui']}")

        # Step 2: Register device
        print("\nStep 2: Registering device...")
        device = self.register_device(credentials, tags)
        print(f"  Device ID: {device['id']}")

        # Step 3: Verify
        print("\nStep 3: Verifying registration...")
        if self.verify_registration(credentials['dev_eui']):
            print("  ✓ Device verified")
        else:
            print("  ✗ Verification failed")
            return False

        # Step 4: Save credentials
        print("\nStep 4: Saving credentials...")
        filename = f"device_{credentials['dev_eui']}_key.bin"
        with open(filename, 'wb') as f:
            f.write(credentials['device_key_binary'])
        print(f"  Saved to: {filename}")

        # Step 5: Wait for packet (optional)
        print("\nStep 5: Waiting for first packet (5 min timeout)...")
        print("  (Flash firmware and send test packet now)")

        packet = self.wait_for_first_packet(device['id'], timeout=300)
        if packet:
            print(f"  ✓ First packet received!")
            print(f"    RSSI: {packet.get('rssi')}")
            print(f"    Received: {packet['received_at']}")
        else:
            print("  ⚠ No packet received (skipping)")

        print(f"\n✓ Onboarding complete for {device_name}!")
        return True

# Example usage
if __name__ == "__main__":
    onboarder = DeviceOnboarder(API_TOKEN, ORG_ID)

    # Onboard multiple devices
    devices = [
        ("Temperature Sensor #001", {"location": "warehouse-a", "type": "temp"}),
        ("Temperature Sensor #002", {"location": "warehouse-b", "type": "temp"}),
        ("Humidity Sensor #001", {"location": "warehouse-a", "type": "humid"})
    ]

    for name, tags in devices:
        try:
            onboarder.onboard_device(name, tags)
            time.sleep(2)  # Rate limiting
        except Exception as e:
            print(f"✗ Onboarding failed: {e}")
```

---

## curl Examples

### Quick Testing with curl

**Register Device**:
```bash
curl -X POST "https://api.hubble.com/api/v2/org/YOUR_ORG_ID/devices" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Sensor",
    "dev_eui": "70B3D54996C4C5A7",
    "device_key": "YmFzZTY0X2VuY29kZWRfa2V5X2hlcmU=",
    "tags": {"location": "lab"}
  }'
```

**List Devices**:
```bash
curl -X GET "https://api.hubble.com/api/org/YOUR_ORG_ID/devices" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Get Packets**:
```bash
curl -X GET "https://api.hubble.com/api/org/YOUR_ORG_ID/packets?start_time=2024-01-15T00:00:00Z&end_time=2024-01-16T00:00:00Z" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Register Webhook**:
```bash
curl -X POST "https://api.hubble.com/api/org/YOUR_ORG_ID/webhooks" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/webhook",
    "name": "Test Webhook",
    "max_batch_size": 100,
    "secret": "your-secret"
  }'
```

---

## Additional Resources

- [API_REFERENCE.md](./API_REFERENCE.md) - Complete endpoint documentation
- [WORKFLOWS.md](./WORKFLOWS.md) - Step-by-step guides
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Error solutions
- [SKILL.md](./SKILL.md) - Skill overview
