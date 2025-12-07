# Spark Notebook

> The user's personal JupyterLab workspace, pre-configured with Spark and data lake access.

| | |
|---|---|
| **Docker Image** | `ghcr.io/berdatalakehouse/spark_notebook:main` |
| **GitHub Repo** | [spark_notebook](https://github.com/BERDataLakehouse/spark_notebook) |
| **Base Image** | [spark_notebook_base](./spark_notebook_base.md) |

## Overview

The Spark Notebook is the primary user interface for the BERDL platform. It provides a JupyterLab environment pre-configured with Spark, Python, and the necessary libraries to interact with the data lake.

## Key Features

- **JupyterLab**: Full-featured notebook environment.
- **Pre-configured Spark**: Ready-to-use Spark session with credentials auto-injected.
- **AI Integration**: Includes `jupyter-ai` for AI-assisted coding.

## Lifecycle

1. **Spawning**: When a user logs into JupyterHub, a dedicated container is spawned.
2. **Initialization**:
    - `entrypoint.sh` sets up user permissions.
    - `patch_jupyter_ai.py` configures AI assistants.
    - `setup_spark_session.py` (via `notebook_utils`) initializes Spark with credentials from MinIO Manager Service.

## Architecture

```mermaid
graph TD
    JH[JupyterHub] -->|Spawns| NB[Spark Notebook Container]
    NB -->|Uses Base| BASE[Spark Notebook Base]
    
    NB -->|Spark Connect| MCP[Datalake MCP Server]
    NB -->|S3| MIN[MinIO]
    NB -->|Metadata| HM[Hive Metastore]
```

## Internal Components

- **`notebook_utils`**: Python package for Spark session management.
- **`configs/`**: Spark defaults, Jupyter extensions, and hooks.
