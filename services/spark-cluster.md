# Spark Cluster (Static/Shared)

> A shared, static Spark cluster used as a fallback for direct MCP access and system tasks.

| | |
|---|---|
| **Docker Image** | `ghcr.io/berdatalakehouse/kube_spark_manager_image:main` |

## Overview

A standalone **static** Spark cluster used for distributed processing.

> **Role**: This cluster primarily serves as a **fallback** or shared resource. It is used by the [Datalake MCP Server](./datalake-mcp-service.md) when no user-specific dynamic cluster is available or for system-level background tasks. Users typically create their own dynamic clusters via the [Spark Cluster Manager](./spark-cluster-manager.md).

## Key Features

- **Always Available**: Runs continuously, unlike dynamic user clusters.
- **MCP Fallback**: Used by the Datalake MCP Server when accessed directly (without JupyterHub context).
- **System Tasks**: Suitable for background jobs and administrative operations.

## Relationship to Notebooks

Notebooks can connect to this cluster to run jobs across multiple nodes (workers), enabling processing of larger datasets than what fits in a single container.
