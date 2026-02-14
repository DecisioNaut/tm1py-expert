# TM1py API Reference

Complete reference for all TM1py services and their methods.

## Service Overview

All services are accessed via the `TM1Service` object:

```python
with TM1Service(**config) as tm1:
    tm1.<service>.<method>()
```

| Service | Purpose | Key Methods |
|---------|---------|-------------|
| `cells` | Read/write cube data | `execute_mdx`, `write`, `write_dataframe` |
| `cubes` | Manage cubes | `get`, `create`, `update`, `delete`, `exists` |
| `dimensions` | Manage dimensions | `get`, `create`, `update`, `delete`, `get_all_names` |
| `hierarchies` | Manage hierarchies | `get`, `create`, `update`, `exists` |
| `elements` | Manage elements | `get`, `create`, `add_edges`, `get_elements_dataframe` |
| `subsets` | Manage subsets | `get`, `create`, `update`, `execute_mdx` |
| `processes` | Execute TI processes | `execute_with_return`, `get`, `compile` |
| `chores` | Manage chores | `get`, `activate`, `deactivate`, `execute_chore` |
| `views` | Manage views | `get`, `create`, `update`, `delete`, `exists` |
| `sandboxes` | Manage sandboxes | `get`, `create`, `publish`, `reset`, `merge` |
| `security` | User/group management | `get_user`, `create_user`, `add_user_to_groups` |
| `server` | Server operations | `get_product_version`, `get_server_name`, `save_data` |
| `monitoring` | Monitor activity | `get_threads`, `get_sessions`, `cancel_thread` |

---

## CellService (`tm1.cells`)

### Reading Data

#### `execute_mdx(mdx, cell_properties=None, **kwargs)`
Execute MDX query and return cells with properties.

**Parameters:**
- `mdx` (str): Valid MDX query
- `cell_properties` (list): Properties to query ['Value', 'Ordinal', 'RuleDerived']
- `sandbox_name` (str): Sandbox name or None for base
- `skip_zeros` (bool): Skip zero values
- `skip_consolidated_cells` (bool): Skip C-level cells
- `skip_rule_derived_cells` (bool): Skip rule-calculated cells
- `element_unique_names` (bool): Return '[dim].[hier].[elem]' format

**Returns:** Dictionary with tuples as keys and cell properties as values

```python
data = tm1.cells.execute_mdx("SELECT ... FROM [Cube]")
# Returns: {('[Product].[P1]', '[Period].[2024]'): {'Value': 100}}
```

#### `execute_mdx_dataframe(mdx, **kwargs)`
Execute MDX and return pandas DataFrame.

**Parameters:**
- `mdx` (str): Valid MDX query  
- `skip_zeros` (bool): Exclude zeros
- `skip_consolidated_cells` (bool): Exclude consolidations
- `use_blob` (bool): Use blob (faster, requires admin)
- `use_iterative_json` (bool): Reduce memory usage
- `shaped` (bool): Preserve MDX shape in DataFrame

**Returns:** pandas DataFrame

```python
df = tm1.cells.execute_mdx_dataframe("SELECT ... FROM [Cube]")
```

#### `execute_mdx_csv(mdx, **kwargs)`
Execute MDX and return CSV string.

**Parameters:**
- Similar to `execute_mdx_dataframe`
- `line_separator` (str): Default '\r\n'
- `value_separator` (str): Default ','

**Returns:** CSV string

#### `execute_view(cube_name, view_name, private=False, **kwargs)`
Execute cube view and return cells.

**Parameters:**
- `cube_name` (str): Cube name
- `view_name` (str): View name
- `private` (bool): Private (True) or public view
- Other parameters same as `execute_mdx`

**Returns:** Dictionary of cells

#### `execute_view_dataframe(cube_name, view_name, private=False, **kwargs)`
Execute view and return DataFrame.

**Parameters:**
- Same as `execute_view`
- `use_blob` (bool): Better performance (requires admin)
- `shaped` (bool): Preserve view shape

**Returns:** pandas DataFrame

```python
df = tm1.cells.execute_view_dataframe(
    cube_name='Sales',
    view_name='Default',
    use_blob=True,
    skip_zeros=True
)
```

#### `get_value(cube_name, elements, dimensions=None, **kwargs)`
Get single cell value.

**Parameters:**
- `cube_name` (str): Cube name
- `elements` (str|iterable): 'Elem1,Elem2,Elem3' or ['Elem1', 'Elem2']
- `dimensions` (list): Dimension names in order
- `sandbox_name` (str): Sandbox or None

**Returns:** Cell value (str or float)

```python
value = tm1.cells.get_value(
    cube_name='Sales',
    elements='ProductA,2024-Q1,Revenue'
)
```

### Writing Data

#### `write_value(value, cube_name, element_tuple, **kwargs)`
Write single cell value.

**Parameters:**
- `value` (str|float): Value to write
- `cube_name` (str): Cube name
- `element_tuple` (iterable): Coordinate tuple
- `dimensions` (list): Optional dimension names
- `sandbox_name` (str): Sandbox or None

**Returns:** Response object

```python
tm1.cells.write_value(
    value=12345,
    cube_name='Sales',
    element_tuple=('ProductA', '2024-Q1', 'Revenue')
)
```

#### `write(cube_name, cellset_as_dict, dimensions=None, **kwargs)`
Write multiple cells efficiently.

**Parameters:**
- `cube_name` (str): Cube name
- `cellset_as_dict` (dict): {('Elem1', 'Elem2'): Value, ...}
- `dimensions` (list): Dimension names (improves performance)
- `increment` (bool): Increment instead of replace
- `deactivate_transaction_log` (bool): Disable logging temporarily
- `use_ti` (bool): Use TI for bulk write (requires admin)
- `use_blob` (bool): Use blob (fastest, requires admin)
- `sandbox_name` (str): Sandbox or None
- `skip_non_updateable` (bool): Skip rule cells
- `allow_spread` (bool): Allow proportional spread on C cells

**Returns:** Changeset ID or None

```python
cellset = {
    ('P1', '2024-Q1', 'Revenue'): 10000,
    ('P2', '2024-Q1', 'Revenue'): 20000
}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Measure'],
    use_ti=True
)
```

#### `write_dataframe(cube_name, data, dimensions=None, **kwargs)`
Write DataFrame to cube.

**Parameters:**
- `cube_name` (str): Cube name
- `data` (DataFrame): pandas DataFrame with cube structure
- `dimensions` (list): Optional dimension order
- Other parameters same as `write()`

**Returns:** Changeset ID

```python
import pandas as pd

df = pd.DataFrame({
    'Product': ['P1', 'P2'],
    'Period': ['2024-Q1', '2024-Q1'],
    'Measure': ['Revenue', 'Revenue'],
    'Value': [10000, 20000]
})

tm1.cells.write_dataframe(
    cube_name='Sales',
    data=df,
    use_blob=True
)
```

#### `write_async(cube_name, cells, max_workers=8, **kwargs)`
Write large datasets in parallel.

**Parameters:**
- `cube_name` (str): Cube name
- `cells` (dict): Cellset dictionary
- `max_workers` (int): Number of parallel threads
- `slice_size` (int): Cells per thread (default 250000)
- Other parameters same as `write()`

**Returns:** Changeset ID or None

```python
tm1.cells.write_async(
    cube_name='Sales',
    cells=large_cellset,
    max_workers=8,
    slice_size=250000
)
```

### Clearing Data

#### `clear(cube, **kwargs)`
Clear cube data using MDX expressions.

**Parameters:**
- `cube` (str): Cube name
- `**kwargs`: Dimension=MDX_expression pairs

```python
tm1.cells.clear(
    cube='Sales',
    product='{[Product].[P1], [Product].[P2]}',
    period='{[Period].[2024].Children}'
)
```

#### `clear_with_mdx(cube, mdx, **kwargs)`
Clear data based on MDX query.

**Parameters:**
- `cube` (str): Cube name
- `mdx` (str): Valid MDX query
- `sandbox_name` (str): Sandbox or None

```python
mdx = "SELECT {[Product].[P1]} ON 0 FROM [Sales]"
tm1.cells.clear_with_mdx(cube='Sales', mdx=mdx)
```

---

## DimensionService (`tm1.dimensions`)

#### `get(dimension_name, **kwargs)`
Get dimension object.

**Parameters:**
- `dimension_name` (str): Dimension name

**Returns:** Dimension object

```python
dim = tm1.dimensions.get('Product')
print(dim.hierarchy_names)
```

#### `create(dimension, **kwargs)`
Create new dimension.

**Parameters:**
- `dimension` (Dimension): Dimension object

**Returns:** Response

```python
from TM1py import Dimension, Hierarchy, Element

dim = Dimension(name='NewDim')
hierarchy = Hierarchy(name='NewDim', dimension_name='NewDim')
# Add elements...
dim.add_hierarchy(hierarchy)

tm1.dimensions.create(dim)
```

#### `update(dimension, **kwargs)`
Update existing dimension.

**Parameters:**
- `dimension` (Dimension): Updated dimension object
- `keep_existing_attributes` (bool): Preserve existing attributes

**Returns:** None

#### `delete(dimension_name, **kwargs)`
Delete dimension.

**Parameters:**
- `dimension_name` (str): Dimension name

**Returns:** Response

#### `exists(dimension_name, **kwargs)`
Check if dimension exists.

**Returns:** Boolean

#### `get_all_names(skip_control_dims=False, **kwargs)`
Get all dimension names.

**Parameters:**
- `skip_control_dims` (bool): Exclude } control dimensions

**Returns:** List of dimension names

```python
dims = tm1.dimensions.get_all_names(skip_control_dims=True)
```

---

## ElementService (`tm1.elements`)

#### `get_element_names(dimension_name, hierarchy_name, **kwargs)`
Get all element names.

**Parameters:**
- `dimension_name` (str): Dimension name
- `hierarchy_name` (str): Hierarchy name

**Returns:** List of element names

```python
elements = tm1.elements.get_element_names('Product', 'Product')
```

#### `get_leaf_element_names(dimension_name, hierarchy_name, **kwargs)`
Get leaf element names only.

**Returns:** List of leaf element names

#### `get_elements_dataframe(dimension_name, hierarchy_name, **kwargs)`
Get elements as DataFrame with attributes and hierarchy structure.

**Parameters:**
- `dimension_name` (str): Dimension name
- `hierarchy_name` (str): Hierarchy name
- `elements` (str|iterable): Element selection (MDX or list)
- `skip_consolidations` (bool): Exclude C elements
- `attributes` (iterable): Attribute names to include
- `skip_parents` (bool): Exclude parent columns
- `skip_weights` (bool): Exclude weight columns
- `use_blob` (bool): Better performance (requires admin)

**Returns:** pandas DataFrame

```python
df = tm1.elements.get_elements_dataframe(
    dimension_name='Product',
    hierarchy_name='Product',
    attributes=['Description', 'Category'],
    skip_consolidations=True
)
```

#### `create(dimension_name, hierarchy_name, element, **kwargs)`
Create new element.

**Parameters:**
- `dimension_name` (str): Dimension name
- `hierarchy_name` (str): Hierarchy name
- `element` (Element): Element object

**Returns:** Response

```python
from TM1py import Element

elem = Element(name='NewProduct', element_type='Numeric')
tm1.elements.create('Product', 'Product', elem)
```

#### `add_edges(dimension_name, hierarchy_name, edges, **kwargs)`
Add parent-child relationships.

**Parameters:**
- `dimension_name` (str): Dimension name
- `hierarchy_name` (str): Hierarchy name
- `edges` (dict): {(parent, child): weight, ...}

**Returns:** Response

```python
edges = {
    ('All Products', 'ProductA'): 1,
    ('All Products', 'ProductB'): 1
}

tm1.elements.add_edges('Product', 'Product', edges)
```

#### `delete(dimension_name, hierarchy_name, element_name, **kwargs)`
Delete element.

**Parameters:**
- `dimension_name` (str): Dimension name
- `hierarchy_name` (str): Hierarchy name
- `element_name` (str): Element name

**Returns:** Response

#### `get_element_types(dimension_name, hierarchy_name, **kwargs)`
Get element types for all elements.

**Parameters:**
- `skip_consolidations` (bool): Exclude C elements

**Returns:** Dictionary {element_name: type}

```python
types = tm1.elements.get_element_types('Product', 'Product')
# {'ProductA': 'Numeric', 'AllProducts': 'Consolidated'}
```

---

## CubeService (`tm1.cubes`)

#### `get(cube_name, **kwargs)`
Get cube object.

**Parameters:**
- `cube_name` (str): Cube name

**Returns:** Cube object

```python
cube = tm1.cubes.get('Sales')
print(cube.dimensions)
print(cube.has_rules)
```

#### `create(cube, **kwargs)`
Create new cube.

**Parameters:**
- `cube` (Cube): Cube object

**Returns:** Response

```python
from TM1py import Cube

cube = Cube(
    name='NewCube',
    dimensions=['Dim1', 'Dim2', 'Measure']
)

tm1.cubes.create(cube)
```

#### `get_all_names(skip_control_cubes=False, **kwargs)`
Get all cube names.

**Parameters:**
- `skip_control_cubes` (bool): Exclude } control cubes

**Returns:** List of cube names

#### `exists(cube_name, **kwargs)`
Check if cube exists.

**Returns:** Boolean

#### `get_dimension_names(cube_name, **kwargs)`
Get dimensions of a cube in order.

**Parameters:**
- `cube_name` (str): Cube name
- `skip_sandbox_dimension` (bool): Exclude sandbox dimension

**Returns:** List of dimension names

```python
dims = tm1.cubes.get_dimension_names('Sales')
# ['Product', 'Period', 'Measure']
```

#### `search_for_dimension(dimension_name, skip_control_cubes=False, **kwargs)`
Find cubes containing a dimension.

**Parameters:**
- `dimension_name` (str): Dimension name
- `skip_control_cubes` (bool): Exclude control cubes

**Returns:** List of cube names

```python
cubes = tm1.cubes.search_for_dimension('Product')
# ['Sales', 'Inventory', 'Prices']
```

---

## ProcessService (`tm1.processes`)

#### `execute_with_return(process_name, timeout=None, **kwargs)`
Execute process and return success status.

**Parameters:**
- `process_name` (str): Process name
- `timeout` (float): Timeout in seconds
- `cancel_at_timeout` (bool): Cancel if timeout reached
- `**kwargs`: Process parameters as keyword arguments

**Returns:** Tuple (success: bool, status: str, error_log: str)

```python
success, status, error_log = tm1.processes.execute_with_return(
    process_name='LoadData',
    pYear='2024',
    pRegion='EMEA',
    timeout=300
)

if success:
    print(f"Completed: {status}")
else:
    print(f"Failed: {error_log}")
```

#### `execute_ti_code(lines_prolog, lines_epilog=None, **kwargs)`
Execute loose TI code.

**Parameters:**
- `lines_prolog` (list): List of TI statements for prolog
- `lines_epilog` (list): List of TI statements for epilog

**Returns:** Response

```python
prolog = [
    "sYear = '2024';",
    "TextOutput('TM1ProcessError.log', sYear);"
]

tm1.processes.execute_ti_code(lines_prolog=prolog)
```

#### `get(name_process, **kwargs)`
Get process object.

**Parameters:**
- `name_process` (str): Process name

**Returns:** Process object

```python
process = tm1.processes.get('LoadData')
print(process.prolog_procedure)
```

#### `compile(name, **kwargs)`
Compile process and return syntax errors.

**Parameters:**
- `name` (str): Process name

**Returns:** List of syntax errors (empty if no errors)

```python
errors = tm1.processes.compile('LoadData')
if errors:
    print(f"Syntax errors: {errors}")
```

#### `get_all_names(skip_control_processes=False, **kwargs)`
Get all process names.

**Parameters:**
- `skip_control_processes` (bool): Exclude } and { processes

**Returns:** List of process names

---

## ChoreService (`tm1.chores`)

#### `get(chore_name, **kwargs)`
Get chore object.

**Returns:** Chore object

#### `execute_chore(chore_name, **kwargs)`
Execute chore immediately.

**Parameters:**
- `chore_name` (str): Chore name

**Returns:** Response

```python
response = tm1.chores.execute_chore('DailyLoad')
```

#### `activate(chore_name, **kwargs)`
Activate chore.

**Returns:** Response

#### `deactivate(chore_name, **kwargs)`
Deactivate chore.

**Returns:** Response

#### `get_all_names(**kwargs)`
Get all chore names.

**Returns:** List of chore names

---

## ViewService (`tm1.views`)

#### `get(cube_name, view_name, private=False, **kwargs)`
Get view object (MDXView or NativeView).

**Parameters:**
- `cube_name` (str): Cube name
- `view_name` (str): View name
- `private` (bool): Private or public view

**Returns:** View object

```python
view = tm1.views.get('Sales', 'Default', private=False)
```

#### `create(view, private=False, **kwargs)`
Create new view.

**Parameters:**
- `view` (MDXView|NativeView): View object
- `private` (bool): Create as private view

**Returns:** Response

#### `exists(cube_name, view_name, private=None, **kwargs)`
Check if view exists.

**Parameters:**
- `private` (bool|None): Check private, public, or both

**Returns:** Boolean or tuple (private_exists, public_exists)

```python
exists = tm1.views.exists('Sales', 'MyView', private=True)
```

---

## SandboxService (`tm1.sandboxes`)

#### `get(sandbox_name, **kwargs)`
Get sandbox object.

**Returns:** Sandbox object

#### `create(sandbox, **kwargs)`
Create new sandbox.

**Parameters:**
- `sandbox` (Sandbox): Sandbox object

**Returns:** Response

```python
from TM1py import Sandbox

sandbox = Sandbox(name='TestScenario')
tm1.sandboxes.create(sandbox)
```

#### `publish(sandbox_name, **kwargs)`
Publish sandbox to base.

**Parameters:**
- `sandbox_name` (str): Sandbox name

**Returns:** Response

```python
tm1.sandboxes.publish('TestScenario')
```

#### `reset(sandbox_name, **kwargs)`
Reset sandbox (clear all changes).

**Returns:** Response

#### `merge(source_sandbox_name, target_sandbox_name, **kwargs)`
Merge one sandbox into another.

**Parameters:**
- `source_sandbox_name` (str): Source sandbox
- `target_sandbox_name` (str): Target sandbox
- `clean_after` (bool): Reset source after merge

**Returns:** Response

---

## SecurityService (`tm1.security`)

#### `get_current_user(**kwargs)`
Get current user object.

**Returns:** User object

```python
user = tm1.security.get_current_user()
print(f"User: {user.name}, Groups: {user.groups}")
```

#### `get_user(user_name, **kwargs)`
Get user object.

**Parameters:**
- `user_name` (str): User name

**Returns:** User object

#### `get_all_user_names(**kwargs)`
Get all user names.

**Returns:** List of user names

#### `add_user_to_groups(user_name, groups, **kwargs)`
Add user to multiple groups.

**Parameters:**
- `user_name` (str): User name
- `groups` (iterable): List of group names

**Returns:** Response

```python
tm1.security.add_user_to_groups('john.doe', ['DataAdmin', 'Readers'])
```

---

## ServerService (`tm1.server`)

#### `get_product_version(**kwargs)`
Get TM1 server version.

**Returns:** Version string

```python
version = tm1.server.get_product_version()
# '12.0.0'
```

#### `get_server_name(**kwargs)`
Get TM1 server name.

**Returns:** Server name string

#### `save_data(**kwargs)`
Save all changes to disk.

**Returns:** Response

```python
tm1.server.save_data()
```

#### `get_message_log_entries(**kwargs)`
Get message log entries.

**Parameters:**
- `reverse` (bool): Reverse order
- `since` (datetime): Start time
- `until` (datetime): End time
- `top` (int): Limit results
- `level` (str): ERROR, WARNING, INFO, DEBUG

**Returns:** Dictionary of log entries

```python
logs = tm1.server.get_message_log_entries(
    level='ERROR',
    top=100,
    reverse=True
)
```

---

## MonitoringService (`tm1.monitoring`)

#### `get_threads(**kwargs)`
Get all active threads.

**Returns:** List of thread dictionaries

```python
threads = tm1.monitoring.get_threads()
for thread in threads:
    print(f"ID: {thread['ID']}, Function: {thread['Function']}")
```

#### `get_active_users(**kwargs)`
Get list of active users.

**Returns:** List of User objects

#### `cancel_thread(thread_id, **kwargs)`
Cancel a running thread.

**Parameters:**
- `thread_id` (int): Thread ID

**Returns:** Response

```python
tm1.monitoring.cancel_thread(12345)
```

---

## Common Parameters

Many methods share these common optional parameters:

- `**kwargs`: Captures additional HTTP request parameters
- `timeout` (float): Request timeout in seconds
- `cancel_at_timeout` (bool): Cancel TM1 operation if timeout reached
- `async_requests_mode` (bool): Use async mode (IBM Cloud)
- `sandbox_name` (str): Execute in sandbox context

## Return Types

- **Response**: HTTP response object from requests library
- **Dict**: Python dictionary
- **DataFrame**: pandas DataFrame (when pandas installed)
- **List**: Python list
- **Boolean**: True/False
- **Object**: TM1py object (Dimension, Cube, Process, etc.)
- **String**: Text value

## Error Handling

TM1py raises these exceptions:

- `TM1pyException`: Base exception
- `TM1pyRestException`: REST API errors (includes status code, reason)
- `TM1pyTimeout`: Request timeout
- `TM1pyVersionException`: TM1 version not supported
- `TM1pyNotAdminException`: Admin rights required
- `TM1pyWriteFailureException`: Write operation failed
- `TM1pyWritePartialFailureException`: Partial write failure

Always handle exceptions:

```python
from TM1py.Exceptions import TM1pyRestException

try:
    tm1.cubes.get('NonExistentCube')
except TM1pyRestException as e:
    print(f"Error {e.status_code}: {e.reason}")
```

---

For detailed examples and use cases, see `EXAMPLES.md`.  
For performance optimization, see `PERFORMANCE.md`.  
For all connection patterns, see `CONNECTION_GUIDE.md`.
