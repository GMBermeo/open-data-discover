# Toronto Open Data DuckDB Instance

A high-performance [DuckDB](https://duckdb.org/) database instance that integrates all downloaded City of Toronto Open Data packages directly into SQL schemas and views.

## Features

- **Zero-Copy Views**: Instantly creates queryable views over CSV, JSON, GeoJSON, Parquet, Excel (XLSX/XLS), Shapefiles (SHP), and GeoPackages (GPKG) without duplicating data on disk.
- **Catalogue & Metadata Search**: Built-in `metadata.packages` and `metadata.resources` tables indexing all 555 datasets from Open Data Toronto.
- **Spatial Support**: Automatically installs and loads DuckDB's `spatial` extension for GeoJSON, Shapefiles, and GPKG geometry queries (`ST_Read`, `ST_Point`, etc.).
- **Dual Naming**: Datasets are exposed both under dedicated schemas (`bicycle_parking_racks.data_4326`) and top-level alias views (`main.bicycle_parking_racks__data_4326`).
- **Incremental Sync**: Run `build.py` at any point to register newly downloaded datasets into the database.

## Usage

### 1. Build / Synchronize the Database

Syncs all completed datasets from `download/data/` into `db/toronto_opendata.duckdb`:

```bash
uv run -m db.build
```

Options:
- `--data PATH`: custom data folder (default: `download/data`)
- `--db PATH`: custom DuckDB file (default: `db/toronto_opendata.duckdb`)
- `--materialize`: create physical persistent tables instead of views (consumes disk space)

### 2. Querying via CLI

Run ad-hoc SQL queries directly from PowerShell / terminal:

```bash
# Query the catalogue
uv run -m db.cli query "SELECT name, num_resources, is_complete FROM metadata.packages WHERE is_complete = true"

# Query a dataset
uv run -m db.cli query "SELECT * FROM bicycle_parking_racks.data_4326 LIMIT 5"

# Search packages in the catalogue
uv run -m db.cli search "transit"

# List all active registered dataset views
uv run -m db.cli tables

# Interactive SQL shell
uv run -m db.cli shell
```

### 3. Querying from Python

By default, `TorontoDuckDB` connections are **read-only**, allowing multiple concurrent readers (Python scripts, DuckDB UI, CLI) without file locking conflicts:

```python
from db.instance import TorontoDuckDB

# Connects in read-only mode by default
manager = TorontoDuckDB("db/toronto_opendata.duckdb")
con = manager.connect()

# Query with DuckDB
df = con.execute("SELECT * FROM bicycle_parking_racks.data_4326 LIMIT 10").df()
print(df)
```

### 4. External SQL Clients & DuckDB UI

Because DuckDB enforces strict single-writer file locking on Windows, external tools should always attach or open the file in **read-only mode** to allow concurrent querying:

- **DuckDB UI (Web)**:
  Launch the UI with the read-only flag:
  ```bash
  duckdb -readonly db/toronto_opendata.duckdb -ui
  ```
  Or attach inside an existing DuckDB UI notebook:
  ```sql
  ATTACH 'db/toronto_opendata.duckdb' AS toronto (READ_ONLY);
  USE toronto;
  ```
- **DuckDB CLI**: `duckdb -readonly db/toronto_opendata.duckdb`
- **Harlequin**: `harlequin --read-only db/toronto_opendata.duckdb`
- **DBeaver**: Configure the connection with `duckdb.read_only=true`.
