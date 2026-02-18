# TM1py Examples and Use Cases

Real-world examples and patterns for common TM1py tasks.

## Contents
- [1. Dimension Management from External Data](#1-dimension-management-from-external-data)
- [2. Data Loading Patterns](#2-data-loading-patterns)
- [3. Automated Reporting](#3-automated-reporting)
- [4. Multi-Environment Data Sync](#4-multi-environment-data-sync)
- [5. Error Handling Patterns](#5-error-handling-patterns)
- [6. Automation Workflows](#6-automation-workflows)
- [7. Advanced Dimension Maintenance](#7-advanced-dimension-maintenance)
- [8. Sandbox Operations](#8-sandbox-operations)
- [9. Security Management](#9-security-management)
- [10. Monitoring and Logging](#10-monitoring-and-logging)

---

## 1. Dimension Management from External Data

### Update Dimension from DataFrame

```python
import pandas as pd
from TM1py import TM1Service, Dimension, Hierarchy, Element, ElementAttribute

# Sample external data
df = pd.read_csv('products.csv')
# products.csv: ProductID, ProductName, Category, Price, Status

def sync_dimension_from_dataframe(tm1, dimension_name, df, 
                                   element_col, type_col='Type',
                                   parent_col=None):
    """
    Synchronize TM1 dimension with DataFrame.
    Creates N-level elements and (optionally) C-level parents.
    """
    
    # Get existing dimension or create new
    if tm1.dimensions.exists(dimension_name):
        dimension = tm1.dimensions.get(dimension_name)
        hierarchy = dimension.default_hierarchy
        existing_elements = set(tm1.elements.get_element_names(
            dimension_name, dimension_name
        ))
    else:
        hierarchy = Hierarchy(name=dimension_name, dimension_name=dimension_name)
        existing_elements = set()
    
    # Create elements from DataFrame
    elements_to_add = []
    for idx, row in df.iterrows():
        elem_name = str(row[element_col])
        
        if elem_name not in existing_elements:
            elem_type = row.get(type_col, 'Numeric')
            if elem_type not in ['Numeric', 'String', 'Consolidated']:
                elem_type = 'Numeric'
            
            elements_to_add.append(Element(
                name=elem_name,
                element_type=elem_type
            ))
    
    # Add new elements
    if elements_to_add:
        if not tm1.dimensions.exists(dimension_name):
            dimension = Dimension(name=dimension_name, hierarchies=[hierarchy])
            tm1.dimensions.create(dimension)
        
        tm1.elements.add_elements(
            dimension_name=dimension_name,
            hierarchy_name=dimension_name,
            elements=elements_to_add
        )
        print(f"Added {len(elements_to_add)} new elements")
    
    # Create parent-child relationships
    if parent_col and parent_col in df.columns:
        edges = {}
        for idx, row in df.iterrows():
            child = str(row[element_col])
            parent = str(row[parent_col])
            if pd.notna(parent) and parent != child:
                edges[(parent, child)] = 1
        
        if edges:
            # Ensure parent elements exist as Consolidated
            parent_elements = list(set(p for p, c in edges.keys()))
            for parent in parent_elements:
                if not tm1.elements.exists(dimension_name, dimension_name, parent):
                    tm1.elements.create(
                        dimension_name, dimension_name, 
                        Element(name=parent, element_type='Consolidated')
                    )
            
            tm1.elements.add_edges(dimension_name, dimension_name, edges)
            print(f"Added {len(edges)} edges")
    
    return f"Dimension '{dimension_name}' synchronized successfully"

# Usage
with TM1Service(**config) as tm1:
    result = sync_dimension_from_dataframe(
        tm1=tm1,
        dimension_name='Product',
        df=df,
        element_col='ProductID',
        parent_col='Category'
    )
    print(result)
```

---

## 2. Data Loading Patterns

### Load Data from CSV to Cube

```python
import pandas as pd

def load_data_from_csv(tm1, csv_path, cube_name, dimension_mapping, 
                        measure_col, value_col):
    """
    Load data from CSV into TM1 cube with proper error handling.
    
    Args:
        csv_path: Path to CSV file
        cube_name: Target cube name
        dimension_mapping: Dict mapping cube dimensions to CSV columns
        measure_col: Column name for Measure dimension
        value_col: Column name containing values
    """
    
    # Read and validate data
    df = pd.read_csv(csv_path)
    print(f"Loaded {len(df):,} rows from CSV")
    
    # Validate columns
    required_cols = list(dimension_mapping.values()) + [measure_col, value_col]
    missing = set(required_cols) - set(df.columns)
    if missing:
        raise ValueError(f"Missing columns in CSV: {missing}")
    
    # Remove rows with missing values
    df = df.dropna(subset=required_cols)
    print(f"Clean rows: {len(df):,}")
    
    # Build cellset
    cellset = {}
    dimensions = list(dimension_mapping.keys()) + ['Measure']
    
    for idx, row in df.iterrows():
        # Build coordinate tuple
        coord = tuple(
            str(row[dimension_mapping[dim]]) 
            for dim in dimension_mapping.keys()
        ) + (str(row[measure_col]),)
        
        # Get value
        value = row[value_col]
        if pd.notna(value) and value != 0:
            cellset[coord] = float(value)
    
    print(f"Writing {len(cellset):,} cells to cube '{cube_name}'")
    
    # Write to TM1
    tm1.cells.write(
        cube_name=cube_name,
        cellset_as_dict=cellset,
        dimensions=dimensions,
        use_blob=True,  # Fast write
        increment=False  # Replace values
    )
    
    return len(cellset)

# Usage example
dimension_mapping = {
    'Product': 'ProductID',
    'Period': 'Month',
    'Version': 'VersionName',
    'Region': 'RegionCode'
}

with TM1Service(**config) as tm1:
    cells_written = load_data_from_csv(
        tm1=tm1,
        csv_path='sales_data.csv',
        cube_name='Sales',
        dimension_mapping=dimension_mapping,
        measure_col='MeasureName',
        value_col='Value'
    )
    print(f"Successfully wrote {cells_written:,} cells")
```

### Incremental Data Load

```python
def incremental_cube_load(tm1, source_df, cube_name, dimensions, 
                           clear_existing=False):
    """
    Load data incrementally with optional clearing of existing data.
    """
    
    if clear_existing:
        print(f"Clearing existing data in '{cube_name}'")
        # Build MDX to clear specific slice
        clear_view = tm1.cubes.views.create_from_mdx(
            cube_name=cube_name,
            view_name='__temp_clear_view',
            mdx="SELECT ... FROM ...",  # Define scope
            private=True
        )
        tm1.cells.clear(cube_name=cube_name, **clear_view.MDXViewMembers)
        tm1.cubes.views.delete(cube_name, '__temp_clear_view', private=True)
    
    # Write new data
    tm1.cells.write_dataframe(
        cube_name=cube_name,
        data=source_df,
        use_blob=True,
        skip_non_updateable=True,
        increment=not clear_existing
    )
    
    print(f"Loaded {len(source_df):,} rows")

# Usage
with TM1Service(**config) as tm1:
    df = pd.read_csv('delta_data.csv')
    incremental_cube_load(
        tm1=tm1,
        source_df=df,
        cube_name='Actuals',
        dimensions=['Product', 'Period', 'Version', 'Measure'],
        clear_existing=True  # Clear before loading
    )
```

---

## 3. Automated Reporting

### Extract Data and Generate Excel Report

```python
import pandas as pd
from openpyxl import Workbook
from openpyxl.utils.dataframe import dataframe_to_rows
from openpyxl.styles import Font, PatternFill

def generate_excel_report(tm1, queries, output_path, report_title):
    """
    Generate multi-sheet Excel report from TM1 data.
    
    Args:
        queries: Dict of {sheet_name: mdx_query}
        output_path: Excel file path
        report_title: Report title for cover sheet
    """
    
    wb = Workbook()
    wb.remove(wb.active)  # Remove default sheet
    
    # Create cover sheet
    cover = wb.create_sheet('Report Info')
    cover['A1'] = report_title
    cover['A1'].font = Font(size=16, bold=True)
    cover['A3'] = f"Generated: {pd.Timestamp.now()}"
    cover['A4'] = f"TM1 Server: {tm1.server.get_server_name()}"
    
    # Execute queries and create sheets
    for sheet_name, mdx in queries.items():
        print(f"Executing query for '{sheet_name}'...")
        
        df = tm1.cells.execute_mdx_dataframe(
            mdx=mdx,
            use_blob=True,
            skip_zeros=True
        )
        
        # Create sheet
        ws = wb.create_sheet(sheet_name)
        
        # Write data
        for r_idx, row in enumerate(dataframe_to_rows(df, index=False, header=True), 1):
            for c_idx, value in enumerate(row, 1):
                cell = ws.cell(row=r_idx, column=c_idx, value=value)
                
                # Format header
                if r_idx == 1:
                    cell.font = Font(bold=True)
                    cell.fill = PatternFill(start_color="CCE5FF", 
                                           end_color="CCE5FF", 
                                           fill_type="solid")
        
        # Auto-size columns
        for column in ws.columns:
            max_length = 0
            column_letter = column[0].column_letter
            for cell in column:
                try:
                    if len(str(cell.value)) > max_length:
                        max_length = len(cell.value)
                except:
                    pass
            adjusted_width = min(max_length + 2, 50)
            ws.column_dimensions[column_letter].width = adjusted_width
        
        print(f"  Added sheet '{sheet_name}' with {len(df):,} rows")
    
    # Save workbook
    wb.save(output_path)
    print(f"\nReport saved to: {output_path}")

# Usage
queries = {
    'Revenue by Region': """
        SELECT
          NON EMPTY [Region].[Region].Members ON ROWS,
          [Period].[2024].Children ON COLUMNS
        FROM [Sales]
        WHERE ([Measure].[Revenue], [Version].[Actual])
    """,
    'Top Products': """
        SELECT TOP 20
          ORDER([Product].[Product].Members, [Measure].[Revenue], DESC) ON ROWS,
          {[Measure].[Revenue], [Measure].[Units]} ON COLUMNS
        FROM [Sales]
        WHERE ([Period].[2024-Q1], [Version].[Actual])
    """
}

with TM1Service(**config) as tm1:
    generate_excel_report(
        tm1=tm1,
        queries=queries,
        output_path='monthly_report.xlsx',
        report_title='Sales Report - Q1 2024'
    )
```

---

## 4. Multi-Environment Data Sync

### Sync Data Between TM1 Instances

```python
def sync_cube_data(source_tm1, target_tm1, cube_name, 
                    mdx_filter=None, clear_target=False):
    """
    Synchronize cube data from source to target TM1 instance.
    """
    
    print(f"Syncing cube: {cube_name}")
    
    # Build query
    if mdx_filter:
        mdx = mdx_filter
    else:
        # Get all leaf data
        dimensions = source_tm1.cubes.get_dimension_names(cube_name)
        mdx = f"SELECT NON EMPTY {{TM1SubsetAll([{dimensions[0]}])}} ON ROWS FROM [{cube_name}]"
    
    # Extract from source
    print("  Extracting data from source...")
    df = source_tm1.cells.execute_mdx_dataframe(
        mdx=mdx,
        use_blob=True,
        skip_zeros=True,
        skip_consolidated_cells=True,
        skip_rule_derived_cells=True
    )
    
    print(f"  Extracted {len(df):,} rows")
    
    # Clear target if requested
    if clear_target:
        print("  Clearing target cube...")
        target_tm1.cells.clear_with_mdx(cube_name, mdx)
    
    # Write to target
    print("  Writing to target...")
    target_tm1.cells.write_dataframe(
        cube_name=cube_name,
        data=df,
        use_blob=True,
        skip_non_updateable=True
    )
    
    print(f"  Sync completed: {len(df):,} rows")

# Usage: Sync DEV → PROD
dev_config = {
    'address': 'dev-server',
    'port': 8001,
    'user': 'admin',
    'password': 'dev_password',
    'ssl': True
}

prod_config = {
    'address': 'prod-server',
    'port': 8001,
    'user': 'admin',
    'password': 'prod_password',
    'ssl': True
}

with TM1Service(**dev_config) as dev_tm1, \
     TM1Service(**prod_config) as prod_tm1:
    
    sync_cube_data(
        source_tm1=dev_tm1,
        target_tm1=prod_tm1,
        cube_name='Budget',
        mdx_filter="SELECT ... WHERE ([Version].[Final])",
        clear_target=True
    )
```

---

## 5. Error Handling Patterns

### Robust Data Processing with Retry Logic

```python
import time
from requests.exceptions import RequestException, Timeout

def execute_with_retry(tm1_func, *args, max_retries=3, 
                        backoff=2, **kwargs):
    """
    Execute TM1 operation with exponential backoff retry.
    """
    for attempt in range(max_retries):
        try:
            return tm1_func(*args, **kwargs)
        except (Timeout, RequestException) as e:
            if attempt == max_retries - 1:
                raise  # Final attempt failed
            
            wait_time = backoff ** attempt
            print(f"Attempt {attempt + 1} failed: {e}")
            print(f"Retrying in {wait_time} seconds...")
            time.sleep(wait_time)
    
    raise RuntimeError(f"Failed after {max_retries} attempts")

# Usage
with TM1Service(**config) as tm1:
    df = execute_with_retry(
        tm1.cells.execute_view_dataframe,
        cube_name='LargeCube',
        view_name='DataView',
        use_blob=True,
        max_retries=3
    )
```

### Validate Before Writing

```python
def validate_and_write(tm1, cube_name, cellset, dimensions):
    """
    Validate data before writing to TM1.
    """
    
    # 1. Check cube exists
    if not tm1.cubes.exists(cube_name):
        raise ValueError(f"Cube '{cube_name}' does not exist")
    
    # 2. Validate dimensions
    cube_dims = tm1.cubes.get_dimension_names(cube_name)
    if set(dimensions) != set(cube_dims):
        raise ValueError(f"Dimension mismatch. Cube has: {cube_dims}")
    
    # 3. Check element existence (sample)
    sample_coord = next(iter(cellset.keys()))
    for idx, elem in enumerate(sample_coord):
        dim_name = dimensions[idx]
        if not tm1.elements.exists(dim_name, dim_name, elem):
            raise ValueError(
                f"Element '{elem}' does not exist in dimension '{dim_name}'"
            )
    
    # 4. Validate values
    invalid_values = [
        coord for coord, val in cellset.items()
        if not isinstance(val, (int, float)) or pd.isna(val)
    ]
    if invalid_values:
        raise ValueError(f"Invalid values found: {invalid_values[:5]}...")
    
    # All validations passed - write
    tm1.cells.write(
        cube_name=cube_name,
        cellset_as_dict=cellset,
        dimensions=dimensions,
        use_blob=True
    )
    
    print(f"Successfully wrote {len(cellset):,} cells")

# Usage
try:
    with TM1Service(**config) as tm1:
        validate_and_write(
            tm1=tm1,
            cube_name='Sales',
            cellset=my_cellset,
            dimensions=['Product', 'Period', 'Version', 'Measure']
        )
except ValueError as e:
    print(f"Validation error: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

---

## 6. Automation Workflows

### Daily Data Refresh Workflow

```python
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(f'tm1_refresh_{datetime.now():%Y%m%d}.log'),
        logging.StreamHandler()
    ]
)

def daily_refresh_workflow(config):
    """
    Complete daily refresh workflow with error handling.
    """
    
    start_time = time.time()
    logging.info("="*50)
    logging.info("Starting Daily Refresh Workflow")
    logging.info("="*50)
    
    try:
        with TM1Service(**config) as tm1:
            # 1. Update dimensions
            logging.info("Step 1: Updating dimensions...")
            tm1.processes.execute('Update_Product_Dimension')
            logging.info("  ✓ Product dimension updated")
            
            # 2. Clear staging cubes
            logging.info("Step 2: Clearing staging cubes...")
            tm1.processes.execute('Clear_Staging_Data')
            logging.info("  ✓ Staging cleared")
            
            # 3. Load new data
            logging.info("Step 3: Loading data from source...")
            success, status, error = tm1.processes.execute_with_return(
                process_name='Load_Daily_Actuals',
                timeout=600,
                pDate=datetime.now().strftime('%Y-%m-%d')
            )
            
            if not success:
                raise RuntimeError(f"Data load failed: {error}")
            logging.info(f"  ✓ Data loaded: {status}")
            
            # 4. Run calculations
            logging.info("Step 4: Running calculations...")
            tm1.processes.execute('Calculate_KPIs')
            logging.info("  ✓ Calculations completed")
            
            # 5. Validate results
            logging.info("Step 5: Validating results...")
            validation_mdx = """
                SELECT [Measure].[RecordCount] ON 0 
                FROM [Control]
                WHERE ([Date].[Today])
            """
            df = tm1.cells.execute_mdx_dataframe(validation_mdx)
            record_count = df['Value'].iloc[0]
            
            if record_count == 0:
                raise ValueError("No records loaded!")
            
            logging.info(f"  ✓ Validation passed: {record_count:,} records")
            
            # 6. Generate reports
            logging.info("Step 6: Generating reports...")
            tm1.processes.execute('Generate_Daily_Reports')
            logging.info("  ✓ Reports generated")
            
    except Exception as e:
        logging.error(f"❌ Workflow failed: {e}")
        # Send alert email here
        raise
    
    finally:
        elapsed = time.time() - start_time
        logging.info(f"\nWorkflow completed in {elapsed:.1f} seconds")
        logging.info("="*50)

# Run workflow
if __name__ == '__main__':
    config = {
        'address': 'localhost',
        'port': 8001,
        'user': 'admin',
        'password': 'apple',
        'ssl': True
    }
    
    daily_refresh_workflow(config)
```

---

## 7. Advanced Dimension Maintenance

### Maintain Alternate Hierarchies

```python
def create_alternate_hierarchy(tm1, dimension_name, hierarchy_name, 
                                 element_mapping):
    """
    Create alternate hierarchy with different element structure.
    
    Args:
        element_mapping: Dict of {leaf_element: parent_element}
    """
    
    dimension = tm1.dimensions.get(dimension_name)
    
    # Create new hierarchy
    from TM1py import Hierarchy, Element
    
    alt_hierarchy = Hierarchy(
        name=hierarchy_name,
        dimension_name=dimension_name
    )
    
    # Add all leaf elements
    leaf_elements = set(element_mapping.keys())
    for elem_name in leaf_elements:
        alt_hierarchy.add_element(
            element_name=elem_name,
            element_type='Numeric'
        )
    
    # Add parent (consolidated) elements
    parent_elements = set(element_mapping.values())
    for parent in parent_elements:
        alt_hierarchy.add_element(
            element_name=parent,
            element_type='Consolidated'
        )
    
    # Add edges
    for child, parent in element_mapping.items():
        alt_hierarchy.add_edge(parent=parent, component=child, weight=1)
    
    # Create hierarchy in TM1
    dimension.add_hierarchy(alt_hierarchy)
    tm1.dimensions.update(dimension)
    
    print(f"Created alternate hierarchy '{hierarchy_name}' in '{dimension_name}'")

# Usage: Create regional grouping hierarchy
region_mapping = {
    'CA': 'North America',
    'US': 'North America',
    'MX': 'North America',
    'UK': 'Europe',
    'DE': 'Europe',
    'FR': 'Europe',
    'JP': 'Asia Pacific',
    'AU': 'Asia Pacific'
}

with TM1Service(**config) as tm1:
    create_alternate_hierarchy(
        tm1=tm1,
        dimension_name='Country',
        hierarchy_name='Region',
        element_mapping=region_mapping
    )
```

---

## 8. Sandbox Operations

### Copy Data to Sandbox for Testing

```python
def create_test_sandbox(tm1, base_sandbox_name, user_name):
    """
    Create sandbox for testing with copy of base data.
    """
    
    sandbox_name = f"{base_sandbox_name}_{user_name}"
    
    # Create sandbox if doesn't exist
    if not tm1.sandboxes.exists(sandbox_name):
        tm1.sandboxes.create(sandbox_name)
        print(f"Created sandbox: {sandbox_name}")
    
    # Publish base sandbox to new sandbox (copies data)
    tm1.sandboxes.publish(
        source_sandbox_name=base_sandbox_name,
        target_sandbox_names=[sandbox_name]
    )
    
    print(f"Copied data from '{base_sandbox_name}' to '{sandbox_name}'")
    
    return sandbox_name

# Usage
with TM1Service(**config) as tm1:
    test_sandbox = create_test_sandbox(
        tm1=tm1,
        base_sandbox_name='Budget_Base',
        user_name='jsmith'
    )
    
    # Work in sandbox
    tm1.cells.write_value(
        value=50000,
        cube_name='Budget',
        element_tuple=('Product1', '2024-Q1', 'Plan', 'Revenue'),
        sandbox_name=test_sandbox
    )
    
    # Merge back or discard
    # tm1.sandboxes.publish(test_sandbox)  # To merge
    # tm1.sandboxes.delete(test_sandbox)   # To discard
```

---

## 9. Security Management

### Grant Access to User

```python
def setup_user_access(tm1, user_name, full_name, groups, 
                       cube_permissions):
    """
    Create user and grant access to cubes.
    
    Args:
        groups: List of group names
        cube_permissions: Dict of {cube_name: 'READ'|'WRITE'|'RESERVE'}
    """
    
    # Create user if doesn't exist
    if not tm1.security.users.exists(user_name):
        tm1.security.users.create(user_name, full_name)
        print(f"Created user: {user_name}")
    
    # Add to groups
    for group in groups:
        if tm1.security.groups.exists(group):
            tm1.security.groups.add_user(group, user_name)
            print(f"  Added to group: {group}")
    
    # Grant cube permissions
    for cube_name, permission in cube_permissions.items():
        tm1.security.cubes.update_permission(
            cube_name=cube_name,
            object_name=user_name,
            permission=permission
        )
        print(f"  Granted {permission} on cube: {cube_name}")

# Usage
with TM1Service(**config) as tm1:
    setup_user_access(
        tm1=tm1,
        user_name='analyst1',
        full_name='Data Analyst',
        groups=['Analysts', 'PowerUsers'],
        cube_permissions={
            'Sales': 'READ',
            'Budget': 'WRITE',
            'Actuals': 'READ'
        }
    )
```

---

## 10. Monitoring and Logging

### Monitor Long-Running Processes

```python
import time

def monitor_process_execution(tm1, process_name, check_interval=10):
    """
    Execute process and monitor progress via log messages.
    """
    
    print(f"Starting process: {process_name}")
    start_time = time.time()
    
    # Execute async
    async_id = tm1.processes.execute_with_return(
        process_name=process_name,
        return_async_id=True
    )
    
    print(f"Process started with async ID: {async_id}")
    
    # Poll for completion
    while True:
        try:
            success, status, error = tm1.processes.poll_execute_with_return(
                async_id
            )
            
            elapsed = time.time() - start_time
            print(f"\n✓ Process completed in {elapsed:.1f}s")
            print(f"Status: {status}")
            
            if not success:
                print(f"Error: {error}")
                return False
            
            return True
            
        except:
            # Still running
            elapsed = time.time() - start_time
            print(f"  Running... ({elapsed:.0f}s)", end='\r')
            time.sleep(check_interval)

# Usage
with TM1Service(**config) as tm1:
    success = monitor_process_execution(
        tm1=tm1,
        process_name='Long_Data_Load',
        check_interval=5
    )
    
    if success:
        print("Process succeeded!")
    else:
        print("Process failed - check logs")
```

---

For API reference, see `API_REFERENCE.md`.  
For data operations details, see `DATA_OPERATIONS.md`.  
For performance optimization, see `PERFORMANCE.md`.
