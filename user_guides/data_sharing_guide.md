# Data Sharing Guide: BERDL JupyterHub

## Overview

The BERDL (BER Data Lake) JupyterHub platform provides data sharing capabilities that allow users to share their datasets with other users and groups.

All data governance functions are **✨ automatically imported** in every notebook - no manual imports needed! This guide shows you how to use these pre-loaded functions for seamless data sharing and collaboration.

## Key Concepts

### Data Organization

All user data in BERDL is organized under personal namespaces:

- **General Data Storage**: `s3a://cdm-lake/users-general-warehouse/{username}/`
- **SQL Warehouse**: `s3a://cdm-lake/users-sql-warehouse/{username}/`

## Getting Started

### Auto-Imported Functions

All BERDL JupyterHub notebooks automatically import these data governance functions at startup (via `startup.py`), so they're ready to use immediately without any imports:

**Auto-Imported Utility Functions:**

*Core Information:*
- `check_governance_health()` - Check service status
- `get_minio_credentials()` - Get your MinIO credentials (sets environment variables)
- `get_my_sql_warehouse()` - Get your SQL warehouse prefix
- `get_my_workspace()` - Get comprehensive workspace information
- `get_namespace_prefix(tenant=None)` - Get namespace prefixes for user/tenant
- `get_my_groups()` - Get list of groups you belong to
- `get_my_policies()` - Get detailed policy information

*Sharing Functions (DEPRECATED):*
- `get_table_access_info(namespace, table_name)` - Check who has access to a SQL table
- `make_table_private(namespace, table_name)` - **(DEPRECATED)** Remove public access from SQL tables
- `make_table_public(namespace, table_name)` - **(DEPRECATED)** Make SQL tables publicly accessible
- `share_table(namespace, table_name, with_users, with_groups)` - **(DEPRECATED)** Share SQL tables
- `unshare_table(namespace, table_name, from_users, from_groups)` - **(DEPRECATED)** Unshare SQL tables

*Admin Functions (Tenant/Group Management):*
- `create_tenant_and_assign_users(tenant_name, usernames)` - Create tenant and add users (admin only)
- `get_group_sql_warehouse(group_name)` - Get SQL warehouse prefix for a group

**Pre-Initialized Client:**
- `governance` - Pre-initialized `DataGovernanceClient()` instance for advanced operations
**Other Auto-Imported Functions:**
- `get_spark_session()` - Create Spark sessions with Iceberg + Delta Lake support
- `create_namespace_if_not_exists()` - Create namespaces (use `iceberg=True` for Iceberg catalogs)
- Plus many other utility functions for data operations

> **Note:** With the migration to Iceberg, **tenant catalogs** are the recommended way to share data. Create tables in a tenant catalog (e.g., `kbase`) and all members can access them. See the [Iceberg Migration Guide](iceberg_migration_guide.md) for details.

### Quick Start

```python
# Check your workspace information
workspace = get_my_workspace()
print(f"👤 Username: {workspace.username}")
print(f"🏠 Home paths: {workspace.home_paths}")
print(f"👥 Groups: {workspace.groups}")
print(f"📁 Accessible paths: {len(workspace.accessible_paths)}")

# Check service health
health = check_governance_health()
print(f"🔍 Service status: {health.status}")
```

## Core Information Functions

### Service Health and Credentials

```python
# Check governance service status
health = check_governance_health()
print(f"Service status: {health.status}")

# Get your SQL warehouse prefix
sql_warehouse = get_my_sql_warehouse()
print(f"SQL warehouse prefix: {sql_warehouse.sql_warehouse_prefix}")

# The pre-initialized governance client is also available
print(f"Governance client ready: {governance is not None}")
```

### Workspace Information

```python
# Get comprehensive workspace information
workspace = get_my_workspace()
print(f"Username: {workspace.username}")
print(f"Home paths: {workspace.home_paths}")
print(f"Groups: {workspace.groups}")
print(f"Total accessible paths: {len(workspace.accessible_paths)}")

# Get your namespace prefix
namespace_info = get_namespace_prefix()
print(f"User namespace prefix: {namespace_info.user_namespace_prefix}")

# Get namespace prefix for a tenant (if you're a member)
tenant_namespace = get_namespace_prefix(tenant="kbase")
print(f"Tenant namespace prefix: {tenant_namespace.tenant_namespace_prefix}")

# Get list of groups you belong to
my_groups = get_my_groups()
print(f"Your groups: {my_groups.groups}")
print(f"Total groups: {my_groups.group_count}")

# Get detailed policy information
policies = get_my_policies()
print(f"User policies: {policies}")
```

## Tenant and Group Management

### Creating Tenants (Admin Only)

Tenants (groups) enable collaborative workspaces where multiple users can share data. **Only admin users** can create tenants.

```python
# Create a new tenant and add users to it
result = create_tenant_and_assign_users(
    tenant_name="kbase",
    usernames=["alice", "bob", "charlie"]
)

# Check creation result
print(f"Tenant creation: {result['create_tenant']}")
print(f"\nUser assignments:")
for username, response in result['add_members']:
    print(f"  {username}: {response}")
```

### Checking Your Group Memberships

```python
# Get list of all groups you belong to
my_groups = get_my_groups()
print(f"Your groups: {my_groups.groups}")
print(f"Total: {my_groups.group_count}")

# Get SQL warehouse prefix for a specific group
group_warehouse = get_group_sql_warehouse("kbase")
print(f"Group SQL warehouse: {group_warehouse.sql_warehouse_prefix}")
```

### Working with Tenant Namespaces

```python
# Get your user namespace prefix
user_ns = get_namespace_prefix()
print(f"Your databases should start with: {user_ns.user_namespace_prefix}")

# Get namespace prefix for a tenant you belong to
tenant_ns = get_namespace_prefix(tenant="kbase")
print(f"Tenant databases should start with: {tenant_ns.tenant_namespace_prefix}")
```

## Sharing SQL Warehouse Tables (DEPRECATED)

> **⚠️ DEPRECATION WARNING**: Direct path sharing functions (`share_table`, `unshare_table`) are deprecated and will be removed in a future release. Please create a **Tenant Workspace** and manage access dynamically through tenant groups instead.

### Share Tables with Users

```python
# Share a table with specific users
response = share_table(
    namespace="personal_namespace",
    table_name="table_name_to_share",
    with_users=["bob", "alice"]
)

print(f"Shared with users: {response.shared_with_users}")
print(f"Success count: {response.success_count}")
if response.errors:
    print(f"Errors: {response.errors}")
```

### Share Tables with Groups

```python
# Share a table with groups
response = share_table(
    namespace="personal_namespace", 
    table_name="table_name_to_share",
    with_groups=["kbase"]
)

print(f"Shared with groups: {response.shared_with_groups}")
if response.errors:
    print(f"Errors: {response.errors}")
```

### Share with Both Users and Groups

```python
# Share with combination of users and groups
response = share_table(
    namespace="personal_namespace",
    table_name="table_name_to_share",
    with_users=["bob", "alice"],
    with_groups=["kbase"]
)

print(f"Successfully shared with {response.success_count} recipients")
if response.errors:
    print(f"Errors: {response.errors}")
```

## Unsharing SQL Warehouse Tables (DEPRECATED)

> **⚠️ DEPRECATION WARNING**: Direct path sharing functions (`share_table`, `unshare_table`) are deprecated and will be removed in a future release. Please create a **Tenant Workspace** and manage access dynamically through tenant groups instead.

### Remove Access from Users

```python
# Remove access from specific users
response = unshare_table(
    namespace="personal_namespace",
    table_name="table_name_to_share",
    from_users=["bob"]  # Remove bob's access, alice keeps access
)

print(f"Removed access from: {response.unshared_from_users}")
if response.errors:
    print(f"Errors: {response.errors}")
```

### Remove Access from Groups

```python
# Remove group access
response = unshare_table(
    namespace="personal_namespace",
    table_name="table_name_to_share", 
    from_groups=["kbase"]
)

print(f"Removed group access from: {response.unshared_from_groups}")
if response.errors:
    print(f"Errors: {response.errors}")
```

### Unshare from Both Users and Groups

```python
# Unshare from both users and groups
response = unshare_table(
    namespace="personal_namespace",
    table_name="table_name_to_share",
    from_users=["bob", "alice"],
    from_groups=["kbase"]
)

print(f"Completely privatized table: {response.success_count} removals")
if response.errors:
    print(f"Errors: {response.errors}")
```

## Public and Private Table Access (DEPRECATED)

> **⚠️ DEPRECATION WARNING**: Direct public path sharing functions (`make_table_public`, `make_table_private`) are deprecated. Please create a namespace under the `kbase` tenant for public sharing activities instead.

### Make Tables Publicly Accessible

```python
# Make a table publicly accessible to all users
response = make_table_public(
    namespace="public_data",
    table_name="climate_dataset"
)

print(f"Table is now public: {response.is_public}")
print(f"Public path: {response.path}")
```

### Make Tables Private

```python
# Remove public access and make table private again
response = make_table_private(
    namespace="public_data", 
    table_name="climate_dataset"
)

print(f"Table is now private: {not response.is_public}")
```

## Managing Access Information

### Check Table Access

For SQL warehouse tables, use the convenient `get_table_access_info` function:

```python
# Check access for a specific table (much easier than constructing paths!)
access_info = get_table_access_info(
    namespace="personal_namespace",
    table_name="table_name_to_share"
)

print(f"Users with access: {access_info.users}")
print(f"Groups with access: {access_info.groups}")
print(f"Public access: {access_info.public}")
print(f"Full path: {access_info.path}")
```

### View Your Complete Workspace

```python
# Get comprehensive workspace information
workspace = get_my_workspace()

print(f"🏠 Your username: {workspace.username}")
print(f"📁 Home directories: {len(workspace.home_paths)}")
for path in workspace.home_paths:
    print(f"   - {path}")

print(f"👥 Group memberships: {len(workspace.groups)}")
for group in workspace.groups:
    print(f"   - {group}")

print(f"🔓 Total accessible paths: {len(workspace.accessible_paths)}")

# Show shared paths (paths not owned by you)
shared_paths = [
    path for path in workspace.accessible_paths 
    if not any(path.startswith(home) for home in workspace.home_paths)
]
print(f"🤝 Shared with you: {len(shared_paths)}")
for path in shared_paths[:5]:  # Show first 5
    print(f"   - {path}")
```

## Working with Shared Tables

### Accessing Shared Tables in Spark

When others share tables with you, you can access them directly in your Spark sessions:

```python
# get_spark_session is auto-imported, no need to import
# Create Spark session
spark = get_spark_session()

# Option 1: Access shared tables via SQL (if shared SQL warehouse)
try:
    shared_df = spark.sql("SELECT * FROM colleague_research.climate_data")
    print(f"📊 Successfully loaded shared table with {shared_df.count()} rows")
    shared_df.show(5)
except Exception as e:
    print(f"❌ Could not access shared table via SQL: {e}")

# Option 2: Access via direct path (if you know the path)
shared_table_path = "s3a://cdm-lake/users-sql-warehouse/colleague/research.db/climate_data"
try:
    shared_df = spark.read.format("delta").load(shared_table_path)
    print(f"📊 Successfully loaded shared table with {shared_df.count()} rows")
except Exception as e:
    print(f"❌ Could not access shared table: {e}")
    # Check access
    access_info = get_table_access_info("research", "climate_data")
    print(f"Your access granted: {get_my_workspace().username in access_info.users}")
```

## Troubleshooting

### Common Issues

1. **Table Not Found**: Ensure the table exists and you have access
   ```python
   # Check if table exists in your accessible workspace
   workspace = get_my_workspace()
   sql_warehouse = get_my_sql_warehouse()
   print(f"Your SQL warehouse: {sql_warehouse.sql_warehouse_prefix}")
   print(f"Accessible paths: {workspace.accessible_paths}")
   
   # List all tables
   spark.sql("SHOW DATABASES").show()
   ```

2. **User Not Found**: Verify usernames are correct and users exist in the system

3. **Namespace Not Found**: Confirm database/namespace names are correct
    ```python
    # Check if namespace exists
    get_databases()
    ```

4. **Permission Denied**: You can only share tables that you own. **Make sure the table is stored in your SQL warehouse location.**

5. **Sharing not taking effect**: MinIO policies may take a few seconds to propagate. Wait 5-10 seconds and retry accessing the shared resource.


### Getting Help
❓ If you have any questions, feel free to reach out to the BERDL Platform Team.
