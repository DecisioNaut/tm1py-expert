# TM1py Connection Guide

Complete guide to connecting TM1py to different TM1/Planning Analytics environments.

## Contents
- [Connection Overview](#connection-overview)
  - [Basic Pattern](#basic-pattern)
- [TM1 11 On-Premise](#tm1-11-on-premise)
  - [Standard Connection](#standard-connection)
  - [LDAP/CAM Authentication](#ldapcam-authentication)
  - [CAM with SSO](#cam-with-sso)
  - [Connection Pool Configuration](#connection-pool-configuration)
- [TM1 11 IBM Cloud](#tm1-11-ibm-cloud)
  - [Standard IBM Cloud Connection](#standard-ibm-cloud-connection)
  - [Important IBM Cloud Settings](#important-ibm-cloud-settings)
  - [Example with All Options](#example-with-all-options)
- [TM1 12 PAaaS](#tm1-12-paaS-planning-analytics-as-a-service)
  - [API Key Authentication](#api-key-authentication)
  - [URL Structure](#url-structure)
  - [Obtaining API Keys](#obtaining-api-keys)
- [TM1 12 On-Premise](#tm1-12-on-premise)
  - [Standard TM1 12 Connection](#standard-tm1-12-connection)
  - [Access Token Authentication](#access-token-authentication)
- [TM1 12 Cloud Pak for Data](#tm1-12-cloud-pak-for-data)
  - [OAuth2 Client Credentials Flow](#oauth2-client-credentials-flow)
  - [Obtaining OAuth Credentials](#obtaining-oauth-credentials)
- [TM1 12 Advanced Features](#tm1-12-advanced-features-tm1py-21)
  - [Hybrid Sync/Async Mode](#hybrid-syncasync-mode)
  - [Auto-Reconnect on Network Issues](#auto-reconnect-on-network-issues)
- [Connection Best Practices](#connection-best-practices)
- [Connection Parameters Reference](#connection-parameters-reference)
- [Troubleshooting Connection Issues](#troubleshooting-connection-issues)
- [Connection Persistence](#connection-persistence)
- [Examples by Use Case](#examples-by-use-case)

## Connection Overview

TM1py connects to TM1 servers via the TM1 REST API. All connections are established through the `TM1Service` class, which manages authentication, session handling, and HTTP communication.

### Basic Pattern

```python
from TM1py import TM1Service

# Always use context manager (recommended)
with TM1Service(**connection_params) as tm1:
    # Your operations here
    pass

# Manual connection (must close explicitly)
tm1 = TM1Service(**connection_params)
try:
    # Your operations
    pass
finally:
    tm1.logout()
```

---

## TM1 11 On-Premise

### Standard Connection

```python
connection_params = {
    'address': 'localhost',  # or server IP/hostname
    'port': 8001,  # HTTP port from tm1s.cfg (HTTPPortNumber)
    'user': 'admin',
    'password': 'apple',
    'ssl': True,  # Use HTTPS
    'verify': False,  # Skip SSL certificate verification
    'namespace': None  # TM1 (native) authentication
}

with TM1Service(**connection_params) as tm1:
    print(tm1.server.get_product_version())
```

### LDAP/CAM Authentication

```python
connection_params = {
    'address': 'tm1server.company.com',
    'port': 8001,
    'user': 'john.doe',
    'password': 'password123',
    'ssl': True,
    'verify': True,  # Verify SSL if corporate cert installed
    'namespace': 'LDAP'  # or 'AD' or 'CAMNamespace'
}

with TM1Service(**connection_params) as tm1:
    print(f"Connected as: {tm1.whoami.name}")
```

### CAM with SSO

```python
# Windows Integrated Authentication
connection_params = {
    'address': 'tm1server.company.com',
    'port': 8001,
    'user': '',  # Empty for SSO
    'password': '',  # Empty for SSO
    'ssl': True,
    'namespace': 'CAMNamespace',
    'integrated_login': True  # Enable Windows SSO
}

with TM1Service(**connection_params) as tm1:
    print("Connected via SSO")
```

### Connection Pool Configuration

```python
# Adjust connection pool for concurrent operations
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
    # Parallel operations can now use up to 20 connections
    pass
```

---

## TM1 11 IBM Cloud

### Standard IBM Cloud Connection

```python
connection_params = {
    'base_url': 'https://mycompany.planning-analytics.ibmcloud.com/tm1/api/tm1/',
    'user': 'non_interactive_user',  # Service account recommended
    'password': 'your_password',
    'namespace': 'LDAP',
    'ssl': True,
    'verify': True,
    'async_requests_mode': True  # REQUIRED for IBM Cloud to avoid 60s timeout
}

with TM1Service(**connection_params) as tm1:
    print("Connected to IBM Cloud")
```

### Important IBM Cloud Settings

- **ALWAYS** use `async_requests_mode=True` - IBM Cloud has a 60-second API gateway timeout
- Use `base_url` instead of `address` and `port`
- Verify SSL certificates (`verify=True`)
- Use dedicated service accounts, not personal accounts

### Example with All Options

```python
import logging

connection_params = {
    'base_url': 'https://mycompany.planning-analytics.ibmcloud.com/tm1/api/tm1/',
    'user': 'api_user',
    'password': 'secure_password',
    'namespace': 'LDAP',
    'ssl': True,
    'verify': True,
    'async_requests_mode': True,
    'timeout': 60.0,  # Request timeout in seconds
    'connection_pool_size': 10,
    'logging': True  # Enable HTTP logging
}

with TM1Service(**connection_params) as tm1:
    version = tm1.server.get_product_version()
    print(f"TM1 Version: {version}")
```

---

## TM1 12 PAaaS (Planning Analytics as a Service)

### API Key Authentication

```python
connection_params = {
    'base_url': 'https://us-east-1.planninganalytics.saas.ibm.com/api/<TenantId>/v0/tm1/<DatabaseName>/',
    'user': 'apikey',  # Literal string "apikey"
    'password': '<YourActualApiKey>',  # Get from PAaaS admin
    'ssl': True,
    'verify': True,
    'async_requests_mode': True
}

with TM1Service(**connection_params) as tm1:
    print(f"Connected to PAaaS: {tm1.server.get_server_name()}")
```

### URL Structure

PAaaS URLs follow this pattern:
```
https://<region>.planninganalytics.saas.ibm.com/api/<TenantId>/v0/tm1/<DatabaseName>/
```

**Components:**
- `<region>`: us-east-1, eu-central-1, etc.
- `<TenantId>`: Your tenant identifier (GUID)
- `<DatabaseName>`: Your database name

**Example:**
```
https://us-east-1.planninganalytics.saas.ibm.com/api/abc123-def456/v0/tm1/Sales/
```

### Obtaining API Keys

1. Log in to Planning Analytics as a Service
2. Navigate to Administration > Security > Users
3. Select your service account
4. Generate API key
5. Copy the key (it won't be shown again)

---

## TM1 12 On-Premise

### Standard TM1 12 Connection

```python
# Using base_url (new TM1 12 format)
connection_params = {
    'base_url': 'https://pa12.company.com/api/<InstanceId>/v0/tm1/<DatabaseName>/',
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    'verify': True
}

with TM1Service(**connection_params) as tm1:
    print("Connected to TM1 12")
```

### Access Token Authentication

For TM1 12, you can use access tokens instead of passwords:

```python
connection_params = {
    'base_url': 'https://pa12.company.com/api/<InstanceId>/v0/tm1/<DatabaseName>/',
    'user': '<user_id_or_guid>',
    'access_token': '<YourAccessToken>',  # Instead of password
    'ssl': True,
    'verify': True,
    'async_requests_mode': True
}

with TM1Service(**connection_params) as tm1:
    print("Connected with access token")
```

---

## TM1 12 Cloud Pak for Data

### OAuth2 Client Credentials Flow

```python
connection_params = {
    'address': 'tm1-ibm-operands-services.apps.cluster.company.com',
    'instance': 'your_instance_name',  # From Cloud Pak
    'database': 'your_database_name',
    'application_client_id': '<client_id>',  # OAuth2 client ID
    'application_client_secret': '<client_secret>',  # OAuth2 client secret
    'user': 'admin',
    'ssl': True,
    'verify': True
}

with TM1Service(**connection_params) as tm1:
    print("Connected to Cloud Pak for Data")
```

### Obtaining OAuth Credentials

1. Access Cloud Pak for Data console
2. Navigate to Administration > Access control > Service IDs
3. Create service ID or use existing
4. Generate client ID and secret
5. Note the instance and database names from your TM1 service

---

## TM1 12 Advanced Features (TM1py 2.1+)

### Hybrid Sync/Async Mode

For scenarios mixing fast and slow operations:

```python
connection_params = {
    'base_url': 'https://pa12.company.com/api/<InstanceId>/v0/tm1/<DatabaseName>/',
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    'verify': True,
    'async_requests_mode': 'hybrid'  # Best of both worlds
}

with TM1Service(**connection_params) as tm1:
    # Quick reads use sync, long operations use async
    df = tm1.cells.execute_mdx_dataframe(query)
```

### Auto-Reconnect on Network Issues

```python
connection_params = {
    'base_url': 'https://pa12.company.com/api/<InstanceId>/v0/tm1/<DatabaseName>/',
    'user': 'admin',
    'password': 'apple',
    'ssl': True,
    'verify': True,
    're_connect_on_remote_disconnect': True,  # Auto-reconnect
    'retry_on_disconnect': True  # Retry operations
}

with TM1Service(**connection_params) as tm1:
    # Automatic resilience to network breaks
    pass
```

---

## Connection Best Practices

### 1. Use Environment Variables

Never hardcode credentials:

```python
import os
from TM1py import TM1Service

connection_params = {
    'address': os.getenv('TM1_ADDRESS', 'localhost'),
    'port': int(os.getenv('TM1_PORT', '8001')),
    'user': os.getenv('TM1_USER'),
    'password': os.getenv('TM1_PASSWORD'),
    'ssl': os.getenv('TM1_SSL', 'True').lower() == 'true',
    'verify': False
}

with TM1Service(**connection_params) as tm1:
    # Your code here
    pass
```

### 2. Use Configuration Files

Store non-sensitive configs in files:

```python
import configparser
from TM1py import TM1Service

config = configparser.ConfigParser()
config.read('config.ini')

connection_params = {
    'address': config.get('tm1', 'address'),
    'port': config.getint('tm1', 'port'),
    'user': config.get('tm1', 'user'),
    'password': config.get('tm1', 'password'),  # Still sensitive!
    'ssl': config.getboolean('tm1', 'ssl')
}

with TM1Service(**connection_params) as tm1:
    pass
```

**config.ini:**
```ini
[tm1]
address = localhost
port = 8001
user = admin
ssl = True
```

### 3. Use Password Encryption

Encrypt sensitive credentials:

```python
import base64
from TM1py import TM1Service

# Encode password (simple base64 - use proper encryption in production)
def encode_password(password):
    return base64.b64encode(password.encode()).decode()

def decode_password(encoded):
    return base64.b64decode(encoded.encode()).decode()

# Store encoded password in config
encoded_pwd = encode_password('apple')

# Use in connection
connection_params = {
    'address': 'localhost',
    'port': 8001,
    'user': 'admin',
    'password': decode_password(encoded_pwd),
    'ssl': True
}

with TM1Service(**connection_params) as tm1:
    pass
```

### 4. Connection Testing Function

```python
def test_connection(connection_params):
    """Test TM1 connection and return status"""
    try:
        with TM1Service(**connection_params) as tm1:
            version = tm1.server.get_product_version()
            server_name = tm1.server.get_server_name()
            user = tm1.whoami.name
            
            print(f"✓ Connected successfully")
            print(f"  Server: {server_name}")
            print(f"  Version: {version}")
            print(f"  User: {user}")
            return True
            
    except Exception as e:
        print(f"✗ Connection failed: {str(e)}")
        return False

# Usage
test_connection(connection_params)
```

### 5. Multiple Environment Management

```python
class TM1Connections:
    """Manage multiple TM1 connections"""
    
    ENVIRONMENTS = {
        'dev': {
            'address': 'tm1-dev.company.com',
            'port': 8001,
            'user': 'admin',
            'password': os.getenv('TM1_DEV_PWD'),
            'ssl': True
        },
        'test': {
            'address': 'tm1-test.company.com',
            'port': 8001,
            'user': 'admin',
            'password': os.getenv('TM1_TEST_PWD'),
            'ssl': True
        },
        'prod': {
            'address': 'tm1-prod.company.com',
            'port': 8001,
            'user': 'service_account',
            'password': os.getenv('TM1_PROD_PWD'),
            'ssl': True
        }
    }
    
    @staticmethod
    def connect(environment):
        """Connect to specified environment"""
        if environment not in TM1Connections.ENVIRONMENTS:
            raise ValueError(f"Unknown environment: {environment}")
        
        params = TM1Connections.ENVIRONMENTS[environment]
        return TM1Service(**params)

# Usage
with TM1Connections.connect('prod') as tm1_prod:
    # Production operations
    pass
```

---

## Connection Parameters Reference

### Core Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `address` | str | Server IP/hostname | For on-prem |
| `port` | int | HTTP(S) port number | For on-prem |
| `base_url` | str | Full REST API URL | For cloud/TM1 12 |
| `user` | str | Username or "apikey" | Yes |
| `password` | str | Password or API key | Yes* |
| `access_token` | str | OAuth access token | Alternative to password |
| `ssl` | bool | Use HTTPS (True/False) | Yes |
| `verify` | bool | Verify SSL certificate | Recommended |
| `namespace` | str | Auth namespace (TM1, LDAP, CAM) | For LDAP/CAM |

### Advanced Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `async_requests_mode` | bool | False | Enable async mode (IBM Cloud required) |
| `timeout` | float | 60.0 | Request timeout in seconds |
| `connection_pool_size` | int | 10 | Max HTTP connections |
| `integrated_login` | bool | False | Windows SSO |
| `logging` | bool | False | Enable HTTP request logging |
| `session_id` | str | None | Resume existing session |
| `cancel_at_timeout` | bool | False | Cancel TM1 operation on timeout |

### Cloud Pak for Data Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `instance` | str | TM1 instance name |
| `database` | str | Database name |
| `application_client_id` | str | OAuth2 client ID |
| `application_client_secret` | str | OAuth2 client secret |

---

## Troubleshooting Connection Issues

### SSL Certificate Errors

**Error:** `SSLError: [SSL: CERTIFICATE_VERIFY_FAILED]`

**Solution:**
```python
connection_params['verify'] = False  # Disable verification
# Or install certificate in Python's cert store
```

**Production Fix:**
1. Export TM1 server SSL certificate
2. Install in Python certificates: `certifi.where()`
3. Set `verify=True`

### Authentication Failures

**Error:** `TM1pyRestException: 401 Unauthorized`

**Checks:**
- Username and password correct
- Correct namespace (TM1, LDAP, CAM)
- User account not locked
- LDAP server accessible (for LDAP auth)

**Test:**
```python
# Verify credentials in browser first
# Navigate to: https://server:port/api/v1/
```

### Timeout Issues

**Error:** `TM1pyTimeout` or operations hang

**For IBM Cloud:**
```python
connection_params['async_requests_mode'] = True  # REQUIRED
```

**For long operations:**
```python
connection_params['timeout'] = 300.0  # 5 minutes
connection_params['cancel_at_timeout'] = False  # Don't cancel in TM1
```

### Network/Firewall Issues

**Error:** `ConnectionError`, `Connection refused`

**Checks:**
- Server is running
- Port is correct (check tm1s.cfg HTTPPortNumber)
- Firewall allows connection
- For cloud: VPN/network access configured

**Test connectivity:**
```bash
# Test HTTP(S) connectivity
curl -k https://server:port/api/v1/

# Test port is open
telnet server port
```

### IBM Cloud Specific

**Problem:** 60-second timeouts on operations

**Solution:**
```python
connection_params['async_requests_mode'] = True
```

**Problem:** Token expiration

**Solution:** TM1py handles token refresh automatically. Ensure continuous operations don't exceed session timeout.

---

## Connection Persistence

### Save and Restore Sessions

```python
# Save connection to file
tm1 = TM1Service(**connection_params)
tm1.save_to_file('tm1_session.pkl')
tm1.logout()

# Restore connection later
tm1 = TM1Service.restore_from_file('tm1_session.pkl')
# Continue working with restored session
tm1.logout()
```

**Note:** Session files contain sensitive data. Secure appropriately.

---

## Examples by Use Case

### Automated Script (Production)

```python
#!/usr/bin/env python
import os
import sys
from TM1py import TM1Service

def main():
    connection_params = {
        'address': os.getenv('TM1_ADDRESS'),
        'port': int(os.getenv('TM1_PORT')),
        'user': os.getenv('TM1_USER'),
        'password': os.getenv('TM1_PASSWORD'),
        'ssl': True,
        'verify': False
    }
    
    try:
        with TM1Service(**connection_params) as tm1:
            # Your automation here
            success, status, error = tm1.processes.execute_with_return('DailyLoad')
            
            if not success:
                print(f"Process failed: {error}")
                sys.exit(1)
                
            print("Process completed successfully")
            
    except Exception as e:
        print(f"Script failed: {str(e)}")
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### Interactive Development

```python
import getpass
from TM1py import TM1Service

# Prompt for credentials
user = input("TM1 Username: ")
password = getpass.getpass("TM1 Password: ")

connection_params = {
    'address': 'localhost',
    'port': 8001,
    'user': user,
    'password': password,
    'ssl': True,
    'verify': False
}

with TM1Service(**connection_params) as tm1:
    # Interactive work
    cubes = tm1.cubes.get_all_names(skip_control_cubes=True)
    print(f"Available cubes: {', '.join(cubes)}")
```

### Multi-Environment Data Sync

```python
def sync_data(source_env, target_env, cube_name, view_name):
    """Sync data between environments"""
    
    # Read from source
    with TM1Service(**source_env) as tm1_source:
        df = tm1_source.cells.execute_view_dataframe(
            cube_name=cube_name,
            view_name=view_name,
            private=False
        )
    
    # Write to target
    with TM1Service(**target_env) as tm1_target:
       tm1_target.cells.write_dataframe(
            cube_name=cube_name,
            data=df,
            use_ti=True
        )
    
    print(f"Synced {len(df)} rows from {source_env['address']} to {target_env['address']}")

# Usage
dev_params = {...}
test_params = {...}

sync_data(dev_params, test_params, 'Sales', 'ExportView')
```

---

For API reference, see `API_REFERENCE.md`.  
For performance optimization, see `PERFORMANCE.md`.  
For examples and use cases, see `EXAMPLES.md`.
