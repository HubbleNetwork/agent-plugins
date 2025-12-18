# Changelog

All notable changes to the Hubble Cloud API Claude Skill will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2025-01-15

### Added

#### Packet Structure Documentation
- **Comprehensive nested structure documentation** in SKILL.md
  - Detailed breakdown of `packet.device` and `packet.location` objects
  - Field-by-field mapping with correct access patterns
  - Common mistakes section highlighting incorrect field access patterns
  - Helper function example for extracting packet information

#### Real-World Examples
- **Example 6: Real-Time Packet Monitoring with Deduplication** (Python)
  - Complete monitoring class with proper packet structure handling
  - Initial load pattern (marks existing packets as seen without processing)
  - Packet deduplication using device ID + timestamp + sequence number
  - Real-time polling with configurable intervals
  - Graceful shutdown and statistics

- **Example 7: Extract and Visualize Packet Data** (JavaScript/Node.js)
  - Demonstrates correct nested structure access
  - Helper functions for device info extraction
  - Unix timestamp to JavaScript Date conversion
  - Packet grouping and aggregation by device
  - RSSI averaging calculations

### Changed

#### SKILL.md
- Added "Packet Structure" subsection under "Packet Retrieval"
- Documented actual API response format with nested objects
- Added visual JSON structure example with inline comments
- Included "Important Field Locations" reference table
- Added "Common Mistakes" section with ❌/✅ indicators

#### EXAMPLES.md
- **Fixed Example 4**: Updated to use correct nested packet structure
  - Changed `packet['device_id']` to `packet['device']['id']`
  - Changed `packet['payload']` to `packet['device']['payload']`
  - Added device name extraction from `packet['device']['name']`

### Technical Improvements

#### Packet Field Mappings Documented
- Device ID: `packet.device.id` (UUID format)
- Device Name: `packet.device.name`
- RSSI: `packet.device.rssi` (not at top level)
- Sequence Number: `packet.device.sequence_number`
- GPS Coordinates: `packet.location.latitude/longitude`
- Timestamp: `packet.location.timestamp` (Unix epoch seconds)
- Network Type: `packet.network_type` (top-level)

#### Real-World Patterns
- Initial fetch pattern (load existing without processing)
- Continuous polling with deduplication
- Packet ID generation for uniqueness
- Nested field extraction helpers
- Location timestamp conversion (Unix to ISO/Date)

### Notes
- Based on production experience building real-time packet visualization application
- All examples tested against live Hubble API
- Addresses common developer confusion about field locations
- Reduces time-to-first-packet for new integrators

## [1.0.0] - 2024-01-21

### Added

#### Core Skill
- Complete Hubble Cloud API integration skill
- Support for 40+ API endpoints across 9 categories
- Progressive disclosure documentation structure

#### Device Management
- Device registration with Base64 key encoding
- Device listing with filtering by tags and platform
- Batch device updates (up to 1,000 devices per request)
- Device tagging and organization

#### Packet Retrieval
- Packet streaming with continuation token pagination
- Time-based filtering (default 7-day windows)
- Device-specific packet queries
- Platform tag filtering
- Base64 payload decoding guidance

#### Webhook Management
- Webhook registration and configuration
- Batch size customization (10-1000 packets)
- Token-based webhook validation
- Webhook metrics monitoring
- Delivery success rate tracking

#### API Key Management
- Key creation with granular scopes
- API key listing and auditing
- Key rotation workflows
- Secure key storage best practices

#### Metrics & Monitoring
- API request metrics with hourly breakdowns
- Packet volume tracking
- Webhook delivery metrics
- Device activity monitoring

#### Organization & User Management
- Organization details and updates
- User invitation system
- Role-based access control (Admin/Member)
- User management operations

#### Billing
- Invoice listing and PDF downloads
- Usage tracking (daily/monthly active devices)

#### Documentation
- Comprehensive API reference with 40+ endpoints
- 5 detailed workflow guides (device onboarding, packet streaming, webhooks, key management, batch operations)
- Runnable code examples (Python and curl)
- Troubleshooting guide for common errors
- Complete OpenAPI specification

#### Best Practices
- Rate limiting strategies (exponential backoff)
- Error handling patterns
- Security recommendations
- Base64 encoding helpers
- Pagination patterns
- Webhook validation

#### Code Examples
- Python implementations for all major operations
- curl examples for testing
- Complete workflow scripts
- Error handling templates
- Batch operation helpers

### Technical Details
- OpenAPI 3.0.0 specification included
- 16 distinct authorization scopes
- Rate limits: 3 req/sec per endpoint, 15 req/sec org-wide
- Continuation token pagination for large datasets
- Base64 encoding for binary data (device keys, payloads)

### Documentation
- SKILL.md: Core skill instructions (~400 lines)
- API_REFERENCE.md: Complete endpoint documentation (~1200 lines)
- WORKFLOWS.md: Step-by-step guides (~400 lines)
- EXAMPLES.md: Runnable code examples (~600 lines)
- TROUBLESHOOTING.md: Error resolution (~300 lines)
- README.md: Installation and overview (~150 lines)

### Notes
- Initial release for Claude Code community
- Production-ready for Hubble customers
- Compatible with Claude Code v2.0.20+

---

## Release Notes Template

### [X.Y.Z] - YYYY-MM-DD

#### Added
- New features

#### Changed
- Changes to existing functionality

#### Deprecated
- Soon-to-be removed features

#### Removed
- Removed features

#### Fixed
- Bug fixes

#### Security
- Security updates

---

## Future Roadmap

### Planned for v1.1.0
- Additional code examples for advanced use cases
- Integration with popular Python frameworks (Django, FastAPI)
- Docker deployment examples
- Kubernetes webhook configuration
- Advanced filtering examples
- Device firmware update workflows

### Planned for v1.2.0
- TypeScript/Node.js examples
- Go language examples
- Terraform configuration templates
- Monitoring and alerting examples (Grafana, Datadog)
- CI/CD pipeline integration examples

### Under Consideration
- Interactive tutorials
- Video walkthroughs
- Sample applications
- Load testing examples
- Multi-region deployment patterns
