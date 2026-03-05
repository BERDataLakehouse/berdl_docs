# Spark Connect Remote

**Repository:** [spark_connect_remote](https://github.com/BERDataLakehouse/spark_connect_remote)

The `spark_connect_remote` package is the official Python client library for connecting external scripts and tools to a user's personal Spark cluster running on the BERDL JupyterHub environment.

## Overview

BERDL Data Lakehouse allows users to spin up personal Spark clusters attached to their Jupyter notebook sessions. This library abstracts the connection logic needed to communicate with those sessions remotely via the [Spark Connect gRPC Proxy](./spark_connect_proxy.md).

It handles:
- **KBase Authentication:** Validates KBase tokens and injects them securely as metadata.
- **Secure Tunneling:** Abstracts TLS/HTTPS connection details through the public BERDL Ingress API (`spark.berdl.kbase.us:443`).
- **Session Provisioning:** Instantiates native PySpark session objects ready for dataframe operations.

## Installation & Usage

### Installing
```bash
pip install "git+https://github.com/BERDataLakehouse/spark_connect_remote.git"
```

### Connection Flow
To connect, a user must have an active Jupyter Notebook server running on the HUB, which internally provides the backend compute.

```python
from spark_connect_remote import create_spark_session

# Connect via the public proxy using a KBase token
spark = create_spark_session(
    kbase_token="<YOUR_KBASE_TOKEN>",
)

# Seamlessly execute Spark operations remotely
spark.sql("SHOW DATABASES").show()
```

## Internal Architecture

Under the hood, `create_spark_session` constructs a `SparkSession.builder.remote("sc://spark.berdl.kbase.us:443/;x-kbase-token=...")` string and applies correct configurations, ensuring the connection complies with the reverse proxy's expectations.
