# Polaris Catalog Migration Guide

## Why We Migrated

KBERDL previously used **Delta Lake + Hive Metastore** with namespace isolation enforced by naming conventions — every database had to be prefixed with `u_{username}__` (personal) or `{tenant}_` (shared). This worked but had several limitations:

- **Naming conventions are fragile** — isolation depends on every user following prefix rules correctly
- **No catalog-level boundaries** — all users share the same Hive Metastore, so a misconfigured namespace could leak data
- **Single-engine lock-in** — Delta Lake tables are only accessible through Spark with the Delta extension
- **No time travel or schema evolution** — Delta supports these, but Hive Metastore doesn't track them natively

We migrated to **Apache Polaris + Apache Iceberg**, which provides:

- **Catalog-level isolation** — each user gets their own Polaris catalog (registered in Spark under two aliases: `my` and your username, e.g. `alice`), and each tenant gets a shared catalog (e.g., `kbase`). No naming prefixes needed.
- **Multi-engine support** — Iceberg tables can be read by Spark, Trino, DuckDB, PyIceberg, and other engines
- **Native time travel** — query any previous snapshot of your data
- **Schema evolution** — add, rename, or drop columns without rewriting data
- **ACID transactions** — concurrent reads and writes are safe

## What Changed

| Aspect | Before (Delta/Hive) | After (Polaris/Iceberg) |
|--------|---------------------|------------------------|
| **Metadata catalog** | Hive Metastore | Apache Polaris (Iceberg REST catalog) |
| **Table format** | Delta Lake | Apache Iceberg |
| **Isolation model** | Naming prefixes (`u_user__`, `tenant_`) | Separate catalogs per user/tenant |
| **Your personal catalog** | Hive (shared, prefix-isolated) | Dedicated Polaris catalog (Spark aliases: `my` or `{username}`; Trino uses `{username}`) |
| **Tenant catalogs** | Hive (shared, prefix-isolated) | One catalog per tenant (e.g., `kbase`) |
| **Credentials** | S3 keys only | S3 keys + Polaris OAuth2 (auto-provisioned) |
| **Spark session** | `get_spark_session()` | `get_spark_session()` (unchanged) |

> **Rollout status:** Polaris/Iceberg is now the **default** in BERDL. `create_namespace_if_not_exists()` creates Iceberg namespaces by default; pass `iceberg=False` only if you specifically need the legacy Delta/Hive flow. Existing Delta Lake tables remain accessible at their original `u_{username}__*` / `{tenant}_*` namespaces during the dual-read window. New tables should be created in Iceberg.

## Side-by-Side Comparison

> **Personal catalog aliases:** In Spark, your personal Polaris catalog is registered under **two interchangeable aliases**: `my` and your username (e.g. `alice`). Anywhere the examples below write `my.analysis.t`, you can equally write `alice.analysis.t` — they resolve to the same table. The Iceberg-side snippets use `my` for brevity. In Trino, only the `{username}` alias is registered.

### Create a Namespace

The Iceberg flow is now the default. Pass `iceberg=False` only if you need to fall back to the legacy Delta/Hive prefix logic. `tenant_name` is the **group name** you belong to (e.g. `"kbase"`).

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
spark = get_spark_session()

# Personal namespace
ns = create_namespace_if_not_exists(spark, "analysis")
# Returns: "u_alice__analysis"

# Tenant namespace
ns = create_namespace_if_not_exists(
    spark, "research", tenant_name="kbase"
)
# Returns: "kbase_research"
```

</td>
<td>

```python
spark = get_spark_session()

# Personal namespace
ns = create_namespace_if_not_exists(spark, "analysis")
# Returns: "my.analysis"

# Tenant namespace
ns = create_namespace_if_not_exists(
    spark, "research", tenant_name="kbase"
)
# Returns: "kbase.research"
```

</td>
</tr>
</table>

### Write a Table

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
ns = create_namespace_if_not_exists(spark, "analysis")
# ns = "u_alice__analysis"

df = spark.createDataFrame(data, columns)

# Delta format
df.write.format("delta").saveAsTable(f"{ns}.my_table")
```

</td>
<td>

```python
ns = create_namespace_if_not_exists(spark, "analysis")
# ns = "my.analysis"

df = spark.createDataFrame(data, columns)

# Iceberg format (via writeTo API)
df.writeTo(f"{ns}.my_table").createOrReplace()

# Or append to existing table
df.writeTo(f"{ns}.my_table").append()
```

</td>
</tr>
</table>

> **Bulk ingestion?** For config-driven loading from CSV/TSV/JSON/XML/Parquet sources, the auto-imported `ingest(config)` function from `data_lakehouse_ingest` handles namespace creation, schema enforcement, and writes Iceberg Silver tables in one call. See its docstring for the config schema.

### Read a Table

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
# Query with prefixed namespace
df = spark.sql("""
    SELECT * FROM u_alice__analysis.my_table
""")

# Or use the variable
df = spark.sql(
    f"SELECT * FROM {ns}.my_table"
)
```

</td>
<td>

```python
# Query with catalog.namespace.
# "my" and your username are aliases for
# the same Polaris catalog — both work:
df = spark.sql("""
    SELECT * FROM my.analysis.my_table
""")
df = spark.sql("""
    SELECT * FROM alice.analysis.my_table
""")

# Or use the variable
df = spark.sql(
    f"SELECT * FROM {ns}.my_table"
)
```

</td>
</tr>
</table>

### Cross-Catalog Queries

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
# Everything in one Hive catalog
# Must know the naming convention
spark.sql("""
    SELECT u.name, d.dept_name
    FROM u_alice__analysis.users u
    JOIN kbase_shared.depts d
    ON u.dept_id = d.id
""")
```

</td>
<td>

```python
# Explicit catalog boundaries
spark.sql("""
    SELECT u.name, d.dept_name
    FROM my.analysis.users u
    JOIN kbase.shared.depts d
    ON u.dept_id = d.id
""")
```

</td>
</tr>
</table>

### List Namespaces and Tables

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
# Lists all Hive databases
# (filtered by u_{user}__ prefix)
get_databases()

# List tables in a namespace
get_tables("u_alice__analysis")
```

</td>
<td>

```python
# List all accessible databases across your personal
# catalog and tenant catalogs. Names come back as
# "my.analysis", "alice.analysis", "kbase.genomes", etc.
# — each is usable directly in a query.
get_databases()

# List tables in a specific database / namespace
# For your personal catalog, "my" and your username
# are aliases — both forms work:
get_tables("kbase.genomes")
get_tables("my.analysis")
get_tables("alice.analysis")
```

</td>
</tr>
</table>

### Drop a Table

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
spark.sql(
    "DROP TABLE IF EXISTS "
    "u_alice__analysis.my_table"
)
```

</td>
<td>

```python
# PURGE removes both the catalog entry and
# the underlying S3 data files. Drop without
# PURGE if you want to keep the files on S3.
spark.sql(
    "DROP TABLE IF EXISTS "
    "my.analysis.my_table PURGE"
)
```

> **Note:** Plain `DROP TABLE` (no `PURGE`) removes the catalog entry but leaves the underlying S3 data files.

</td>
</tr>
</table>

### Drop a Whole Namespace (and its S3 data)

Hive lets you drop a database and all its tables in one `CASCADE` statement. **Polaris/Iceberg REST catalogs do not support `CASCADE`** — `DROP NAMESPACE` only succeeds once the namespace is empty. So to remove an entire namespace including the S3 data, you must `DROP TABLE ... PURGE` every table first (PURGE deletes the S3 files), then drop the now-empty namespace.

<table>
<tr><th>Before (Delta/Hive)</th><th>After (Iceberg/Polaris)</th></tr>
<tr>
<td>

```python
# CASCADE drops the database and every
# table in it (managed table data deleted)
spark.sql(
    "DROP DATABASE IF EXISTS "
    "u_alice__analysis CASCADE"
)
```

</td>
<td>

```python
# No CASCADE in Polaris/Iceberg. Purge every
# table (removes S3 data), then drop the
# now-empty namespace.
ns = "my.analysis"

# return_json=False gives a Python list to iterate
for table in get_tables(ns, return_json=False):
    spark.sql(f"DROP TABLE IF EXISTS {ns}.{table} PURGE")

spark.sql(f"DROP NAMESPACE IF EXISTS {ns}")
```

</td>
</tr>
</table>

> **Note:** `get_tables(ns)` returns a JSON string by default — pass `return_json=False` (as above) to get a Python list to iterate over. Skipping `PURGE` on the tables leaves orphaned files in S3 even after the namespace is gone.

## Iceberg-Only Features

These features are only available with Iceberg tables.

### Time Travel

Query a previous version of your table:

```python
# View snapshot history
spark.sql("SELECT * FROM my.analysis.my_table.snapshots")

# Read data as it was at a specific snapshot
spark.sql("""
    SELECT * FROM my.analysis.my_table
    VERSION AS OF 1234567890
""")

# Read data as it was at a specific timestamp
spark.sql("""
    SELECT * FROM my.analysis.my_table
    TIMESTAMP AS OF '2026-03-01 12:00:00'
""")
```

### Schema Evolution

Modify table schema without rewriting data:

```python
# Add a column
spark.sql("ALTER TABLE my.analysis.my_table ADD COLUMN email STRING")

# Rename a column
spark.sql("ALTER TABLE my.analysis.my_table RENAME COLUMN name TO full_name")

# Drop a column
spark.sql("ALTER TABLE my.analysis.my_table DROP COLUMN temp_field")
```

### Snapshot Management

```python
# View snapshot history
display_df(spark.sql("SELECT * FROM my.analysis.my_table.snapshots"))

# View file-level details
display_df(spark.sql("SELECT * FROM my.analysis.my_table.files"))

# View table history
display_df(spark.sql("SELECT * FROM my.analysis.my_table.history"))
```

## Complete Example

```python
# 1. Create a Spark session
print("1. Creating Spark session...")
spark = get_spark_session("MyAnalysis")
print("   Spark session ready.")

# 2. Create a personal namespace
print("\n2. Creating personal namespace...")
ns = create_namespace_if_not_exists(spark, "demo")
print(f"   Namespace: {ns}")  # "my.demo"

# 3. Create a table
print(f"\n3. Creating table {ns}.employees...")
data = [(1, "Alice", 25), (2, "Bob", 30), (3, "Charlie", 35)]
df = spark.createDataFrame(data, ["id", "name", "age"])
df.writeTo(f"{ns}.employees").createOrReplace()
print(f"   Table {ns}.employees created with {df.count()} rows.")

# 4. Query the table
print(f"\n4. Querying {ns}.employees:")
result = spark.sql(f"SELECT * FROM {ns}.employees")
display_df(result)

# 5. Append more data
print(f"\n5. Appending data to {ns}.employees...")
new_data = [(4, "Diana", 28)]
new_df = spark.createDataFrame(new_data, ["id", "name", "age"])
new_df.writeTo(f"{ns}.employees").append()
print(f"   Appended {new_df.count()} row(s). Total: {spark.sql(f'SELECT * FROM {ns}.employees').count()} rows.")
display_df(spark.sql(f"SELECT * FROM {ns}.employees"))

# 6. View snapshots and files (two snapshots now: create + append)
print(f"\n6. Viewing snapshots and files for {ns}.employees:")
print("   Snapshots:")
display_df(spark.sql(f"SELECT * FROM {ns}.employees.snapshots"))
print("   Data files:")
display_df(spark.sql(f"SELECT * FROM {ns}.employees.files"))

# 7. Time travel to the original version
print(f"\n7. Time travel to original version (before append)...")
first_snapshot = spark.sql(
    f"SELECT snapshot_id FROM {ns}.employees.snapshots "
    f"ORDER BY committed_at LIMIT 1"
).collect()[0]["snapshot_id"]
print(f"   First snapshot ID: {first_snapshot}")
original = spark.sql(
    f"SELECT * FROM {ns}.employees VERSION AS OF {first_snapshot}"
)
print(f"   Original version ({original.count()} rows, before append):")
display_df(original)

# 8. Tenant namespace (shared with your team)
print("\n8. Creating tenant namespace and shared table...")
tenant_ns = create_namespace_if_not_exists(
    spark, "shared_data", tenant_name="globalusers"
)
print(f"   Tenant namespace: {tenant_ns}")
df.writeTo(f"{tenant_ns}.team_employees").createOrReplace()
print(f"   Table {tenant_ns}.team_employees created.")

# 9. Cross-catalog query
print(f"\n9. Cross-catalog query ({ns} + {tenant_ns}):")
cross_result = spark.sql(f"""
    SELECT * FROM {ns}.employees
    UNION ALL
    SELECT * FROM {tenant_ns}.team_employees
""")
display_df(cross_result)

# 10. Schema evolution
print(f"\n10. Adding 'email' column to {ns}.employees...")
spark.sql(f"ALTER TABLE {ns}.employees ADD COLUMN email STRING")
print(f"   Updated schema:")
spark.sql(f"DESCRIBE {ns}.employees").show()
display_df(spark.sql(f"SELECT * FROM {ns}.employees"))
print("Done!")
```

## FAQ

**Q: Do I need to change my `get_spark_session()` call?**
No. `get_spark_session()` automatically configures both Delta and Iceberg catalogs. Your Polaris catalogs are ready to use.

**Q: Can I still access my old Delta tables?**
Yes. During the dual-read period, your Delta tables remain accessible at their original names (e.g., `u_{username}__analysis.my_table`). Iceberg copies are at `my.analysis.my_table`.

**Q: What happened to my namespace prefixes (`u_{username}__`)?**
They're no longer needed. With Iceberg, your personal catalog is isolated at the catalog level — only you can access it. In Spark you can address it as either `my` or your username (e.g. `alice`); in Trino, only `{username}` is registered (the `my` alias is Spark-only because Trino catalog names are global on the coordinator).

**Q: How do I share data with my team?**
Create a table in a tenant catalog:
```python
ns = create_namespace_if_not_exists(
    spark, "shared_data", tenant_name="kbase"
)
df.writeTo(f"{ns}.my_shared_table").createOrReplace()
```
All members of the `kbase` tenant can read this table.

**Q: Does `DROP TABLE PURGE` delete the S3 files?**
Yes. `DROP TABLE ... PURGE` removes both the catalog entry and the underlying S3 data files. Plain `DROP TABLE` (no `PURGE`) removes only the catalog entry and leaves the files in S3.

**Q: Can I use `df.write.format("iceberg").saveAsTable(...)` instead of `writeTo`?**
Yes, both work. `writeTo` is the recommended Iceberg API since it supports `createOrReplace()` and `append()` natively.
