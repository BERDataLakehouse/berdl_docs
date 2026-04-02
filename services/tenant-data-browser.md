# Tenant Data Browser

| | |
|---|---|
| **GitHub Repo** | [tenant-data-browser](https://github.com/BERDataLakehouse/tenant-data-browser) |
| **Python** | >=3.10 (JupyterLab extension) |

The `tenant_data_browser` is a custom JupyterLab extension developed specifically for the BERDL JupyterHub environment.

## Overview

BERDL stores user and tenant data in MinIO object storage. Standard JupyterLab file browsers are designed for local POSIX filesystems and cannot directly interface with S3-compatible endpoints like MinIO effectively without FUSE mounts (which can face performance limitations in data lake scenarios). 

The Tenant Data Browser provides a native UI sidebar within JupyterLab that interfaces directly with the BERDL Datalake MCP backend API to list, preview, and interact with objects stored in MinIO buckets assigned to the user or their tenant groups.

## Features

- **Object Storage Integration:** Native browsing of S3/MinIO bucket hierarchies.
- **Tenant Context:** Automatically displays buckets and paths available to the user based on their KBase organization/tenant memberships.
- **JupyterLab UX:** Integrates into the JupyterLab left sidebar for a seamless "file browser"-like experience for cloud-native data.

## Requirements

- JupyterLab >= 4.0.0

## Development

The extension is built using TypeScript (React) for the frontend and Python for the Jupyter Server extension backend. It uses `uv` for Python dependency management and Jupyter's `jlpm` (Yarn) for JS/TS.

```bash
# Setup
uv sync
uv run jupyter labextension develop . --overwrite
uv run jlpm build

# Run for development
uv run jlpm watch
uv run jupyter lab
```
