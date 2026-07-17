# Object Storage (S3) Guide

BERDL stores data in S3-compatible object storage. Inside a BERDL notebook, your
S3 credentials and endpoint are **configured automatically** and kept current —
the AWS CLI and `boto3` work out of the box, with no keys or endpoint to set by
hand.

## How access is configured

When your notebook server starts, BERDL writes two standard AWS files for you:

| File | Contents |
|------|----------|
| `~/.aws/credentials` | Your S3 access key and secret key (the `[default]` profile). |
| `~/.aws/config` | The S3 endpoint (`endpoint_url`) and region for the `[default]` profile. |

Because these are the standard locations the AWS SDKs and CLI read, any tool that
speaks S3 — the `aws` CLI, `boto3`, `s3fs`/`pandas`, `duckdb` — finds your
credentials and the right endpoint with no configuration.

> **Managed files — do not edit.** BERDL owns `~/.aws/credentials` and
> `~/.aws/config` and overwrites them on startup and on every credential
> rotation. Hand edits will be lost.

## Using the AWS CLI

The `aws` CLI is pre-installed and pre-configured. Just use it:

```bash
# List your buckets
aws s3 ls

# List the contents of a bucket
aws s3 ls s3://cdm-lake/

# Upload a file
aws s3 cp ./local_file.parquet s3://<bucket>/path/to/local_file.parquet

# Download a file
aws s3 cp s3://<bucket>/path/to/file.parquet ./file.parquet

# Sync a directory (recursive)
aws s3 sync ./local_dir/ s3://<bucket>/path/to/dir/
```

No `--endpoint-url` and no access keys are needed — they come from the managed
`~/.aws` files.

## Using Python (boto3)

Create a client with **no arguments** — `boto3` reads your credentials from
`~/.aws/credentials` and the endpoint from `~/.aws/config`:

```python
import boto3

s3 = boto3.client("s3")

# List buckets
for bucket in s3.list_buckets()["Buckets"]:
    print(bucket["Name"])

# Upload / download
s3.upload_file("local_file.parquet", "my-bucket", "path/to/local_file.parquet")
s3.download_file("my-bucket", "path/to/file.parquet", "file.parquet")
```

`pandas` (via `s3fs`) works the same way — it picks up the same credentials, so
`pd.read_parquet("s3://my-bucket/path/to/file.parquet")` just works.

## Credentials are kept current automatically

Your S3 credentials rotate from time to time. You do **not** need to reconfigure
anything when they do:

- `~/.aws/credentials` is rewritten with the new secret, and the AWS CLI and
  `boto3` re-read that file on every call — so they always use current
  credentials, even immediately after a rotation.
- New terminals and new notebook kernels automatically pick up the latest
  credentials.

You can also trigger a rotation yourself from any notebook cell:

```python
refresh_spark_environment()
```

This rotates your S3 (and Polaris) credentials with the platform, refreshes your
Spark session, and updates `~/.aws/credentials` in the same step. After it runs,
`aws s3 ls` and `boto3` continue to work with no further action.

> **Long-running jobs:** a `boto3` client or open file handle created *before* a
> rotation keeps using the old credentials for the life of that object. Create a
> fresh client after rotating if a long job outlives a rotation.

## Finding your credentials and endpoint

Usually you never need the raw values, but if you do:

```bash
# From a terminal
aws configure get aws_access_key_id
aws configure get aws_secret_access_key
aws configure get endpoint_url
```

```python
# From a notebook — the access/secret key are also exposed as env vars
import os
print(os.environ["S3_ACCESS_KEY"])   # access key
# Keep your secret key private; avoid printing it in shared notebooks.
```

## Accessing storage from your laptop

The steps above cover the notebook environment. To reach the object store from
your own machine, you need the KBase SSH tunnel and your S3 keys.

**1. Get your keys** from a notebook: `aws configure get aws_access_key_id` and
`aws configure get aws_secret_access_key`.

**2. Open an SSH tunnel** to KBase (a SOCKS5 proxy on port 1338):

```bash
ssh -f -D 1338 -N <your_kbase_username>@login.kbase.us
```

- `-f` runs it in the background, `-D 1338` opens the SOCKS proxy, `-N` forwards
  ports only. Use a different local port if 1338 is taken.
- Verify with `ps aux | grep "ssh -f -D 1338"`; stop it later with `kill <PID>`.

**3. Configure a local AWS profile** for your environment's S3 endpoint and route
it through the proxy:

| Environment | S3 endpoint |
|-------------|-------------|
| Development | `https://minio.dev.berdl.kbase.us` |
| Staging | `https://minio.stage.berdl.kbase.us` |
| Production | `https://minio.berdl.kbase.us` |

> **⚠️ Endpoints may change.** These are the **legacy MinIO** URLs. BERDL is
> migrating object storage to Ceph, so the endpoint for your environment may be
> different — confirm the current S3 endpoint with a BERDL admin.

```bash
# Use the endpoint for your environment (production shown)
aws configure set aws_access_key_id     <your_access_key>            --profile berdl
aws configure set aws_secret_access_key <your_secret_key>            --profile berdl
aws configure set endpoint_url          https://minio.berdl.kbase.us --profile berdl
aws configure set region                us-east-1                    --profile berdl

# Route S3 traffic through the SOCKS proxy for this command
HTTPS_PROXY=socks5://127.0.0.1:1338 aws s3 ls --profile berdl
```

> **⚠️ Proxy scope:** exporting `HTTPS_PROXY`/`HTTP_PROXY` routes *all* traffic in
> that shell through the tunnel. Set it inline (as above) or use a dedicated
> terminal, and unset it when you're done.

## Additional resources

- [AWS CLI S3 reference](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html)
