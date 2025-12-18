# Hubble Cloud API - Claude Skill

A comprehensive Claude Code Agent Skill for integrating with the Hubble Network Platform API.

## Overview

This skill enables Claude to interact with the Hubble Network Platform API for managing IoT devices, retrieving packet data, configuring webhooks, and monitoring metrics across your Hubble Network deployment.

## Features

- **Complete API Coverage**: 40+ endpoints across 9 resource categories
- **Device Management**: Register, update, and organize devices with custom tags
- **Packet Streaming**: Efficient retrieval with continuation token pagination
- **Webhook Configuration**: Real-time packet delivery with validation
- **Metrics & Analytics**: Track API usage, packet volumes, and webhook health
- **User Management**: Invite users and manage access control
- **Billing Integration**: View invoices and track usage
- **Best Practices**: Rate limiting, error handling, security patterns
- **Code Examples**: Python and curl examples for all major operations

## Installation

### Via GitHub (Recommended)

```bash
claude plugin install github:hubble-network/claude-hubble-cloud-api-skill
```

### Local Development

```bash
# Clone repository
git clone https://github.com/hubble-network/claude-hubble-cloud-api-skill.git

# Install locally
claude plugin install file://$(pwd)/claude-hubble-cloud-api-skill
```

### Verify Installation

```bash
claude plugin list
```

## Quick Start

Once installed, Claude will automatically recognize Hubble API-related tasks:

```
You: Register a new temperature sensor called "Sensor-001" with my Hubble account
Claude: I'll help you register a new device...
```

## Prerequisites

- **Hubble Account**: Sign up at [hubble.com](https://hubble.com)
- **API Token**: Generate from Dashboard → Developer → API Tokens
- **Organization ID**: Found in API Tokens section or Organization Settings
- **Claude Code**: Install from [claude.com/claude-code](https://claude.com/claude-code)

## Documentation Structure

The skill includes comprehensive documentation:

- **[SKILL.md](skills/hubble-cloud-api/SKILL.md)** - Core skill instructions (loaded by Claude)
- **[API_REFERENCE.md](skills/hubble-cloud-api/API_REFERENCE.md)** - Complete endpoint documentation
- **[WORKFLOWS.md](skills/hubble-cloud-api/WORKFLOWS.md)** - Step-by-step implementation guides
- **[EXAMPLES.md](skills/hubble-cloud-api/EXAMPLES.md)** - Runnable code examples
- **[TROUBLESHOOTING.md](skills/hubble-cloud-api/TROUBLESHOOTING.md)** - Common issues and solutions
- **[resources/hubble-openapi.yaml](skills/hubble-cloud-api/resources/hubble-openapi.yaml)** - Official OpenAPI specification

## Usage Examples

### Register a Device

```
You: Register a new device with dev_eui 70B3D54996C4C5A7 and name "Temperature Sensor #1"
Claude: I'll register this device with Hubble...
```

### Stream Packets

```
You: Get all packets from the last 24 hours for warehouse devices
Claude: I'll query packets with the warehouse tag filter...
```

### Configure Webhook

```
You: Set up a webhook at https://my-app.com/webhook with batch size 500
Claude: I'll configure the webhook endpoint...
```

## Configuration

Set environment variables for convenience:

```bash
export HUBBLE_API_TOKEN="your-jwt-token"
export HUBBLE_ORG_ID="your-organization-id"
```

Store securely using:
- Environment variables (`.env` files with `python-dotenv`)
- AWS Secrets Manager
- HashiCorp Vault
- 1Password / LastPass

**Never commit API tokens to version control!**

## Key Capabilities

### Authentication
- Bearer token (JWT) authentication
- 16 distinct authorization scopes
- Granular access control

### Rate Limits
- 3 requests/second per endpoint
- 15 requests/second organization-wide
- Automatic retry with exponential backoff

### Data Handling
- Base64 encoding for binary data
- Continuation token pagination
- Batch operations (up to 1,000 devices)
- Webhook validation with secrets

## Common Workflows

1. **Device Onboarding**
   - Generate credentials → Register → Verify → Flash firmware → Test connectivity

2. **Packet Streaming**
   - Query time range → Process batch → Handle continuation tokens → Decode payloads

3. **Webhook Setup**
   - Implement endpoint → Register with Hubble → Validate tokens → Monitor metrics

4. **API Key Rotation**
   - Create new key → Update applications → Verify → Revoke old key

5. **Fleet Management**
   - List devices → Filter by tags → Prepare updates → Execute batch → Verify

## Support

- **Documentation**: [docs.hubble.com](https://docs.hubble.com)
- **GitHub Issues**: [github.com/hubble-network/claude-hubble-cloud-api-skill/issues](https://github.com/hubble-network/claude-hubble-cloud-api-skill/issues)
- **Support Email**: support@hubble.com
- **Developer Portal**: [hubble.com/developers](https://hubble.com/developers)

## Contributing

Contributions welcome! Please submit pull requests or open issues for:
- Bug fixes
- Documentation improvements
- New examples or workflows
- Feature requests

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Version

1.0.0

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and updates.

## Credits

Developed by **Hubble Network** for the Claude Code community.

## Related Resources

- [Hubble Cloud API Docs](https://docs.hubble.com/docs/api-specification/hubble-cloud-api)
- [Hubble Device SDK](https://docs.hubble.com/docs/device-sdk/intro)
- [Claude Code Documentation](https://code.claude.com/docs)
- [Claude Agent Skills Guide](https://code.claude.com/docs/en/skills)
