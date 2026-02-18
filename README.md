# TM1py Expert: Agent Skill

> Comprehensive guidance for TM1py development, best practices, and advanced usage patterns.

## Overview

The `tm1py-expert` skill provides expert-level assistance for working with [TM1py](https://github.com/cubewise-code/tm1py), the Python package for IBM Planning Analytics (TM1). It helps developers and administrators with:

- **Connection Management**: Secure connections to all TM1/Planning Analytics environments
- **Data Operations**: Reading and writing data efficiently with all available methods
- **Metadata Management**: CRUD operations for dimensions, hierarchies, cubes, processes, and more
- **Performance Optimization**: Best practices for high-performance operations
- **Real-World Examples**: Production-ready code samples and patterns

## Installation as Agent Skill

### Option 1: Copy to Skills Directory (Recommended)

For use with Claude/GitHub Copilot (via agent skills):

1. Copy the **`tm1py-expert` subfolder** (not the root) to your agent skills directory:
   ```bash
   # macOS/Linux
   mkdir -p ~/.copilot/skills/
   cp -r tm1py-expert/tm1py-expert ~/.copilot/skills/tm1py-expert
   
   # Or create a symbolic link for development
   ln -s /path/to/tm1py-expert/tm1py-expert ~/.copilot/skills/tm1py-expert
   ```

2. The skill will be automatically loaded when you mention TM1py topics in your conversations.

### Option 2: Reference Directly

If hosting in a repository, configure your agent to load from:
```
<repository-root>/tm1py-expert/tm1py-expert/
```

The skill structure complies with the [Agent Skills Specification](https://agentskills.io/specification).

## When to Use

The skill activates when working with:
- TM1py package development
- IBM Planning Analytics / TM1 automation  
- Python scripts for TM1 data operations
- TM1 REST API interactions via Python
- Planning Analytics administration tasks

## Prerequisites

- Python 3.7 or higher
- TM1py package installed (`pip install tm1py`)
- Access to TM1 or Planning Analytics instance (v11 or higher)
- Appropriate TM1 user credentials

## Repository Structure

```
tm1py-expert/ (repository root)
├── README.md                         # This file
├── LICENSE                           # MIT License
├── CHANGELOG.md                      # Version history
└── tm1py-expert/ (agent skill folder)
    ├── SKILL.md                      # Core skill instructions
    └── references/                   # Detailed reference documentation
        ├── API_REFERENCE.md          # Complete API documentation
        ├── CONNECTION_GUIDE.md       # Connection patterns for all environments
        ├── DATA_OPERATIONS.md        # Reading/writing data guide
        ├── METADATA_MANAGEMENT.md    # CRUD for all TM1 objects
        ├── METADATA_MANAGEMENT_ADVANCED.md # Advanced metadata patterns
        ├── PERFORMANCE.md            # Optimization techniques
        └── EXAMPLES.md               # Real-world code samples
```

**Note:** Only the contents of `tm1py-expert/tm1py-expert/` folder should be installed as an agent skill.

## Documentation

### Core Documentation

- **[SKILL.md](tm1py-expert/SKILL.md)**: Start here for step-by-step guidance on common tasks

### Reference Guides

- **[API Reference](tm1py-expert/references/API_REFERENCE.md)**: Complete documentation of all TM1py services and methods
- **[Connection Guide](tm1py-expert/references/CONNECTION_GUIDE.md)**: Connection patterns for TM1 11, TM1 12, PAaaS, Cloud Pak for Data
- **[Data Operations](tm1py-expert/references/DATA_OPERATIONS.md)**: Comprehensive guide to reading and writing data
- **[Metadata Management](tm1py-expert/references/METADATA_MANAGEMENT.md)**: CRUD operations for dimensions, cubes, processes, etc.
- **[Metadata Management Advanced](tm1py-expert/references/METADATA_MANAGEMENT_ADVANCED.md)**: Advanced patterns and workflows
- **[Performance Guide](tm1py-expert/references/PERFORMANCE.md)**: Performance optimization and benchmarking
- **[Examples](tm1py-expert/references/EXAMPLES.md)**: Real-world code samples and use cases

## Features

### Connection Management
- Secure connection patterns for all TM1 versions
- Environment variable and config file patterns
- SSL/TLS configuration
- OAuth2 and API key authentication
- Connection pooling and timeouts

### Data Operations
- MDX query execution (DataFrame, CSV, JSON)
- View execution (native and MDX views)
- Single cell and bulk cell operations
- Write operations with blob, TI, and async methods
- Data validation and transformation patterns

### Metadata Management
- Complete CRUD for all TM1 objects:
  - Dimensions, Hierarchies, Elements, Attributes
  - Cubes, Views, Subsets
  - Processes (TI), Chores
  - Sandboxes, Security objects
- Batch operations for performance
- Alternate hierarchy management

### Performance Optimization
- Blob operations (40-100% faster)
- Async/parallel processing
- Memory-efficient iterative JSON
- Transaction log management
- Benchmarking utilities

## Examples

### Quick Connect

```python
from TM1py import TM1Service

config = {
    'address': 'localhost',
    'port': 8001,
    'user': 'admin',
    'password': 'apple',
    'ssl': True
}

with TM1Service(**config) as tm1:
    server_name = tm1.server.get_server_name()
    print(f"Connected to: {server_name}")
```

### Read Data

```python
mdx = """
SELECT
  [Product].[Product].Members ON ROWS,
  [Period].[2024].Children ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual])
"""

with TM1Service(**config) as tm1:
    df = tm1.cells.execute_mdx_dataframe(mdx, use_blob=True)
    print(df)
```

### Write Data

```python
cellset = {
    ('Product1', '2024-Q1', 'Actual', 'Revenue'): 50000,
    ('Product2', '2024-Q1', 'Actual', 'Revenue'): 75000
}

with TM1Service(**config) as tm1:
    tm1.cells.write(
        cube_name='Sales',
        cellset_as_dict=cellset,
        dimensions=['Product', 'Period', 'Version', 'Measure'],
        use_blob=True
    )
```

See [EXAMPLES.md](tm1py-expert/references/EXAMPLES.md) for more comprehensive examples.

## Resources

### Official Documentation
- [TM1py Documentation](https://tm1py.readthedocs.io/)
- [TM1py GitHub Repository](https://github.com/cubewise-code/tm1py)
- [TM1py Samples Repository](https://github.com/cubewise-code/tm1py-samples)
- [IBM TM1 REST API Documentation](https://www.ibm.com/docs/en/planning-analytics/2.0.0?topic=api-tm1-rest)

### Community
- GitHub Issues: [Report bugs or request features](https://github.com/cubewise-code/tm1py/issues)
- Cubewise Community Forums

## Contributing

Contributions to improve this skill are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This skill is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Version

Current version: 1.1.0

See [CHANGELOG.md](CHANGELOG.md) for version history.

## Author

Created following the [Agent Skills Specification](https://agentskills.io/specification).

## Acknowledgments

- [skill-smith](https://github.com/copilot-skills/skills/blob/main/skill-smith) for skill structure and validation guidance
- [Cubewise](https://www.cubewise.com/) for developing and maintaining TM1py
- The TM1py community for contributions and examples
- IBM for Planning Analytics / TM1

---

For detailed guidance, start with [SKILL.md](tm1py-expert/SKILL.md).
