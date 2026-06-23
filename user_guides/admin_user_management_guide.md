# JupyterHub Admin: User & Tenant Management Guide

This guide covers administrative operations for managing users, groups (tenants), and their access in BERDL. These operations require **JupyterHub Admin** privileges.

---

## Prerequisites

### Required Role: `CDM_JUPYTERHUB_ADMIN`

To perform user and tenant management operations, you must have the `CDM_JUPYTERHUB_ADMIN` role assigned via the **KBase Auth Server**.

> [!IMPORTANT]
> The `CDM_JUPYTERHUB_ADMIN` role must be granted by a KBase System Administrator. Contact them in the [#sysadmin Slack channel](https://kbase.slack.com/archives/C02MHNMCQ) to request this role.

Once you have the admin role, you can access management operations from within a BERDL JupyterHub notebook.

---

## Available Management Operations

Import the management functions:

```python
from berdl_notebook_utils.governance import (
    # List operations
    list_users,
    list_groups,
    # Group member management
    add_group_member,
    remove_group_member,
    # Tenant creation
    create_tenant_and_assign_users,
)
```

---

## User Management

### List All Users

Retrieve all users and their group memberships:

```python
from berdl_notebook_utils.governance import list_users

users = list_users()

# Print users nicely (username + groups)
print(f"{'Username':<20} {'Groups'}")
print("-" * 60)
for user in users.users:
    groups_str = ", ".join(user.groups) if user.groups else "None"
    print(f"{user.username:<20} {groups_str}")
print(f"\nTotal Jupyterhub users: {users.retrieved_count}")
```

---

## Group (Tenant) Management

### List All Groups

View all existing groups and their members:

```python
from berdl_notebook_utils.governance import list_groups

groups = list_groups()

# Print groups nicely (group_name + members)
print(f"{'Group Name':<20} {'Members'}")
print("-" * 60)
for group in groups['groups']:
    members_str = ", ".join(group['members']) if group['members'] else "None"
    print(f"{group['group_name']:<20} {members_str}")
print(f"\nTotal groups: {groups['total_count']}")
```

### Create a New Tenant

Create a new tenant (group) with optional initial members:

```python
from berdl_notebook_utils.governance import create_tenant_and_assign_users

# Create tenant without members
result = create_tenant_and_assign_users("kbase")

# Create tenant with initial members
result = create_tenant_and_assign_users(
    tenant_name="kbase",
    usernames=["alice", "bob", "charlie"]
)

# Check results
print(result["create_tenant"])  # Tenant creation status
print(result["add_members"])    # List of (username, status) tuples
```

### Add Users to a Group

Add users to an existing group. By default, users are added to the read/write group. Use `read_only=True` to add users to the read-only variant instead.

```python
from berdl_notebook_utils.governance import add_group_member
from governance_client.models import GroupManagementResponse

# Add users to read/write group (default)
results = add_group_member(
    group_name="kbase",
    usernames=["alice", "bob"]
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully added {username} to read/write group")
    else:
        print(f"Error adding {username}: {response}")

# Add users to read-only group
results = add_group_member(
    group_name="kbase",
    usernames=["charlie", "david"],
    read_only=True
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully added {username} to read-only group")
    else:
        print(f"Error adding {username}: {response}")
```

### Remove Users from a Group

Remove users from a group. By default, users are removed from the read/write group. Use `read_only=True` to remove users from the read-only variant instead.

```python
from berdl_notebook_utils.governance import remove_group_member
from governance_client.models import GroupManagementResponse

# Remove users from read/write group (default)
results = remove_group_member(
    group_name="kbase",
    usernames=["alice", "bob"]
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully removed {username} from read/write group")
    else:
        print(f"Error removing {username}: {response}")

# Remove users from read-only group
results = remove_group_member(
    group_name="kbase",
    usernames=["charlie"],
    read_only=True
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully removed {username} from read-only group")
    else:
        print(f"Error removing {username}: {response}")
```

---

## Operations Summary

| Operation | Function | Description |
|-----------|----------|-------------|
| List users | `list_users()` | Get all system users |
| List groups | `list_groups()` | Get all groups with members |
| Create tenant | `create_tenant_and_assign_users(name, users)` | Create new group with optional members |
| Add members (R/W) | `add_group_member(group, usernames)` | Add users to read/write group |
| Add members (RO) | `add_group_member(group, usernames, read_only=True)` | Add users to read-only group |
| Remove members (R/W) | `remove_group_member(group, usernames)` | Remove users from read/write group |
| Remove members (RO) | `remove_group_member(group, usernames, read_only=True)` | Remove users from read-only group |

---

## Requesting Admin Access

To obtain the `CDM_JUPYTERHUB_ADMIN` role:

1. Contact the KBase System Administrators in the [#sysadmin Slack channel](https://kbase.slack.com/archives/C02MHNMCQ)
2. Request the `CDM_JUPYTERHUB_ADMIN` role assignment
3. The admin will assign the role via the **KBase Auth Server**
4. Log out and log back into JupyterHub for changes to take effect

> [!NOTE]
> Admin operations will fail with an authentication error if you don't have the required role.
