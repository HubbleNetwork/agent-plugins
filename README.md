# Hubble Agent Plugins

Official Claude Code plugins and skills for Hubble Network Platform API integration.

## Overview

This repository provides a curated marketplace of Claude Code plugins designed to enhance your development workflow when working with Hubble Network's IoT platform. These plugins enable Claude to interact seamlessly with Hubble's Cloud API for device management, packet retrieval, webhook configuration, and more.

## Available Plugins

### hubble-cloud-api

Complete Claude Skill for Hubble Network Cloud API integration.

**Features:**
- Device Management (registration, updates, tagging)
- Packet Streaming with pagination
- Webhook Configuration and validation
- Metrics & Analytics
- User Management
- Billing Integration

[View Plugin Documentation →](./hubble-cloud-api-skill/README.md)

## Installation

### Option 1: Add the Marketplace (Recommended)

Add this repository as a marketplace source to browse and install plugins:

```bash
claude plugin marketplace add https://github.com/hubble-network/agent-plugins
```

Once added, you can browse available plugins:

```bash
claude plugin marketplace list
```

### Option 2: Direct Plugin Installation

Install plugins directly from this repository:

#### Install via GitHub

```bash
claude plugin install github:hubble-network/agent-plugins/hubble-cloud-api-skill
```

#### Install from Local Clone

```bash
# Clone the repository
git clone https://github.com/hubble-network/agent-plugins.git
cd agent-plugins

# Install a specific plugin
claude plugin install file://$(pwd)/hubble-cloud-api-skill
```

### Verify Installation

Check that the plugin is installed and active:

```bash
claude plugin list
```

You should see `hubble-cloud-api` in the list of installed plugins.

## Prerequisites

Before using these plugins, ensure you have:

1. **Claude Code**: Install from [claude.com/claude-code](https://claude.com/claude-code)
2. **Hubble Account**: Sign up at [hubble.com](https://hubble.com)
3. **API Token**: Generate from Dashboard → Developer → API Tokens
4. **Organization ID**: Found in API Tokens section or Organization Settings

## Quick Start

Once installed, Claude will automatically recognize Hubble API-related tasks:

```
You: Register a new temperature sensor with dev_eui 70B3D54996C4C5A7
Claude: I'll help you register this device with Hubble Network...
```

```
You: Get packets from the last 24 hours for devices tagged "warehouse"
Claude: I'll query packets with the warehouse tag filter...
```

```
You: Set up a webhook at https://my-app.com/webhook
Claude: I'll configure the webhook endpoint...
```

## Configuration

### API Authentication

Set environment variables for convenience:

```bash
export HUBBLE_API_TOKEN="your-jwt-token"
export HUBBLE_ORG_ID="your-organization-id"
```

### Secure Storage

Store credentials securely using:
- Environment variables (`.env` files with `python-dotenv`)
- AWS Secrets Manager
- HashiCorp Vault
- 1Password / LastPass

**Never commit API tokens to version control!**

## Plugin Management

### List Installed Plugins

```bash
claude plugin list
```

### Update Plugins

```bash
claude plugin update hubble-cloud-api
```

### Uninstall Plugins

```bash
claude plugin uninstall hubble-cloud-api
```

### Remove Marketplace

```bash
claude plugin marketplace remove hubble-agent-plugins
```

## Development

### Repository Structure

```
agent-plugins/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace configuration
├── hubble-cloud-api-skill/       # Hubble Cloud API plugin
│   ├── .claude-plugin/
│   │   └── plugin.json           # Plugin metadata
│   ├── skills/
│   │   └── hubble-cloud-api/     # Skill implementation
│   └── README.md                 # Plugin documentation
└── README.md                     # This file
```

### Adding New Plugins

To contribute a new plugin:

1. Create a plugin directory following the structure above
2. Add plugin metadata in `.claude-plugin/plugin.json`
3. Update `.claude-plugin/marketplace.json` with the new plugin
4. Include comprehensive documentation
5. Submit a pull request

## Support

### Documentation
- **Hubble API Docs**: [docs.hubble.com](https://docs.hubble.com)
- **Claude Code Docs**: [code.claude.com/docs](https://code.claude.com/docs)
- **Plugin Guide**: [code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins)

### Getting Help
- **GitHub Issues**: [github.com/hubble-network/agent-plugins/issues](https://github.com/hubble-network/agent-plugins/issues)
- **Support Email**: support@hubble.com
- **Developer Portal**: [hubble.com/developers](https://hubble.com/developers)

## Contributing

Contributions are welcome! Please feel free to:
- Report bugs or issues
- Suggest new features or plugins
- Improve documentation
- Submit pull requests

## License

MIT License - See individual plugin directories for specific license details.

## About Hubble Network

Hubble Network provides global IoT connectivity via satellite, enabling devices to communicate from anywhere on Earth. Our Cloud API allows developers to manage devices, retrieve data, and integrate Hubble connectivity into their applications.

Learn more at [hubble.com](https://hubble.com)

## Related Resources

- [Hubble Cloud API Documentation](https://docs.hubble.com/docs/api-specification/hubble-cloud-api)
- [Hubble Device SDK](https://docs.hubble.com/docs/device-sdk/intro)
- [Claude Code Skills Guide](https://code.claude.com/docs/en/skills)
- [Anthropic API Documentation](https://docs.anthropic.com)
