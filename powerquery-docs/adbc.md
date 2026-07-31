---
title: Developing an ADBC-based Power Query connector
description: Provides an overview of building a Power Query connector using Arrow Database Connectivity (ADBC), using the DuckDB FlightSQL sample as a reference. Learn when to use Adbc.Connection and how to configure driver settings, connection properties, type mappings, and authentication for an ADBC-based connector.
author: simplywilson
ms.topic: concept-article
ms.date: 7/22/2026
ms.author: tinglee
ms.subservice: custom-connectors
---

# Developing an ADBC-based connector

[Arrow Database Connectivity (ADBC)](https://arrow.apache.org/docs/format/ADBC.html) provides a set of standard interfaces for interacting with [Apache Arrow](https://arrow.apache.org/) data. Because results are returned as Arrow columnar batches, ADBC is especially efficient at fetching large datasets with minimal overhead and no row-by-row serialization or copying. Power BI and Microsoft Fabric are progressively [transitioning supported connectors from embedded ODBC drivers to ADBC drivers](transition-to-adbc.md).

The M engine exposes the `Adbc.Connection` function for building custom connectors against data sources that ship an ADBC driver. Like the [Odbc.DataSource](/powerquery-m/odbc-datasource) function for ODBC, wrapping `Adbc.Connection` lets your connector inherit query-folding behavior, so the engine can push filters, projections, aggregations, and other transformations down to the data source as SQL.

> [!NOTE]
> Building an ADBC connector with query folding raises the difficulty and complexity level of your connector. This article assumes familiarity with [creating a basic custom connector](creating-first-connector.md) and with [query folding concepts](query-folding-basics.md).

> [!TIP]
> This article covers foundational concepts of ADBC-based connector development. For query folding architecture, and advanced implementation guidance, see [Implement Query Folding and Advanced Patterns in ADBC-based connectors](adbc-advanced-patterns.md).

This article uses the **DuckDB FlightSQL** sample as its reference implementation. The sample connector connects to [DuckDB](https://duckdb.org/) over the [FlightSQL](https://arrow.apache.org/docs/format/FlightSql.html) protocol via ADBC, and shows the patterns described here. For the complete source code, see the [DuckDB FlightSQL sample](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL).

## When to use ADBC

Use `Adbc.Connection` when your data source ships an ADBC driver (for example, a [FlightSQL](https://arrow.apache.org/docs/format/FlightSql.html) driver, or a database-specific driver such as Snowflake or BigQuery). If your data source only exposes an ODBC driver, use the [ODBC extensibility functions](odbc.md) instead.

The DuckDB FlightSQL sample uses the open-source [FlightSQL ADBC driver](https://github.com/apache/arrow-adbc/tree/main/go/adbc/driver/flightsql) (`libadbc_driver_flightsql.dll`), which ships with both the **Power Query SDK tools** and **Power BI Desktop**. No separate driver installation is needed for FlightSQL-backed sources. Other ADBC drivers might need to be installed and registered separately.

## The Adbc.Connection function

`Adbc.Connection` establishes a connection through an ADBC driver and returns a record that exposes an `ExecuteQuery` function used to run SQL and retrieve results. The function takes the driver configuration, a connection string, database properties, and an options record:

```powerquery-m
Connection = Adbc.Connection(
    FlightSqlAdbcConfig,     // driver configuration record (see next section)
    ConnectionString,        // record of connection properties
    [],                      // database properties
    [ConnectionPoolType = 2] // options
)
```

In the sample, user-facing parameters (`Server`, `Encryption`, and an optional `Database`) are mapped internally to ADBC connection properties. The connection URI scheme is derived from the `Encryption` parameter (`grpc://` when TLS is disabled, `grpc+tls://` when enabled), so end users never need to enter raw ADBC property keys such as `adbc.flight.sql.*`.

Credentials are supplied through the standard Power Query credential dialog and read with `Extension.CurrentCredential()`. The sample maps each authentication kind to the appropriate ADBC property:

- **Username/Password** maps to the `username` and `password` properties.
- **Key** (bearer token) maps to the `adbc.flight.sql.authorization_header` property with a `Bearer` prefix.
- **Anonymous** supplies no credential properties.

As with any connector, security and credential-related arguments must never be part of your data source function parameters, because values entered in the connect dialog are persisted to the user's query. Credentials are handled through the connector's [supported authentication methods](handling-authentication.md).

## Driver configuration

A configuration record describes ADBC driver behavior. In the sample, this record lives in `FlightSqlAdbcConfig.pqm` and specifies:

- **Driver location and entry point**: the `Folder` (`FlightSQL`), `File` (`libadbc_driver_flightsql.dll`), `DriverType` (`Unmanaged`), and `EntryPoint` (`FlightSqlDriverInit`).
- **Identifier metadata**: the `IdentifierQuoteChar` (`"`) and `CatalogNameSeparator` (`.`).
- **Catalog and schema support**: `SupportsCatalog`, `SupportsSchema`, and the supported `TableTypes` (`BASE TABLE, VIEW`).
- **Type information**: a `TypeInfo` table that maps the source's native types to M types.

```powerquery-m
FlightSqlAdbcConfig = [
    Name = "FlightSqlAdbcDuckDb",
    Folder = "FlightSQL",
    File = "libadbc_driver_flightsql.dll",
    DriverType = "Unmanaged",
    EntryPoint = "FlightSqlDriverInit",
    IdentifierQuoteChar = """",
    CatalogNameSeparator = ".",
    TableTypes = "BASE TABLE, VIEW",
    SupportsCatalog = true,
    SupportsSchema = true,
    TypeInfo = ...   // see Type mapping
]
```

## Type mapping

ADBC delivers data as Arrow columnar batches, so your connector maps the source's native types to M types. The sample keeps this mapping in `TypeInfo.pqm` and references it both from the driver configuration (`FlightSqlAdbcConfig.pqm`) and from the SQL generator (`SqlGenerator.pqm`).

The DuckDB sample maps 24 DuckDB types, including extended numeric types (`HUGEINT`), `UUID`, `JSON`, `INTERVAL`, and nested types (`ARRAY`, `STRUCT`, `MAP`). This list reflects DuckDB's type system; a connector for a different backend maps a different set of types based on that source's type system.

## Authentication

The sample declares three authentication kinds in its `DataSource.Kind` record:

| Kind | Power Query credential | ADBC mapping |
| --- | --- | --- |
| `UsernamePassword` | Username and password | `username` / `password` properties |
| `Key` | API key / bearer token | `adbc.flight.sql.authorization_header` = `Bearer <token>` |
| `Anonymous` | No credentials | None |

For more information about declaring and handling credentials, see [Handling authentication](handling-authentication.md).

## Navigation tables

The sample builds a hierarchical navigation tree (Database > Schema > Table/View) by querying the source's `information_schema` views and the `SHOW DATABASES` command. Each level is a [navigation table](handling-navigation-tables.md) whose `Data` column lazily loads the next level.

Navigation steps are wrapped in `Table.View` with an `OnSelectRows` handler, so selecting a specific database, schema, or table folds into a filtered metadata query rather than enumerating everything. This approach keeps navigation responsive even against sources with many objects.

## Related content

- [Implement Query Folding and Advanced Patterns in ADBC-based connectors](adbc-advanced-patterns.md)
- [SQL generator for ADBC connectors](adbc-sql-generator.md)
- [Transition from ODBC to ADBC drivers in Power BI and Fabric](transition-to-adbc.md)
- [Enabling DirectQuery for an ODBC-based connector](odbc.md)
- [Handling native query support](native-query-sdk.md)
- [DuckDB FlightSQL sample on GitHub](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL)
