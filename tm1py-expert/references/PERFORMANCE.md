# TM1py Performance Optimization

Guide to optimizing TM1py operations for speed and efficiency.

## Contents
- [Performance Priorities](#performance-priorities)
- [Reading Data Performance](#reading-data-performance)
- [Writing Data Performance](#writing-data-performance)
- [Connection Resilience](#connection-resilience-tm1py-21)
- [Metadata Operations](#metadata-operations)
- [MDX Query Optimization](#mdx-query-optimization)
- [Connection Optimization](#connection-optimization)
- [Process Execution](#process-execution)
- [Memory Optimization](#memory-optimization)
- [Benchmarking](#benchmarking)
- [Performance Checklist](#performance-checklist)
- [Common Performance Issues](#common-performance-issues)

## Performance Priorities

1. **Use blob operations** when available (requires ADMIN rights)
2. **Leverage async/parallel operations** for bulk work
3. **Filter data early** (skip zeros, consolidations, rule cells)
4. **Specify dimensions explicitly** in cell operations
5. **Use TI processes** for bulk writes
6. **Minimize round trips** to TM1 server

---

## Reading Data Performance

### Fastest Read Methods (Ranked)

| Method | Performance | Requirements | Use When |
|--------|-------------|--------------|----------|
| `execute_view_dataframe(..., use_blob=True)` | ★★★★★ | Admin rights | Large datasets (>100K cells) |
| `execute_mdx_dataframe(..., use_blob=True)` | ★★★★★ | Admin rights | Large MDX queries |
| `execute_view_dataframe(..., use_iterative_json=True)` | ★★★★☆ | None | Memory constrained |
| `execute_mdx_dataframe()` | ★★★☆☆ | None | Standard queries |
| `execute_mdx_async()` | ★★★★☆ | None | Parallel reading |
| `execute_mdx()` | ★★☆☆☆ | None | Small queries |

### Blob Operations

Blob operations provide 40-100% performance improvement:

```python
# Standard read (slower)
df = tm1.cells.execute_view_dataframe(
    cube_name='LargeCube',
    view_name='DataView',
    private=False
)

# Blob read (faster, requires admin)
df = tm1.cells.execute_view_dataframe(
    cube_name='LargeCube',
    view_name='DataView',
    private=False,
    use_blob=True  # Significant performance gain
)
```

### Skip Unnecessary Data

```python
# Optimize by skipping unwanted data
df = tm1.cells.execute_mdx_dataframe(
    mdx=query,
    skip_zeros=True,  # Exclude zero values
    skip_consolidated_cells=True,  # Exclude C-level
    skip_rule_derived_cells=True,  # Exclude rule cells
    use_compact_json=True  # Smaller payload
)
```

### Async/Parallel Reading

```python
# Read multiple queries in parallel
from concurrent.futures import ThreadPoolExecutor
import pandas as pd

queries = {
    '2024-Q1': "SELECT... WHERE ([Period].[2024-Q1])",
    '2024-Q2': "SELECT... WHERE ([Period].[2024-Q2])",
    '2024-Q3': "SELECT... WHERE ([Period].[2024-Q3])",
    '2024-Q4': "SELECT... WHERE ([Period].[2024-Q4])"
}

def read_data(tm1, name, mdx):
    df = tm1.cells.execute_mdx_dataframe(mdx, use_blob=True)
    df['Period'] = name
    return df

results = []
with TM1Service(**params) as tm1:
    with ThreadPoolExecutor(max_workers=4) as executor:
        futures = [
            executor.submit(read_data, tm1, name, mdx)
            for name, mdx in queries.items()
        ]
        results = [f.result() for f in futures]

combined_df = pd.concat(results, ignore_index=True)
```

### Iterative JSON for Memory Efficiency

```python
# For very large datasets with memory constraints
df = tm1.cells.execute_mdx_dataframe(
    mdx=large_query,
    use_iterative_json=True  # Reduces memory footprint 50-70%
)
# Note: 3-5% slower but much lower memory usage
```

---

## Writing Data Performance

### Fastest Write Methods (Ranked)

| Method | Performance | Requirements | Use When |
|--------|-------------|--------------|----------|
| `write(..., use_blob=True)` | ★★★★★ | Admin rights | Bulk writes (>50K cells) |
| `write(..., use_ti=True)` | ★★★★☆ | Admin rights | Large writes |
| `write_async()` | ★★★★☆ | None | Parallel processing |
| `write_dataframe(..., use_ti=True)` | ★★★★☆ | Admin rights | DataFrame writes |
| `write()` | ★★★☆☆ | None | Standard writes |
| `write_value()` | ★☆☆☆☆ | None | Single values only |

### Blob Writes (Fastest)

```python
# Blob write - 10x faster than TI, requires admin
cellset = {
    ('P1', '2024-Q1', 'Value'): 1000,
    ('P2', '2024-Q1', 'Value'): 2000,
    # ... thousands more cells
}

tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=cellset,
    dimensions=['Product', 'Period', 'Measure'],  # Specify for performance
    use_blob=True  # Requires ADMIN
)
```

### TI Process Writes

```python
# TI write - faster than  standard, requires admin
tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=large_cellset,
    dimensions=['Product', 'Period', 'Measure'],
    use_ti=True,  # Uses unbound TI process
    precision=15  # Max decimal places to avoid TI errors with large numbers
)
```

### Async Parallel Writes

```python
# Write massive datasets in parallel
huge_cellset = {...}  # Millions of cells

changeset = tm1.cells.write_async(
    cube_name='Sales',
    cells=huge_cellset,
    dimensions=['Product', 'Period', 'Measure'],
    max_workers=8,  # Number of parallel threads
    slice_size=250000,  # Cells per thread
    deactivate_transaction_log=True,  # Disable logging temporarily
    reactivate_transaction_log=True  # Re-enable after
)

print(f"Write completed with changeset: {changeset}")
```

### DataFrame Writes

```python
import pandas as pd

# Efficient DataFrame write
df = pd.DataFrame({...})  # Large DataFrame

tm1.cells.write_dataframe(
    cube_name='Sales',
    data=df,
    use_blob=True,  # or use_ti=True
    skip_non_updateable=True,  # Skip rule cells automatically
    sum_numeric_duplicates=True  # Aggregate duplicates
)
```

### Disable Transaction Log for Bulk Loads

```python
# For very large one-time loads
tm1.cells.write(
    cube_name='Sales',
    cellset_as_dict=massive_cellset,
    dimensions=['Product', 'Period', 'Measure'],
    deactivate_transaction_log=True,  # Disable before write
    reactivate_transaction_log=True,  # Re-enable after
    use_blob=True
)
```

---

## Connection Resilience (TM1py 2.1+)

### Auto-Reconnect on Disconnect

TM1py 2.1+ includes built-in resilience:

```python
connection_params = {
    'address': 'localhost',
    'port': 8001,
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    're_connect_on_remote_disconnect': True,  # Auto-reconnect on network issues
    'retry_on_disconnect': True  # Retry failed operations
}

with TM1Service(**connection_params) as tm1:
    # Operations automatically retry on connection loss
    pass
```

### Hybrid Sync/Async Mode

For better responsiveness in mixed workloads:

```python
connection_params = {
    'address': 'localhost',
    'port': 8001,
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    'async_requests_mode': 'hybrid'  # Use hybrid mode for best performance
}

with TM1Service(**connection_params) as tm1:
    df = tm1.cells.execute_mdx_dataframe(query)
```

---

## Metadata Operations

### Bulk Element Creation

```python
# Create many elements at once
from TM1py import Element

elements = [Element(name=f'Product{i}', element_type='Numeric') for i in range(1000)]

# Use add_elements (batch) instead of create (individual)
tm1.elements.add_elements(
    dimension_name='Product',
    hierarchy_name='Product',
    elements=elements
)
```

### Bulk Edge Creation

```python
# Add many edges at once
edges = {
    ('AllProducts', f'Product{i}'): 1
    for i in range(1000)
}

tm1.elements.add_edges(
    dimension_name='Product',
    hierarchy_name='Product',
    edges=edges
)
```

### Efficient Element Queries

```python
# Fast: Get all elements as DataFrame with blob
df = tm1.elements.get_elements_dataframe(
    dimension_name='Product',
    hierarchy_name='Product',
    skip_consolidations=True,
    use_blob=True  # Faster for large dimensions
)

# Slow: Get elements one by one
for elem_name in tm1.elements.get_element_names('Product', 'Product'):
    elem = tm1.elements.get('Product', 'Product', elem_name)  # Many API calls
```

---

## MDX Query Optimization

### Best Practices

1. **Filter early** - Use WHERE clause for dimension filtering
2. **Limit results** - Use TOP clause when testing
3. **Avoid calculated members** - Pre-calculate in TM1 rules
4. **Use TM1SubsetToSet** - Reference existing subsets

```python
# Optimized MDX
mdx = """
SELECT
  NON EMPTY [Product].[Product].Members ON ROWS,
  [Period].[Period].Members ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual])
"""

# Further optimization
mdx_with_subset = """
SELECT
  TM1SubsetToSet([Product], [TopProducts]) ON ROWS,
  {[Period].[2024-Q1]:[Period].[2024-Q4]} ON COLUMNS
FROM [Sales]
WHERE ([Measure].[Revenue], [Version].[Actual])
"""
```

### Query Cellcount First

```python
# Check query size before execution
mdx = "SELECT ... FROM [LargeCube]"

cell_count = tm1.cells.execute_mdx_cellcount(mdx)
print(f"Query will return {cell_count:,} cells")

if cell_count > 1_000_000:
    print("Large query - using blob")
    df = tm1.cells.execute_mdx_dataframe(mdx, use_blob=True)
else:
    df = tm1.cells.execute_mdx_dataframe(mdx)
```

---

## Connection Optimization

### Connection Pool Sizing

```python
# Increase pool for concurrent operations
connection_params = {
    'address': 'localhost',
    'port': 8001,
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    'connection_pool_size': 20,  # Default is 10
    'verify': False
}

with TM1Service(**connection_params) as tm1:
    # Can now handle 20 concurrent operations
    pass
```

### Reuse Connections

```python
# Good: Reuse connection
with TM1Service(**params) as tm1:
    for cube in cubes:
        df = tm1.cells.execute_view_dataframe(cube, 'DataView', private=False)
        # Process df

# Bad: Create connection per operation
for cube in cubes:
    with TM1Service(**params) as tm1:  # Connection overhead each time
        df = tm1.cells.execute_view_dataframe(cube, 'DataView', private=False)
```

---

## Process Execution

### Timeout Management

```python
# Set appropriate timeouts for long processes
success, status, error = tm1.processes.execute_with_return(
    process_name='LongRunningETL',
    timeout=1800.0,  # 30 minutes
    cancel_at_timeout=False,  # Don't cancel if timeout reached
    pYear='2024'
)
```

### Async Process Execution

For IBM Cloud or long processes:

```python
# Execute asynchronously
async_id = tm1.processes.execute_with_return(
    process_name='LongProcess',
    return_async_id=True  # Returns immediately
)

# Poll for completion
import time
while True:
    try:
        success, status, error = tm1.processes.poll_execute_with_return(async_id)
        print(f"Process completed: {status}")
        break
    except:
        time.sleep(5)  # Check every 5 seconds
```

---

## Memory Optimization

### Iterative Processing

```python
# Process large datasets in chunks
chunk_size = 100_000

all_data = []
skip = 0

while True:
    df = tm1.cells.execute_mdx_dataframe(
        mdx=query,
        skip=skip,
        top=chunk_size,
        use_iterative_json=True
    )
    
    if len(df) == 0:
        break
    
    # Process chunk
    processed = process_chunk(df)
    all_data.append(processed)
    
    skip += chunk_size

final_df = pd.concat(all_data, ignore_index=True)
```

### Clear Variables

```python
import gc

# For very large operations
with TM1Service(**params) as tm1:
    df = tm1.cells.execute_mdx_dataframe(large_query, use_blob=True)
    
    # Process
    result = process(df)
    
    # Clear memory
    del df
    gc.collect()
```

---

## Benchmarking

### Measure Performance

```python
import time

def benchmark_operation(operation_func, *args, **kwargs):
    """Measure operation performance"""
    start = time.time()
    result = operation_func(*args, **kwargs)
    elapsed = time.time() - start
    print(f"Operation took {elapsed:.2f} seconds")
    return result, elapsed

# Usage
with TM1Service(**params) as tm1:
    df, duration = benchmark_operation(
        tm1.cells.execute_view_dataframe,
        cube_name='Sales',
        view_name='LargeView',
        use_blob=True
    )
```

### Compare Methods

```python
methods = [
    ('Standard', {'use_blob': False}),
    ('Blob', {'use_blob': True}),
    ('Iterative JSON', {'use_iterative_json': True}),
    ('Blob + Skip Zeros', {'use_blob': True, 'skip_zeros': True})
]

results = {}
with TM1Service(**params) as tm1:
    for name, kwargs in methods:
        start = time.time()
        df = tm1.cells.execute_view_dataframe(
            cube_name='Sales',
            view_name='TestView',
            **kwargs
        )
        elapsed = time.time() - start
        results[name] = {
            'rows': len(df),
            'time': elapsed,
            'rows_per_sec': len(df) / elapsed
        }

for name, metrics in results.items():
    print(f"{name}: {metrics['rows']:,} rows in {metrics['time']:.2f}s "
          f"({metrics['rows_per_sec']:,.0f} rows/sec)")
```

---

## Performance Checklist

### Reading Data
- [ ] Use `use_blob=True` when possible (admin required)
- [ ] Enable `skip_zeros=True` for sparse data
- [ ] Skip consolidations and rule cells if not needed
- [ ] Use async operations for multiple queries
- [ ] Specify exact elements needed in MDX

### Writing Data
- [ ] Use `use_blob=True` or `use_ti=True` for bulk writes
- [ ] Specify dimensions explicitly
- [ ] Use `write_async()` for very large datasets
- [ ] Consider `deactivate_transaction_log` for one-time loads
- [ ] Aggregate duplicates before writing

### General
- [ ] Reuse TM1Service connections
- [ ] Size connection pool appropriately
- [ ] Test queries with small datasets first
- [ ] Monitor TM1 server performance
- [ ] Use appropriate timeouts

---

## Common Performance Issues

### Issue: Slow MDX Queries

**Solutions:**
- Pre-create subsets and use `TM1SubsetToSet()`
- Use views instead of MDX
- Filter with WHERE clause
- Check cube rules/feeders performance

### Issue: Memory Errors

**Solutions:**
- Use `use_iterative_json=True`
- Process data in chunks
- Clear variables and run `gc.collect()`
- Close unused connections

### Issue: Write Timeouts

**Solutions:**
- Use `use_blob=True` or `use_ti=True`
- Disable transaction log temporarily
- Use `write_async()` for parallel processing
- Increase timeout value

### Issue: Connection Pool Exhaustion

**Solutions:**
- Increase `connection_pool_size`
- Close connections properly
- Reduce concurrent operations
- Use connection pooling wisely

---

For API reference, see `API_REFERENCE.md`.  
For connection patterns, see `CONNECTION_GUIDE.md`.  
For examples, see `EXAMPLES.md`.
