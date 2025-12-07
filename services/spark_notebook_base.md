# Spark Notebook Base

> The foundational Docker image containing PySpark, Java dependencies, and Python packages for all notebook-based services.

| | |
|---|---|
| **Docker Image** | `ghcr.io/berdatalakehouse/spark_notebook_base:main` |
| **GitHub Repo** | [spark_notebook_base](https://github.com/BERDataLakehouse/spark_notebook_base) |

## Overview

This image provides the core runtime environment for `spark_notebook` and dynamic Spark cluster components. It bundles:
- **PySpark** (based on `quay.io/jupyter/pyspark-notebook`).
- **Java Dependencies**: Custom JARs built via Gradle (Delta Lake connectors, etc.).
- **Python Dependencies**: Managed via `uv` and `pyproject.toml`.
- **System Utilities**: `mc` (MinIO Client), `redis-tools`, `gettext`.

## Key Features

- **Centralized Dependencies**: All client libraries (MCP, MinIO Manager, Spark Manager) are pre-installed here.
- **Multi-stage Build**: Uses Gradle to download JARs in a builder stage, keeping the final image lean.
- **`uv` Package Management**: Fast, reliable Python dependency resolution.

## Build Process

1.  **Builder Stage**: Uses `gradle:9.1.0-jdk24-ubi-minimal` to download Java dependencies.
2.  **Runtime Stage**: Copies JARs to `/usr/local/spark/jars/` and installs Python packages.

## Usage by Other Services

This image is the `FROM` base for:
- `spark_notebook`
- `kube_spark_manager_image` (Dynamic Spark Cluster pods)
