# BERDL Remote

| | |
|---|---|
| **GitHub Repo** | [berdl_remote](https://github.com/BERDataLakehouse/berdl_remote) |
| **Python** | 3.13 |
| **Package Manager** | uv |

`berdl-remote` is a CLI tool designed for remotely executing notebooks and shell commands on a user's BERDL JupyterHub environment directly from their local machine. This allows seamless integration of BERDL's scalable cloud compute resources into local workflows and automated scripts.

## Core Features

- **Authentication:** Authenticats users against KBase Auth2 via short-lived tokens and manages secure JupyterHub session cookies.
- **Server Spawning:** Can start the default user Jupyter server ("Large" profile) via the JupyterHub API and wait until it is ready for execution.
- **Remote Execution (Papermill):** Executes Jupyter notebooks remotely using Papermill, capable of reading and writing to/from S3/MinIO paths directly.
- **Remote Code Evaluation:** Execute Python strings directly within a remote Spark-configured kernel environment.
- **Remote Shell Execution:** Run shell scripts or CLI commands in the remote Jupyter environment.

## Usage Overview

The CLI provides commands for interacting with the user's remote session:

```bash
# Authenticate and obtain session cookies
berdl-remote login

# Start the user's Jupyter Server instances
berdl-remote spawn

# Check the connection status to the server
berdl-remote status

# Execute a notebook with parameters using Papermill
berdl-remote run /home/myuser/notebooks/analysis.ipynb -p batch_size 100 --output /home/myuser/notebooks/analysis_executed.ipynb

# Execute Python or Shell commands in the remote Jupyter server
berdl-remote python "print(get_settings().USER)"
berdl-remote shell "ls -la /minio/my-files/notebooks/"
```

## Security & Configuration

The CLI safely manages sessions securely:
- Credentials (session cookies and config) are stored locally at `~/.berdl/remote-config.yaml` with restricted permissions.
- KBase Auth Tokens can be passed securely via interactive prompts or via the `KBASE_AUTH_TOKEN` environment variable for automation. The token natively resolves to a KBase user before obtaining JupyterHub proxy cookies.
