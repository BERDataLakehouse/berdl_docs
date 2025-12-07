# Hive Metastore

> The central metadata catalog for all Delta Lake tables.

| | |
|---|---|
| **Docker Image** | `ghcr.io/berdatalakehouse/hive_metastore:main` |

## Overview

The Hive Metastore serves as the central catalog for all tables in the Delta Lake. It stores the metadata (schema, partition info, location) that allows Spark to locate data in MinIO.

## Key Features

- **Schema Registry**: Stores table schemas and partition metadata.
- **Delta Lake Integration**: Works with Delta tables stored in MinIO.
- **Shared Service**: Used by all Spark-based services in the platform.

## Architecture

```mermaid
graph TD
    NB[Spark Notebook] -->|Query Metadata| HM[Hive Metastore]
    MCP[Datalake MCP Server] -->|Query Metadata| HM
    SC[Spark Cluster] -->|Query Metadata| HM
    HM -->|Store| PG[PostgreSQL]
```

## Integrations

- **PostgreSQL**: Backend database for metadata storage.
- **MinIO**: `s3a://cdm-lake/users-sql-warehouse` - Warehouse directory.
- **Used By**: `spark_notebook`, `datalake-mcp-server`, Spark clusters.
