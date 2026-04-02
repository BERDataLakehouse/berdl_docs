# Datalake MCP Server Client

| | |
|---|---|
| **GitHub Repo** | [datalake-mcp-server-client](https://github.com/BERDataLakehouse/datalake-mcp-server-client) |
| **Python** | 3.13 (auto-generated client) |
| **Package Manager** | uv |

The `datalake-mcp-server-client` is an auto-generated Python client library for interacting with the `datalake-mcp-server` APIs. 

## Overview

Unlike manually crafted SDKs, this library is generated directly from the OpenAPI specification of the `datalake-mcp-server`. This ensures that the client remains strictly in sync with the upstream API definitions and data models with zero manual maintenance overhead.

This client is used by other internal BERDL services (such as `spark_notebook_base` and Jupyter tools) to interact programmatically with the Datalake MCP backend services securely.

## Generation Process

The client is generated using the [`openapi-python-client`](https://github.com/openapi-generators/openapi-python-client) utility. A convenience script (`run.sh`) is provided in the repository to automate generation:

1. The script extracts the OpenAPI JSON spec directly from the local `datalake-mcp-server` codebase using an intermediate Python generator (`generate_spec_from_local.py`).
2. The `openapi-python-client` consumes this specification and outputs static Python type hints and API client methods.
3. The resulting code is placed cleanly into `src/datalake_mcp_server_client`.

### Running Generation Locally
```bash
# In the datalake-mcp-server-client repo
./run.sh
```

*(Note: Requires `uv` and Python 3.13 for the build environment).*

## Usage
Other packages can depend on this client by installing it directly from its git repository or path:

```bash
pip install "git+https://github.com/BERDataLakehouse/datalake-mcp-server-client.git"
```
