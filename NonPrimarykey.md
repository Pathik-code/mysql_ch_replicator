# ClickHouse MySQL Dump Migration: Final Changes & Setup

## Final Changes Made

### 1. Primary Key Handling in Converter
- In `converter.py`, logic was added to check if a table has no primary key.
- If no primary key is found but the table has a unique column (e.g., `UserID`), `UserID` is set as the primary key for ClickHouse.
- This ensures ClickHouse always has a primary key for the `ORDER BY` clause, which is required by MergeTree engines.

### 2. ClickHouse Table Creation Query
- In `clickhouse_api.py`, the `CREATE_TABLE_QUERY` was updated to use:
  ```sql
  ENGINE = MergeTree()
  ORDER BY tuple()
  SETTINGS index_granularity = 8192, allow_nullable_key = 1
  ```
- This allows nullable keys and uses a generic `ORDER BY tuple()` if no unique column is available.

### 3. Indentation and Query Fixes
- Instead of relying on config file creation, changes were made directly in the `mysql_ch_replicator` library files.
- This approach allows for easier future modifications in your script.
- Python indentation issues were fixed (replacing tabs with spaces).
- The MySQL query logic was updated to omit the `LIMIT` clause when `limit=None`, preventing SQL errors:
    ```python
    if limit is not None:
        query = f'SELECT * FROM `{table_name}` {where}ORDER BY {order_by_str} LIMIT {limit}'
    else:
        query = f'SELECT * FROM `{table_name}` {where}ORDER BY {order_by_str}'
    ```

### 4. ClickHouse Table Drop Logic
- Switched to using `clickhouse-connect` for dropping tables before recreating them, ensuring a clean migration.

---

## How You Set Up to Dump Non-Primary Key Tables

- **Bulk Data Dump:**
  - Data is fetched from MySQL and inserted into ClickHouse using the mapped structure.
  - If there are duplicate `UserID` values and you use `ReplacingMergeTree`, only the latest row per `UserID` will be kept.

---

## Example Table Creation

```sql
CREATE TABLE ramfin.loanbook20250531
(
    UserID Int32,
    ... -- other columns
    _version UInt64
)
ENGINE = MergeTree()
ORDER BY tuple()
SETTINGS index_granularity = 8192, allow_nullable_key = 1;
```

---

## Example Migration Script for Bulk Dumping Non-Primary Key Tables

```python
import sys
import yaml
import clickhouse_connect
from mysql_ch_replicator.mysql_api import MySQLApi
from mysql_ch_replicator.clickhouse_api import ClickhouseApi
from mysql_ch_replicator.config import ClickhouseSettings, MysqlSettings
from mysql_ch_replicator.converter import MysqlToClickhouseConverter

# Load config
with open('config.yaml', 'r') as f:
    config = yaml.safe_load(f)

mysql_conf = config['mysql']
clickhouse_conf = config['clickhouse']

# Read table list from config file
tables = config.get('tables', [])

# MySQL connection
mysql_settings = MysqlSettings(
    host=mysql_conf['host'],
    port=mysql_conf['port'],
    user=mysql_conf['user'],
    password=mysql_conf['password']
)
mysql_api = MySQLApi(
    database=mysql_conf['database'],
    mysql_settings=mysql_settings
)

# ClickHouse connection
clickhouse_settings = ClickhouseSettings(
    host=clickhouse_conf['host'],
    port=clickhouse_conf['port'],
    user=clickhouse_conf['user'],
    password=clickhouse_conf['password'],
    connection_timeout=clickhouse_conf.get('connection_timeout', 30),
    send_receive_timeout=clickhouse_conf.get('send_receive_timeout', 120)
)
clickhouse_api = ClickhouseApi(
    database=clickhouse_conf['database'],
    clickhouse_settings=clickhouse_settings
)

converter = MysqlToClickhouseConverter()

for table in tables:
    # Get MySQL table structure
    mysql_create_stmt = mysql_api.get_table_create_statement(table)
    mysql_struct = converter.parse_mysql_table_structure(mysql_create_stmt, required_table_name=table)
    clickhouse_struct = converter.convert_table_structure(mysql_struct)

    # Drop table in ClickHouse using clickhouse-connect
    ch_client = clickhouse_connect.get_client(
        host=clickhouse_conf['host'],
        port=clickhouse_conf['port'],
        username=clickhouse_conf['user'],
        password=clickhouse_conf['password'],
        database=clickhouse_conf['database']
    )
    ch_client.command(f"DROP TABLE IF EXISTS {table}")
    ch_client.close()

    # Create table in ClickHouse
    clickhouse_api.create_table(clickhouse_struct)

    # Dump data from MySQL to ClickHouse
    records = mysql_api.get_records(table_name=table, order_by=clickhouse_struct.primary_keys, limit=None)
    converted_records = converter.convert_records(records, mysql_struct, clickhouse_struct)
    clickhouse_api.insert(table, converted_records, table_structure=clickhouse_struct)

    print(f"Table '{table}' structure replicated and data dumped from MySQL to ClickHouse.")
```

---

## Summary

- Every ClickHouse table has a required primary key (even if forced).
- The right engine and settings are used for efficient storage and fast inserts.
- Code changes allow smooth dumping of non-primary key tables from MySQL to ClickHouse.
- Direct modification of library files enables easier future script adjustments.