# TM1py Data Operations

Comprehensive guide to reading and writing data in TM1 using TM1py.

## Contents
- [Reading Data](#reading-data)
  - [1. Execute MDX Queries](#1-execute-mdx-queries)
  - [2. Execute Views](#2-execute-views)
  - [3. Get Individual Cell Values](#3-get-individual-cell-values)
  - [4. Performance Options for Reading](#4-performance-options-for-reading)
  - [5. Query Cell Count Before Execution](#5-query-cell-count-before-execution)
  - [6. Read with Dimensions and Attributes](#6-read-with-dimensions-and-attributes)
- [Writing Data](#writing-data)
  - [1. Write Single Cell Value](#1-write-single-cell-value)
  - [2. Write Multiple Cells](#2-write-multiple-cells-cellset)
  - [3. Write DataFrame](#3-write-dataframe)
  - [4. Async/Parallel Writes](#4-asyncparallel-writes)
  - [5. Write from View/MDX Results](#5-write-from-viewmdx-results)
  - [6. Incremental Writes](#6-incremental-writes)
  - [7. Write with Transaction Log Control](#7-write-with-transaction-log-control)
- [Clearing Data](#clearing-data)
- [Data Validation](#data-validation)
- [Data Transformations](#data-transformations)
- [Advanced Patterns](#advanced-patterns)
- [Error Handling](#error-handling)

---

## Reading Data

### 1. Execute MDX Queries

MDX (Multi-Dimensional Expressions) is the primary query language for TM1 cubes.

#### Basic MDX Execution

```python
# Returns raw cellset object
cellset = tm1.cells.execute_mdx(mdx="SELECT ... FROM [Cube]")

# Returns pandas DataFrame (recommended)
df = tm1.cells.execute_mdx_dataframe(
    mdx="SELECT ... FROM [Cube]"
)

# Returns CSV string
csv = tm1.cells.execute_mdx_csv(
    mdx="SELECT ... FROM [Cube]"
)

# Returns raw JSON
json_data = tm1.cells.execute_mdx_raw(
    mdx="SELECT ... FROM [Cube]"
)
```

#### MDX Examples

```python
# Simple query
mdx = """
SELECT
  [Product].[Product].Members ON ROWS,
  [Period].[Period].Members ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual])
"""

df = tm1.cells.execute_mdx_dataframe(mdx)
```

```python
# Query with filtering
mdx = """
SELECT
  NON EMPTY {[Product].[Product1], [Product].[Product2]} ON ROWS,
  {[Period].[2024-01]:[Period].[2024-12]} ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual], [Region].[US])
"""

df = tm1.cells.execute_mdx_dataframe(mdx)
```

```python
# Top N query
mdx = """
SELECT TOP 10
  ORDER([Product].[Product].Members, [Measure].[Revenue], DESC) ON ROWS,
  {[Measure].[Revenue], [Measure].[Units]} ON COLUMNS
FROM [Sales]
WHERE ([Period].[2024-Q1], [Version].[Actual])
"""

top_products = tm1.cells.execute_mdx_dataframe(mdx)
```

#### MDX with Parameters

```python
def get_sales_data(tm1, product, period, version='Actual'):
    """Parameterized MDX query"""
    
    mdx = f"""
    SELECT
      [Region].[Region].Members ON ROWS,
      {{[Measure].[Revenue], [Measure].[Units]}} ON COLUMNS
    FROM [Sales]
    WHERE ([Product].[{product}], [Period].[{period}], [Version].[{version}])
    """
    
    return tm1.cells.execute_mdx_dataframe(mdx)

# Usage
df = get_sales_data(tm1, product='Product1', period='2024-Q1')
```

### 2. Execute Views

Views are pre-defined queries saved in TM1.

#### Execute Native View

```python
# Returns DataFrame
df = tm1.cells.execute_view_dataframe(
    cube_name='Sales',
    view_name='MonthlyRevenue',
    private=False  # False for public views, True for private
)

# Returns raw cellset
cellset = tm1.cells.execute_view(
    cube_name='Sales',
    view_name='MonthlyRevenue',
    private=False
)

# Returns CSV
csv = tm1.cells.execute_view_csv(
    cube_name='Sales',
    view_name='MonthlyRevenue',
    private=False
)
```

#### Execute MDX View

```python
df = tm1.cells.execute_view_dataframe(
    cube_name='Sales',
    view_name='CustomQuery',  # MDX view
    private=True
)
```

### 3. Get Individual Cell Values

```python
# Get single cell value
value = tm1.cells.get_value(
    cube_name='Sales',
    element_string='Product1,2024-Q1,Actual,Revenue'
)

# Alternative tuple format
value = tm1.cells.get_value(
    cube_name='Sales',
    elements=('Product1', '2024-Q1', 'Actual', 'Revenue')
)

# With sandbox
value = tm1.cells.get_value(
    cube_name='Sales',
    element_string='Product1,2024-Q1,Actual,Revenue',
    sandbox_name='MyGoodbye'
)
```

### 4. Performance Options for Reading

#### Use Blob (Fastest)

```python
# Requires ADMIN rights, 40-100% faster
df = tm1.cells.execute_mdx_dataframe(
    mdx=query,
    use_blob=True  # Significant performance improvement
)
```

#### Skip Unnecessary Data

```python
df = tm1.cells.execute_mdx_dataframe(
    mdx=query,
    skip_zeros=True,  # Exclude cells with 0 values
    skip_consolidated_cells=True,  # Exclude C-level elements
    skip_rule_derived_cells=True,  # Exclude cells calculated by rules
    use_compact_json=True  # Reduce JSON payload size
)
```

#### Iterative JSON (Memory Efficient)

```python
# For very large datasets, reduces memory by 50-70%
df = tm1.cells.execute_mdx_dataframe(
    mdx=large_query,
    use_iterative_json=True
)
```

#### Async Execution

```python
# Execute query asynchronously
async_id = tm1.cells.execute_mdx_async(
    mdx=query,
    max_workers=4  # Number of parallel workers
)

# Poll for results
import time
while True:
    try:
        df = tm1.cells.poll_execute_mdx_async(async_id)
        break
    except:
        time.sleep(1)
```

### 5. Query Cell Count Before Execution

```python
# Check how many cells will be returned
cell_count = tm1.cells.execute_mdx_cellcount(mdx=query)

print(f"Query will return {cell_count:,} cells")

# Decide read method based on size
if cell_count > 1_000_000:
    df = tm1.cells.execute_mdx_dataframe(mdx=query, use_blob=True)
elif cell_count > 100_000:
    df = tm1.cells.execute_mdx_dataframe(mdx=query, use_iterative_json=True)
else:
    df = tm1.cells.execute_mdx_dataframe(mdx=query)
```

### 6. Read with Dimensions and Attributes

```python
# Include element attributes in results
mdx = """
SELECT
  [Product].[Product].Members ON ROWS,
  [Period].[Period].Members ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue])
"""

df = tm1.cells.execute_mdx_dataframe(
    mdx=mdx,
    include_attributes=True  # Include all attributes
)

# Manually specify attribute columns
df = tm1.cells.execute_mdx_dataframe(
    mdx=mdx,
    element_attributes=[
        ('Product', ['Category', 'Status']),
        ('Period', ['Year', 'Quarter'])
    ]
)
```

---

## Writing Data

### 1. Write Single Cell Value

```python
# Write one cell
tm1.cells.write_value(
    value=50000,
    cube_name='Sales',
    element_tuple=('Product1', '2024-Q1', 'Actual', 'Revenue')
)

# Write to sandbox
tm1.cells.write_value(
    value=60000,
    cube_name='Sales',
    element_tuple=('Product1', '2024-Q1', 'Plan', 'Revenue'),
    sandbox_name='Budget_2024'
)
```

### 2. Write Multiple Cells (Cellset)

```python
# Write cellset as dictionary
cellset = {
    ('Product1', '2024-Q1', 'Actual', 'Revenue'): 50000,
    ('Product1', '2024-Q1', 'Actual', 'Units'): 1000,
    ('Product2', '2024-Q1', 'Actual', 'Revenue'): 75000,
    ('Product2', '2024-Q1', 'Actual', 'Units'): 1500
}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure']  # Specify order
)
```

#### Write Options

```python
tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure'],
    
    # Performance options
    use_blob=True,  # Fastest (requires ADMIN)
    use_ti=True,  # Fast (requires ADMIN, uses TI process)
    
    # Behavior options
    increment=False,  # Replace values (default: False)
    sandbox_name=None,  # Write to specific sandbox
    
    # Data quality options
    skip_non_updateable=True,  # Skip rule-calculated cells
    precision=15,  # Decimal precision for TI writes
    
    # Transaction log
    deactivate_transaction_log=False,  # Disable logging (for bulk loads)
    reactivate_transaction_log=False  # Re-enable after write
)
```

### 3. Write DataFrame

```python
import pandas as pd

# DataFrame with cube dimensions as columns
df = pd.DataFrame({
    'Product': ['Product1', 'Product1', 'Product2', 'Product2'],
    'Period': ['2024-Q1', '2024-Q2', '2024-Q1', '2024-Q2'],
    'Version': ['Actual', 'Actual', 'Actual', 'Actual'],
    'Measure': ['Revenue', 'Revenue', 'Revenue', 'Revenue'],
    'Value': [50000, 55000, 75000, 80000]
})

tm1.cells.write_dataframe(
    cube_name='Sales',
    data=df
)
```

#### DataFrame Write Options

```python
tm1.cells.write_dataframe(
    cube_name='Sales',
    data=df,
    
    # Performance
    use_blob=True,  # Fastest
    use_ti=True,  # Fast alternative
    
    # Column mapping
    dimension_mapping={  # Map DataFrame columns to cube dimensions
        'Product': 'ProductCode',
        'Period': 'MonthName'
    },
    
    # Behavior
    increment=False,  # Replace instead of add
    skip_non_updateable=True,  # Skip rule cells
    sum_numeric_duplicates=True,  # Aggregate duplicate coordinates
    
    sandbox_name=None  # Target sandbox
)
```

### 4. Async/Parallel Writes

For very large cellsets:

```python
massive_cellset = {...}  # Millions of cells

changeset = tm1.cells.write_async(
    cube_name='Sales',
    cells=massive_cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure'],
    max_workers=8,  # Parallel threads
    slice_size=250000,  # Cells per thread
    increment=False,
    deactivate_transaction_log=True,
    reactivate_transaction_log=True
)

print(f"Write changeset: {changeset}")
```

### 5. Write from View/MDX Results

```python
# Read from one cube, write to another
df = tm1.cells.execute_mdx_dataframe(
    mdx="SELECT ... FROM [SourceCube]"
)

# Transform data
df['Value'] = df['Value'] * 1.1  # Apply 10% increase

# Write to target
tm1.cells.write_dataframe(
    cube_name='TargetCube',
    data=df,
    use_blob=True
)
```

### 6. Incremental Writes

```python
# Add to existing values instead of replacing
cellset = {
    ('Product1', '2024-Q1', 'Actual', 'Revenue'): 5000  # Add $5000
}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure'],
    increment=True  # Add to existing value
)
```

### 7. Write with Transaction Log Control

For massive one-time loads:

```python
# Disable transaction log temporarily for performance
tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=huge_cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure'],
    deactivate_transaction_log=True,  # Disable before write
    reactivate_transaction_log=True,  # Re-enable after
    use_blob=True
)
```

---

## Clearing Data

### 1. Clear Cube Data

```python
# Clear entire cube
tm1.cells.clear_cube(cube_name='Sales')
```

### 2. Clear with MDX

```python
# Clear specific slice
mdx = """
SELECT
  [Product].[Product].Members ON ROWS,
  [Period].[2024].Children ON COLUMNS
FROM [Sales]
WHERE ([Version].[Forecast])
"""

tm1.cells.clear_with_mdx(cube_name='Sales', mdx=mdx)
```

### 3. Clear Specific Cells

```python
# Zero out specific cells
cellset_to_clear = {
    ('Product1', '2024-Q1', 'Forecast', 'Revenue'): 0,
    ('Product2', '2024-Q1', 'Forecast', 'Revenue'): 0
}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset_to_clear,
    dimensions=['Product', 'Period', 'Version', 'Measure']
)
```

---

## Data Validation

### 1. Check Values Before Writing

```python
import pandas as pd

def validate_data(df, cube_name, tm1):
    """Validate DataFrame before writing to TM1"""
    
    errors = []
    
    # Check for nulls
    if df.isnull().any().any():
        null_cols = df.columns[df.isnull().any()].tolist()
        errors.append(f"Null values found in columns: {null_cols}")
    
    # Check for duplicates
    dim_cols = [c for c in df.columns if c != 'Value']
    duplicates = df[df.duplicated(subset=dim_cols, keep=False)]
    if not duplicates.empty:
        errors.append(f"Found {len(duplicates)} duplicate coordinates")
    
    # Validate cube dimensions
    cube_dims = tm1.cubes.get_dimension_names(cube_name)
    expected_cols = cube_dims + ['Value']
    missing = set(expected_cols) - set(df.columns)
    if missing:
        errors.append(f"Missing columns: {missing}")
    
    # Check element existence (sample)
    for dim in cube_dims[:2]:  # Check first 2 dimensions
        sample_values = df[dim].unique()[:5]
        for val in sample_values:
            if not tm1.elements.exists(dim, dim, str(val)):
                errors.append(f"Element '{val}' not found in dimension '{dim}'")
                break
    
    if errors:
        raise ValueError("\n".join(errors))
    
    return True

# Usage
try:
    validate_data(df, 'Sales', tm1)
    tm1.cells.write_dataframe('Sales', df, use_blob=True)
except ValueError as e:
    print(f"Validation failed:\n{e}")
```

### 2. Verify Write Success

```python
# Write data
cellset = {('Product1', '2024-Q1', 'Actual', 'Revenue'): 50000}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Version', 'Measure']
)

# Verify
written_value = tm1.cells.get_value(
    cube_name='Sales',
    element_string='Product1,2024-Q1,Actual,Revenue'
)

assert written_value == 50000, f"Expected 50000, got {written_value}"
print("✓ Write verified")
```

---

## Data Transformations

### 1. Pivot DataFrame for TM1

```python
# Source data (wide format)
df_wide = pd.DataFrame({
    'Product': ['Product1', 'Product2'],
    '2024-Q1': [50000, 75000],
    '2024-Q2': [55000, 80000]
})

# Transform to TM1 format (long format)
df_long = df_wide.melt(
    id_vars=['Product'],
    var_name='Period',
    value_name='Value'
)

# Add other dimensions
df_long['Version'] = 'Actual'
df_long['Measure'] = 'Revenue'

# Write to TM1
tm1.cells.write_dataframe('Sales', df_long, use_blob=True)
```

### 2. Aggregate Data Before Writing

```python
# Source data with duplicates
df_raw = pd.read_csv('transactions.csv')

# Aggregate by dimensions
df_agg = df_raw.groupby(['Product', 'Period', 'Version', 'Measure'])['Value'].sum().reset_index()

# Write aggregated data
tm1.cells.write_dataframe('Sales', df_agg, use_blob=True)
```

### 3. Apply Business Logic

```python
# Read current data
df = tm1.cells.execute_view_dataframe('Sales', 'CurrentActuals', private=False)

# Apply transformations
df['Value'] = df['Value'] * 1.1  # 10% increase
df['Version'] = 'Forecast'  # Change version
df['Period'] = df['Period'].str.replace('2024', '2025')  # Next year

# Write transformed data
tm1.cells.write_dataframe('Sales', df, use_blob=True)
```

---

## Advanced Patterns

### 1. Conditional Writing

```python
def conditional_write(tm1, cube_name, cellset, dimensions, threshold=0):
    """Only write cells above threshold"""
    
    filtered_cellset = {
        coord: value 
        for coord, value in cellset.items()
        if abs(value) > threshold
    }
    
    print(f"Writing {len(filtered_cellset):,} of {len(cellset):,} cells")
    
    tm1.cells.write(
        cube_name=cube_name,
        cellset_as_dict=filtered_cellset,
        dimensions=dimensions,
        use_blob=True
    )

# Usage: Only write non-zero values
conditional_write(tm1, 'Sales', my_cellset, dimensions, threshold=0)
```

### 2. Upsert Pattern

```python
def upsert_data(tm1, cube_name, new_data_df):
    """Update existing data or insert new data"""
    
    # Read existing data
    existing_df = tm1.cells.execute_view_dataframe(
        cube_name=cube_name,
        view_name='AllData',
        private=False
    )
    
    # Merge on dimensions
    dim_cols = [c for c in new_data_df.columns if c != 'Value']
    
    merged = existing_df.merge(
        new_data_df,
        on=dim_cols,
        how='outer',
        suffixes=('_old', '_new')
    )
    
    # Take new values where available, else keep old
    merged['Value'] = merged['Value_new'].fillna(merged['Value_old'])
    
    # Write back
    result_df = merged[dim_cols + ['Value']]
    tm1.cells.write_dataframe(cube_name, result_df, use_blob=True)

# Usage
new_data = pd.DataFrame(...)
upsert_data(tm1, 'Sales', new_data)
```

### 3. Copy Data Between Cubes

```python
def copy_cube_data(tm1, source_cube, target_cube, mdx_filter=None, transform_func=None):
    """Copy data from one cube to another with optional transformation"""
    
    # Build MDX query
    if mdx_filter:
        mdx = mdx_filter
    else:
        dims = tm1.cubes.get_dimension_names(source_cube)
        mdx = f"SELECT {{TM1SubsetAll([{dims[0]}])}} ON 0 FROM [{source_cube}]"
    
    # Read source
    df = tm1.cells.execute_mdx_dataframe(
        mdx=mdx,
        use_blob=True,
        skip_zeros=True,
        skip_consolidated_cells=True
    )
    
    # Apply transformation if provided
    if transform_func:
        df = transform_func(df)
    
    # Write to target
    tm1.cells.write_dataframe(
        cube_name=target_cube,
        data=df,
        use_blob=True
    )
    
    return len(df)

# Usage
def scale_values(df):
    df['Value'] = df['Value'] * 0.9  # 10% reduction
    return df

rows_copied = copy_cube_data(
    tm1=tm1,
    source_cube='Actuals',
    target_cube='Budget',
    transform_func=scale_values
)
```

---

## Error Handling

### Handling Write Errors

```python
from requests.exceptions import TM1pyException

try:
    tm1.cells.write_dataframe('Sales', df, use_blob=True)
except TM1pyException as e:
    if 'rule' in str(e).lower():
        print("Error: Attempted to write to rule-calculated cells")
        # Retry with skip_non_updateable=True
        tm1.cells.write_dataframe(
            'Sales', df, 
            use_blob=True, 
            skip_non_updateable=True
        )
    elif 'element' in str(e).lower():
        print("Error: Invalid element in data")
        # Log and skip invalid rows
    else:
        raise
```

---

For API reference, see `API_REFERENCE.md`.  
For performance optimization, see `PERFORMANCE.md`.  
For examples, see `EXAMPLES.md`.
