# TM1py Metadata Management - Advanced

Advanced patterns and workflows for TM1 metadata management using TM1py.

---

## Cubes

### Create Cube

```python
from TM1py import Cube

# Simple cube
cube = Cube(
    name='Sales',
    dimensions=['Product', 'Period', 'Version', 'Measure']
)
tm1.cubes.create(cube)

# With rules
rules = """
# Calculate Profit
['Profit'] = ['Revenue'] - ['Cost'];

# Spread to children
FEEDERS;
['Revenue'] => ['Profit'];
['Cost'] => ['Profit'];
"""

cube = Cube(
    name='Sales',
    dimensions=['Product', 'Period', 'Version', 'Measure'],
    rules=rules
)
tm1.cubes.create(cube)
```

### Read Cube

```python
# Get cube object
cube = tm1.cubes.get(cube_name='Sales')

# Get cube dimension names
dimensions = tm1.cubes.get_dimension_names(cube_name='Sales')

# Get all cube names
all_cubes = tm1.cubes.get_all_names()

# Get control cubes only
control_cubes = tm1.cubes.get_all_names(skip_control_cubes=True)

# Check if exists
exists = tm1.cubes.exists(cube_name='Sales')

# Get cube properties
rules = tm1.cubes.get_rules(cube_name='Sales')
```

### Update Cube

```python
# Get cube
cube = tm1.cubes.get('Sales')

# Update rules
new_rules = """
# Updated calculation
['Profit'] = ['Revenue'] * 0.2;

FEEDERS;
['Revenue'] => ['Profit'];
"""

tm1.cubes.update_rules(cube_name='Sales', rules=new_rules)

# Or update entire cube
cube.rules = new_rules
tm1.cubes.update(cube)
```

### Delete Cube

```python
tm1.cubes.delete(cube_name='Sales')
```

### Check Feeders

```python
# Check if feeders are valid
trace = tm1.cubes.check_feeders(cube_name='Sales')
print(trace)
```

---

## Common Patterns

### Complete Dimension Setup

```python
def setup_dimension_full(tm1, dim_name, elements_data):
    """
    Complete dimension setup with elements, attributes, and edges.

    Args:
        elements_data: List of dicts with 'name', 'type', 'parent', 'attr1', 'attr2'...
    """

    # Create dimension
    dimension = Dimension(name=dim_name)
    hierarchy = Hierarchy(name=dim_name, dimension_name=dim_name)

    # Add elements
    for elem_data in elements_data:
        hierarchy.add_element(
            element_name=elem_data['name'],
            element_type=elem_data.get('type', 'Numeric')
        )

    # Add edges
    edges = {}
    for elem_data in elements_data:
        if 'parent' in elem_data and elem_data['parent']:
            edges[(elem_data['parent'], elem_data['name'])] = 1

    for (parent, child), weight in edges.items():
        hierarchy.add_edge(parent, child, weight)

    dimension.add_hierarchy(hierarchy)
    tm1.dimensions.create(dimension)

    # Create attributes
    attribute_cols = [k for k in elements_data[0].keys() if k not in ['name', 'type', 'parent']]
    for attr_name in attribute_cols:
        attr_type = 'String'  # Detect type from first value
        first_val = elements_data[0][attr_name]
        if isinstance(first_val, (int, float)):
            attr_type = 'Numeric'

        attr = ElementAttribute(name=attr_name, attribute_type=attr_type)
        tm1.elements.create_element_attribute(dim_name, dim_name, attr)

    # Set attribute values
    for elem_data in elements_data:
        for attr_name in attribute_cols:
            tm1.elements.update_element_attribute(
                dimension_name=dim_name,
                hierarchy_name=dim_name,
                element_name=elem_data['name'],
                attribute_name=attr_name,
                attribute_value=elem_data[attr_name]
            )

    print(f"Dimension '{dim_name}' created with {len(elements_data)} elements")
```

### Clone Cube Structure

```python
def clone_cube(tm1, source_cube, target_cube):
    """Clone cube structure (dimensions and rules)"""

    # Get source cube
    source = tm1.cubes.get(source_cube)

    # Create new cube with same dimensions
    new_cube = Cube(
        name=target_cube,
        dimensions=source.dimensions,
        rules=source.rules
    )

    tm1.cubes.create(new_cube)
    print(f"Cloned '{source_cube}' to '{target_cube}'")
```

---

For core CRUD operations, see `METADATA_MANAGEMENT.md`.  
For API reference, see `API_REFERENCE.md`.
