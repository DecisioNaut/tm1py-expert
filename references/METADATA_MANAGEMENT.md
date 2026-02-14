# TM1py Metadata Management

Comprehensive guide to CRUD operations for all TM1 metadata objects.

---

## Dimensions

### Create Dimension

```python
from TM1py import Dimension, Hierarchy, Element, ElementAttribute

# Create simple dimension
dimension = Dimension(name='Region')
hierarchy = Hierarchy(name='Region', dimension_name='Region')

# Add elements
hierarchy.add_element(element_name='US', element_type='Numeric')
hierarchy.add_element(element_name='UK', element_type='Numeric')
hierarchy.add_element(element_name='DE', element_type='Numeric')

# Add consolidated element
hierarchy.add_element(element_name='Total', element_type='Consolidated')

# Add edges (parent-child relationships)
hierarchy.add_edge(parent='Total', component='US', weight=1)
hierarchy.add_edge(parent='Total', component='UK', weight=1)
hierarchy.add_edge(parent='Total', component='DE', weight=1)

# Add to dimension and create
dimension.add_hierarchy(hierarchy)
tm1.dimensions.create(dimension)
```

### Read Dimension

```python
# Get full dimension object
dimension = tm1.dimensions.get(dimension_name='Region')

# Check if exists
exists = tm1.dimensions.exists(dimension_name='Region')

# Get all dimension names
all_dims = tm1.dimensions.get_all_names()

# Get dimension count
count = tm1.dimensions.__len__()
```

### Update Dimension

```python
# Get, modify, update
dimension = tm1.dimensions.get('Region')

# Get default hierarchy
hierarchy = dimension.default_hierarchy

# Add new element
hierarchy.add_element('FR', 'Numeric')
hierarchy.add_edge('Total', 'FR', 1)

# Update in TM1
tm1.dimensions.update(dimension)
```

### Delete Dimension

```python
tm1.dimensions.delete(dimension_name='Region')
```

---

## Hierarchies

### Create Alternate Hierarchy

```python
# Get existing dimension
dimension = tm1.dimensions.get('Product')

# Create new hierarchy
alt_hierarchy = Hierarchy(
    name='ProductByCategory',
    dimension_name='Product'
)

# Add elements (can reference existing elements)
alt_hierarchy.add_element('Product1', 'Numeric')
alt_hierarchy.add_element('Product2', 'Numeric')
alt_hierarchy.add_element('Electronics', 'Consolidated')
alt_hierarchy.add_element('Furniture', 'Consolidated')

# Different structure
alt_hierarchy.add_edge('Electronics', 'Product1', 1)
alt_hierarchy.add_edge('Furniture', 'Product2', 1)

# Add hierarchy to dimension
dimension.add_hierarchy(alt_hierarchy)
tm1.dimensions.update(dimension)
```

### Read Hierarchy

```python
# Get hierarchy
hierarchy = tm1.hierarchies.get(
    dimension_name='Product',
    hierarchy_name='ProductByCategory'
)

# Get all hierarchy names in dimension
hierarchy_names = tm1.hierarchies.get_all_names(dimension_name='Product')

# Check if exists
exists = tm1.hierarchies.exists(
    dimension_name='Product',
    hierarchy_name='ProductByCategory'
)
```

### Update Hierarchy

```python
hierarchy = tm1.hierarchies.get('Product', 'ProductByCategory')

# Modify structure
hierarchy.add_element('Product3', 'Numeric')
hierarchy.add_edge('Electronics', 'Product3', 1)

# Update
tm1.hierarchies.update(hierarchy)
```

### Delete Hierarchy

```python
tm1.hierarchies.delete(
    dimension_name='Product',
    hierarchy_name='ProductByCategory'
)
```

---

## Elements

### Create Elements

```python
from TM1py import Element

# Single element
element = Element(name='Product1', element_type='Numeric')
tm1.elements.create(
    dimension_name='Product',
    hierarchy_name='Product',
    element=element
)

# Multiple elements (batch - more efficient)
elements = [
    Element(name=f'Product{i}', element_type='Numeric')
    for i in range(1, 101)
]

tm1.elements.add_elements(
    dimension_name='Product',
    hierarchy_name='Product',
    elements=elements
)
```

### Read Elements

```python
# Get single element
element = tm1.elements.get(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1'
)

# Get all element names
element_names = tm1.elements.get_element_names(
    dimension_name='Product',
    hierarchy_name='Product'
)

# Get leaf elements only
leaf_elements = tm1.elements.get_leaf_element_names(
    dimension_name='Product',
    hierarchy_name='Product'
)

# Get elements as DataFrame
df = tm1.elements.get_elements_dataframe(
    dimension_name='Product',
    hierarchy_name='Product',
    skip_consolidations=False,
    attributes=['Category', 'Status']
)

# Check if exists
exists = tm1.elements.exists(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1'
)

# Get element type
elem_type = tm1.elements.get_element_type(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Total'
)  # Returns 'Numeric', 'String', or 'Consolidated'
```

### Update Element

```python
# Change element type (limited - usually delete/recreate)
element = tm1.elements.get('Product', 'Product', 'Product1')
element.element_type = 'String'

tm1.elements.update(
    dimension_name='Product',
    hierarchy_name='Product',
    element=element
)

# Note: Changing element type has restrictions in TM1
```

### Delete Elements

```python
# Delete single element
tm1.elements.delete(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1'
)

# Delete multiple (more efficient)
tm1.elements.delete_elements(
    dimension_name='Product',
    hierarchy_name='Product',
    element_names=['Product1', 'Product2', 'Product3']
)
```

---

## Element Attributes

### Create Attributes

```python
from TM1py import ElementAttribute

# Create attribute
attribute = ElementAttribute(name='Category', attribute_type='String')
tm1.elements.create_element_attribute(
    dimension_name='Product',
    hierarchy_name='Product',
    element_attribute=attribute
)

# Create numeric attribute
price_attribute = ElementAttribute(name='Price', attribute_type='Numeric')
tm1.elements.create_element_attribute('Product', 'Product', price_attribute)
```

### Read Attributes

```python
# Get all attributes
attributes = tm1.elements.get_element_attributes(
    dimension_name='Product',
    hierarchy_name='Product'
)

# Get attribute names
attr_names = tm1.elements.get_element_attribute_names(
    dimension_name='Product',
    hierarchy_name='Product'
)

# Get attribute value for element
value = tm1.elements.get_element_attribute(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1',
    attribute_name='Category'
)
```

### Update Attribute Values

```python
# Set attribute value for element
tm1.elements.update_element_attribute(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1',
    attribute_name='Category',
    attribute_value='Electronics'
)

# Bulk update from DataFrame
import pandas as pd

df = pd.DataFrame({
    'Element': ['Product1', 'Product2', 'Product3'],
    'Category': ['Electronics', 'Furniture', 'Electronics'],
    'Price': [299.99, 599.99, 399.99]
})

for idx, row in df.iterrows():
    element = row['Element']
    for attr in ['Category', 'Price']:
        tm1.elements.update_element_attribute(
            dimension_name='Product',
            hierarchy_name='Product',
            element_name=element,
            attribute_name=attr,
            attribute_value=row[attr]
        )
```

### Delete Attribute

```python
tm1.elements.delete_element_attribute(
    dimension_name='Product',
    hierarchy_name='Product',
    attribute_name='Category'
)
```

---

## Edges (Parent-Child Relationships)

### Create Edges

```python
# Single edge
tm1.elements.add_edge(
    dimension_name='Product',
    hierarchy_name='Product',
    parent='AllProducts',
    component='Product1',
    weight=1
)

# Multiple edges (batch - more efficient)
edges = {
    ('AllProducts', 'Product1'): 1,
    ('AllProducts', 'Product2'): 1,
    ('AllProducts', 'Product3'): 1,
    ('Electronics', 'Product1'): 1,
    ('Furniture', 'Product2'): 1
}

tm1.elements.add_edges(
    dimension_name='Product',
    hierarchy_name='Product',
    edges=edges
)
```

### Read Edges

```python
# Get children of element
children = tm1.elements.get_element_names(
    dimension_name='Product',
    hierarchy_name='Product'
)

# Get parents of element
parents = tm1.elements.get_parents(
    dimension_name='Product',
    hierarchy_name='Product',
    element_name='Product1'
)

# Get all edges as DataFrame
edges_df = tm1.elements.get_edges_dataframe(
    dimension_name='Product',
    hierarchy_name='Product'
)
```

### Delete Edge

```python
tm1.elements.delete_edge(
    dimension_name='Product',
    hierarchy_name='Product',
    parent='AllProducts',
    component='Product1'
)
```

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

## Processes (TI)

### Create Process

```python
from TM1py import Process

# Simple process
process = Process(
    name='Update_Products',
    prolog_procedure="""
    # Prolog code
    sMessage = 'Starting update';
    """,
    metadata_procedure="",
    data_procedure="",
    epilog_procedure="""
    # Epilog code
    sMessage = 'Update complete';
    """
)

tm1.processes.create(process)
```

### Create Process with Parameters

```python
from TM1py import Process, ProcessParameter

process = Process(name='Load_Data')

# Add parameters
process.add_parameter(
    ProcessParameter(name='pYear', prompt='Year', value='2024')
)
process.add_parameter(
    ProcessParameter(name='pVersion', prompt='Version', value='Actual')
)

# Add code
process.prolog_procedure = """
# Use parameters
sYear = pYear;
sVersion = pVersion;

# Process logic
...
"""

tm1.processes.create(process)
```

### Create Process with Data Source

```python
from TM1py import Process

# Process with ASCII data source
process = Process(
    name='Import_Data',
    datasource_type='ASCII',
    datasource_ascii_delimiter_char=',',
    datasource_ascii_quote_character='"',
    datasource_data_source_name_for_server='C:\\Data\\import.csv',
    datasource_data_source_name_for_client='C:\\Data\\import.csv'
)

# Define variables
process.add_variable(name='vProduct', variable_type='String')
process.add_variable(name='vValue', variable_type='Numeric')

# Add processing code
process.metadata_procedure = """
# Metadata tab - runs once
"""

process.data_procedure = """
# Data tab - runs per row
CellPutN(vValue, 'Sales', vProduct, '2024-Q1', 'Actual', 'Revenue');
"""

tm1.processes.create(process)
```

### Read Process

```python
# Get process
process = tm1.processes.get(process_name='Update_Products')

# Get all process names
all_processes = tm1.processes.get_all_names()

# Check if exists
exists = tm1.processes.exists(process_name='Update_Products')

# Get process code
code = tm1.processes.get_processcode(process_name='Update_Products')
```

### Update Process

```python
# Get and modify
process = tm1.processes.get('Update_Products')

process.prolog_procedure = """
# Updated logic
sMessage = 'New version';
"""

tm1.processes.update(process)
```

### Execute Process

```python
# Simple execution
tm1.processes.execute(process_name='Update_Products')

# With parameters
tm1.processes.execute(
    process_name='Load_Data',
    pYear='2024',
    pVersion='Forecast'
)

# With return values
success, status, error = tm1.processes.execute_with_return(
    process_name='Load_Data',
    pYear='2024'
)

if success:
    print(f"Process succeeded: {status}")
else:
    print(f"Process failed: {error}")

# With timeout
success, status, error = tm1.processes.execute_with_return(
    process_name='Long_Process',
    timeout=600,  # 10 minutes
    cancel_at_timeout=True
)
```

### Compile Process

```python
# Check for syntax errors
errors = tm1.processes.compile(process_name='Update_Products')

if errors:
    print("Compilation errors:", errors)
else:
    print("Process compiled successfully")
```

### Delete Process

```python
tm1.processes.delete(process_name='Update_Products')
```

---

## Chores

### Create Chore

```python
from TM1py import Chore, ChoreTask, ChoreStartTime, ChoreFrequency
from datetime import datetime, time

# Create chore
chore = Chore(
    name='Daily_Refresh',
    start_time=ChoreStartTime(
        datetime=datetime(2024, 1, 1, 6, 0, 0)  # 6 AM
    ),
    dst_sensitivity=False,
    active=True,
    execution_mode='SingleCommit',
    frequency=ChoreFrequency(
        days=1,  # Daily
        hours=0,
        minutes=0,
        seconds=0
    )
)

# Add tasks
chore.add_task(ChoreTask(
    step=1,
    process_name='Load_Dimensions',
    parameters=[
        {'Name': 'pDate', 'Value': '20240101'}
    ]
))

chore.add_task(ChoreTask(
    step=2,
    process_name='Load_Data',
    parameters=[
        {'Name': 'pYear', 'Value': '2024'}
    ]
))

tm1.chores.create(chore)
```

### Read Chore

```python
# Get chore
chore = tm1.chores.get(chore_name='Daily_Refresh')

# Get all chore names
all_chores = tm1.chores.get_all_names()

# Check if exists
exists = tm1.chores.exists(chore_name='Daily_Refresh')

# Check if active
is_active = tm1.chores.is_active(chore_name='Daily_Refresh')
```

### Update Chore

```python
chore = tm1.chores.get('Daily_Refresh')

# Change frequency
chore.frequency = ChoreFrequency(hours=2)  # Every 2 hours

tm1.chores.update(chore)
```

### Activate/Deactivate Chore

```python
# Activate
tm1.chores.activate(chore_name='Daily_Refresh')

# Deactivate
tm1.chores.deactivate(chore_name='Daily_Refresh')
```

### Execute Chore

```python
# Run chore immediately
tm1.chores.execute(chore_name='Daily_Refresh')
```

### Delete Chore

```python
tm1.chores.delete(chore_name='Daily_Refresh')
```

---

## Views

### Create Native View

```python
from TM1py import NativeView, ViewAxisSelection, ViewTitleSelection

view = NativeView(
    cube_name='Sales',
    view_name='Revenue_Analysis'
)

# Define rows
view.add_row(
    dimension_name='Product',
    subset=tm1.subsets.get(
        dimension_name='Product',
        subset_name='AllProducts',
        private=False
    )
)

# Define columns
view.add_column(
    dimension_name='Period',
    subset=tm1.subsets.get(
        dimension_name='Period',
        subset_name='Months2024',
        private=False
    )
)

# Define titles (fixed dimensions)
view.add_title(
    dimension_name='Version',
    selection='Actual',
    subset=None
)

view.add_title(
    dimension_name='Measure',
    selection='Revenue',
    subset=None
)

tm1.cubes.views.create(view=view, private=False)
```

### Create MDX View

```python
from TM1py import MDXView

mdx = """
SELECT
  NON EMPTY [Product].[Product].Members ON ROWS,
  [Period].[2024].Children ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual])
"""

view = MDXView(
    cube_name='Sales',
    view_name='2024_Revenue',
    MDX=mdx
)

tm1.cubes.views.create(view=view, private=False)
```

### Read View

```python
# Get view
view = tm1.cubes.views.get(
    cube_name='Sales',
    view_name='Revenue_Analysis',
    private=False
)

# Get all view names
view_names = tm1.cubes.views.get_all_names(
    cube_name='Sales',
    private=False
)

# Check if exists
exists = tm1.cubes.views.exists(
    cube_name='Sales',
    view_name='Revenue_Analysis',
    private=False
)
```

### Update View

```python
view = tm1.cubes.views.get('Sales', 'Revenue_Analysis', private=False)

# Modify and update
# ... changes ...

tm1.cubes.views.update(view=view, private=False)
```

### Delete View

```python
tm1.cubes.views.delete(
    cube_name='Sales',
    view_name='Revenue_Analysis',
    private=False
)
```

---

## Subsets

### Create Static Subset

```python
from TM1py import Subset

subset = Subset(
    dimension_name='Product',
    hierarchy_name='Product',
    subset_name='TopProducts',
    elements=['Product1', 'Product2', 'Product3']
)

tm1.subsets.create(subset=subset, private=False)
```

### Create Dynamic Subset (MDX)

```python
subset = Subset(
    dimension_name='Product',
    hierarchy_name='Product',
    subset_name='HighValueProducts',
    expression="""
    {FILTER([Product].[Product].Members, [Product].[Product].CurrentMember.Properties("Price") > 100)}
    """
)

tm1.subsets.create(subset=subset, private=False)
```

### Read Subset

```python
# Get subset
subset = tm1.subsets.get(
    dimension_name='Product',
    subset_name='TopProducts',
    private=False
)

# Get elements in subset
elements = tm1.subsets.get_elements(
    dimension_name='Product',
    subset_name='TopProducts',
    private=False
)

# Check if exists
exists = tm1.subsets.exists(
    dimension_name='Product',
    subset_name='TopProducts',
    private=False
)
```

### Update Subset

```python
subset = tm1.subsets.get('Product', 'TopProducts', private=False)

# Modify elements
subset.elements = ['Product1', 'Product2', 'Product4', 'Product5']

tm1.subsets.update(subset=subset, private=False)
```

### Delete Subset

```python
tm1.subsets.delete(
    dimension_name='Product',
    subset_name='TopProducts',
    private=False
)
```

---

## Sandboxes

### Create Sandbox

```python
tm1.sandboxes.create(sandbox_name='Budget_2024')
```

### Read Sandbox

```python
# Get all sandboxes
sandboxes = tm1.sandboxes.get_all_names()

# Check if exists
exists = tm1.sandboxes.exists(sandbox_name='Budget_2024')

# Get sandbox details
sandbox = tm1.sandboxes.get(sandbox_name='Budget_2024')
```

### Merge Sandbox (Publish)

```python
# Merge sandbox to base
tm1.sandboxes.publish(sandbox_name='Budget_2024')

# Merge to another sandbox
tm1.sandboxes.publish(
    source_sandbox_name='Budget_2024',
    target_sandbox_names=['Approved_Budget']
)
```

### Delete Sandbox

```python
tm1.sandboxes.delete(sandbox_name='Budget_2024')
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

For API reference, see `API_REFERENCE.md`.  
For data operations, see `DATA_OPERATIONS.md`.  
For examples, see `EXAMPLES.md`.
