# Changelog

All notable changes to the tm1py-expert skill will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2024-01-13

### Added
- Initial release of tm1py-expert skill
- Core SKILL.md with step-by-step guidance for common tasks
- Comprehensive reference documentation:
  - API_REFERENCE.md: Complete TM1py API documentation
  - CONNECTION_GUIDE.md: Connection patterns for all TM1 environments
  - DATA_OPERATIONS.md: Reading and writing data guide
  - METADATA_MANAGEMENT.md: CRUD operations for all TM1 objects
  - PERFORMANCE.md: Performance optimization techniques
  - EXAMPLES.md: Real-world code samples and use cases
- Support for TM1py v2.0+
- Coverage of TM1 11, TM1 12, Planning Analytics on-premise, IBM Cloud, and Cloud Pak for Data
- Best practices for:
  - Secure connection management
  - Data operations (MDX, views, cell operations)
  - Metadata management (dimensions, cubes, processes, chores)
  - Performance optimization (blob, async, parallel operations)
  - Error handling and validation
  - Production workflow patterns

### Documentation
- README.md with quick start guide
- LICENSE (MIT)
- Comprehensive examples for:
  - Dimension management from external data
  - Data loading from CSV
  - Automated reporting
  - Multi-environment data sync
  - Security management
  - Monitoring and logging

### Features
- Connection patterns for all TM1 environments
- Complete API coverage for 13 major services
- Performance benchmarking utilities
- Production-ready code samples
- Troubleshooting guides

## [Unreleased]

### Planned
- Additional examples for advanced scenarios
- Integration patterns with pandas, numpy, other data tools
- Docker containerization examples
- CI/CD pipeline patterns
- Unit testing examples for TM1py scripts

### Changed
- Split metadata management reference into base and advanced guides
- Fixed documentation typos and links

---

[1.0.0]: https://github.com/DecisioNaut/tm1py-expert/releases/tag/v1.0.0
