# MinIO Manager Service Client

| | |
|---|---|
| **GitHub Repo** | [minio_manager_service_client](https://github.com/BERDataLakehouse/minio_manager_service_client) |
| **Python** | 3.13 (auto-generated client) |
| **Package Manager** | uv |

The `minio_manager_service_client` (also known functionally as the Governance Client) is an auto-generated Python client library for interacting with the `minio_manager_service` APIs. 

## Overview

Unlike manually crafted SDKs, this library is generated directly from the OpenAPI specification of the `minio_manager_service`. This ensures that the client remains strictly in sync with the upstream API definitions and data models with zero manual maintenance overhead.

This client is heavily utilized by `BERDL_JupyterHub` to automatically configure, rotate, and manage secure tenant MinIO credentials upon user login, as well as by general Spark notebook workflows.

## Generation Process

The client is generated using the [`openapi-python-client`](https://github.com/openapi-generators/openapi-python-client) utility. A convenience script (`run.sh`) is provided to automate generation:

1. The script clones the remote `minio_manager_service` codebase and extracts its OpenAPI JSON spec using an intermediate Python generator (`generate_spec_from_local.py`).
2. The `openapi-python-client` consumes this specification and outputs static Python type hints and API client methods.
3. The resulting code is placed cleanly into `src/governance_client`.

### Running Generation Locally
```bash
# In the minio-manager-service-client repo
./run.sh
```

*(Note: Requires `uv` and Python 3.13 for the build environment).*

## Usage
Other packages can depend on this client by installing it directly from its git repository or path:

```bash
pip install "git+https://github.com/BERDataLakehouse/minio_manager_service_client.git"
```
