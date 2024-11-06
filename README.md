# BemiDB

BemiDB is a Postgres read replica optimized for analytics.
It consists of a single binary that seamlessly connects to a Postgres database, replicates the data in a compressed columnar format,
and allows running complex queries using the Postgres-compatible analytical query engine.

## Contents

- [Highlights](#highlights)
- [Use cases](#use-cases)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
  - [Local disk storage](#local-disk-storage)
  - [S3 block storage](#s3-block-storage)
- [Architecture](#architecture)
- [Future roadmap](#future-roadmap)
- [Benchmark](#benchmark)
- [Development](#development)
- [Alternatives](#alternatives)
- [License](#license)

## Highlights

- **Performance**: runs analytical queries up to 2000x faster than Postgres.
- **Single Binary**: consists of a single binary that can be run on any machine.
- **Postgres Replication**: automatically syncs data from Postgres databases.
- **Compressed Data**: uses an open columnar format for tables with 4x compression.
- **Scalable Storage**: storage is separated from compute and can natively work on S3.
- **Query Engine**: embeds a query engine optimized for analytical workloads.
- **Postgres-Compatible**: integrates with any services and tools in the Postgres ecosystem.
- **Open-Source**: released under the OSI-approved license.

## Use cases

- **Run complex analytical queries like it's your Postgres database**. Without worrying about performance impact and indexing.
- **Simplify your data stack down to a single binary**. No complex setup, no data movement, no CDC, no ETL, no DW.
- **Integrate with Postgres-compatible tools and services**. Query and visualize data with BI tools, notebooks, and ORMs.
- **Have all data automatically synced into your data lakehouse**. Using Iceberg tables with Parquet data on object storage.

## Quickstart

Install BemiDB:

```sh
curl -sSL https://api.bemidb.com/install.sh | bash
```

Sync data from a Postgres database:

```sh
bemidb sync --pg-database-url postgres://postgres:postgres@localhost:5432/dbname
```

Run BemiDB database:

```sh
bemidb start
```

Run Postgres queries on top of the BemiDB database:

```sh
# List all tables
psql postgres://localhost:54321/bemidb -c "SELECT * FROM information_schema.tables"

# Query a table
psql postgres://localhost:54321/bemidb -c "SELECT COUNT(*) FROM [table_name]"
```

## Configuration

### Local disk storage

By default, BemiDB stores data on the local disk.
Here is an example of running BemiDB with default settings and storing data in a local `iceberg` directory:

```sh
bemidb start \
  --port 54321 \
  --database bemidb \
  --storage-type LOCAL \
  --iceberg-path ./iceberg \ # $PWD/iceberg/*
  --init-sql ./init.sql \
  --log-level INFO
```

### S3 block storage

BemiDB natively supports S3 storage. You can specify the S3 settings using the following flags:

```sh
bemidb start \
  --port 54321 \
  --database bemidb \
  --storage-type AWS_S3 \
  --iceberg-path iceberg \ # s3://[AWS_S3_BUCKET]/iceberg/*
  --aws-region us-east-1 \
  --aws-s3-bucket [AWS_S3_BUCKET] \
  --aws-access-key-id [AWS_ACCESS_KEY_ID] \
  --aws-secret-access-key [AWS_SECRET_ACCESS_KEY]
```

Here is the minimal IAM policy required for BemiDB to work with S3:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::[AWS_S3_BUCKET]",
                "arn:aws:s3:::[AWS_S3_BUCKET]/*"
            ]
        }
    ]
}
```

## Architecture

BemiDB consists of the following main components:

- **Database Server**: implements the [Postgres protocol](https://www.postgresql.org/docs/current/protocol.html) to enable Postgres compatibility.
- **Query Engine**: embeds the [DuckDB](https://duckdb.org/) query engine to run analytical queries.
- **Storage Layer**: uses the [Iceberg](https://iceberg.apache.org/) table format to store data in a columnar compressed Parquet files.
- **Postgres Connector**: connects to a Postgres databases to sync tables' schema and data.

<img src="/img/architecture.png" alt="Architecture" width="720px">

## Future roadmap

- [ ] Native support for complex data structures like JSON and arrays.
- [ ] Incremental data synchronization into Iceberg tables.
- [ ] Direct Postgres-compatible write operations.
- [ ] Real-time replication from Postgres using CDC.
- [ ] TLS and authentication support for Postgres connections.
- [ ] Iceberg table compaction and partitioning.
- [ ] Cache layer for frequently accessed data.
- [ ] Add support for materialized views.

## Benchmark

BemiDB is optimized for analytical workloads and can run complex queries up to 2000x faster than Postgres.

On the TPC-H benchmark with 22 sequential queries, BemiDB outperforms Postgres by a significant margin:

* Scale factor: 0.1
  * BemiDB unindexed: 2.3s 👍
  * Postgres unindexed: 1h23m13s 👎 (2,170x slower)
  * Postgres indexed: 1.5s 👍 (99.97% bottleneck reduction)
* Scale factor: 1.0
  * BemiDB unindexed: 25.6s 👍
  * Postgres unindexed: ∞ 👎 (infinitely slower)
  * Postgres indexed: 1h34m40s 👎 (220x slower)

See the [benchmark](/benchmark) directory for more details.

## Development

We develop BemiDB using [Devbox](https://www.jetify.com/devbox) to ensure a consistent development environment without relying on Docker.

To start developing BemiDB and run tests, follow these steps:

```sh
cp .env.sample .env
make install
make test
```

To run BemiDB locally, use the following command:

```sh
make up
```

To sync data from a Postgres database, use the following command:

```sh
make sync
```

## Alternatives

- PostgreSQL
  - The most loved general-purpose transactional (OLTP) database. Can run analytical queries at small scale.
  - Slow for analytical (OLAP) queries on medium and large datasets. Requires manual tuning and indexing.
- PostgreSQL + foreign data wrapper extensions (parquet_fdw, parquet_s3_fdw, etc.)
  - Allow querying external data sources like columnar Parquet files directly from PostgreSQL.
  - Not optimized query engine. Requires manual data syncing and schema mapping. Extensions may not be supported by PostgreSQL hosting providers.
- PostgreSQL + OLAP query engine extensions (pg_duckdb, pg_analytics, etc.)
  - Integrate an analytical query engine directly into PostgreSQL.
  - Cumbersome to set up and use (creating foreign tables, secrets management, calling custom functions). PostgreSQL data is not integrated and optimized. Extensions may not be supported by PostgreSQL hosting providers.
- DuckDB
  - Designed for OLAP use cases. Easy to run with a single binary.
  - Limited support in the data ecosystem (notebooks, BI tools, etc.). Requires manual data syncing and schema mapping for best performance.
- Real-time and high-volume databases (ClickHouse, Druid, etc.)
  - High-performance OLAP databases optimized for real-time analytics.
  - Require expertise to set up and manage the distributed systems. Limitations on data mutability. Steeper learning curve. Require manual data syncing and schema mapping.
- Big data query engines (Spark, Trino, etc.)
  - Distributed SQL query engines for big data analytics.
  - Complex to set up and manage a distributed query engine (ZooKeeper, JVM, etc.). Don't have a storage layer themselves. Require manual data syncing and schema mapping.
- Proprietary solutions (Snowflake, AWS Redshift, GCP BigQuery, Databricks, etc.)
  - Fully managed cloud data warehouses and lakehouses optimized for OLAP.
  - Can be expensive compared to other alternatives. Vendor lock-in, proprietary solutions. Require separate systems for data syncing and schema mapping.

## License

Distributed under the terms of the [AGPL-3.0 License](/LICENSE). If you need to modify and distribute the code, please release it to contribute back to the open-source community.
