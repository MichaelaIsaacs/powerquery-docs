---
title: Implement Query Folding and Advanced Patterns in ADBC-based connectors
description: Learn how to implement navigation tables, SQL dialect generation and query folding, key metadata, DirectQuery/native query support, and sample-to-template customization for ADBC connectors.
author: simplywilson
ms.topic: concept-article
ms.date: 7/22/2026
ms.author: tinglee
ms.subservice: custom-connectors
---

# Implement query folding and advanced patterns in ADBC-based connectors

This article builds on [Developing an ADBC-based Power Query connector: fundamentals](adbc.md) and focuses on implementation patterns that affect folding behavior, metadata quality, and production readiness.

This article uses the **DuckDB FlightSQL** sample as its reference implementation. The sample connects to [DuckDB](https://duckdb.org/) over the [FlightSQL](https://arrow.apache.org/docs/format/FlightSql.html) protocol via ADBC, and shows the patterns described here. For the complete source code, see the [DuckDB FlightSQL sample](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL).

## Query folding

An ADBC connector folds queries through the engine helper `SqlView.Generator`, combined with a SQL generator that describes your source's SQL dialect. `SqlView.Generator` takes the source identity, the SQL generator record, and a data-fetch callback, and returns a function. The sample calls that function with a table reference and the table's type to build each leaf table, which then folds downstream operations (such as `Table.SelectRows`, `Table.Group`, and `Table.Sort`) into dialect-specific SQL.

The SQL dialect is defined through a two-layer override architecture: a shared SQL92 base (`SqlGeneratorCommon.pqm`) and a source-specific override layer (`SqlGenerator.pqm`). This architecture is the core of an ADBC connector and is covered in detail in [SQL generator for ADBC connectors](adbc-sql-generator.md).

## Primary and foreign keys

To enable Power BI to automatically create relationships, the sample detects keys from `information_schema`:

- **Primary keys** are read from `information_schema.table_constraints` joined with `key_column_usage`, then applied to the table type with `Type.ReplaceTableKeys`.
- **Foreign keys** are discovered through `information_schema.constraint_column_usage` and exposed for relationship creation.

Power BI uses this key metadata to suggest and create relationships between the imported tables.

## DirectQuery and native query

Because the sample folds operations into SQL through `SqlView.Generator`, it supports **DirectQuery** (`SupportsDirectQuery = true` in the publish record). It also enables **native query** with folding, allowing a user-supplied SQL statement to be used as a foldable source through `NativeQueryProperties`. For background on native query support in the SDK, see [Handling native query support](native-query-sdk.md).

## Using the sample as a template

To adapt the DuckDB FlightSQL sample for a different FlightSQL-backed database, the main swap-out points are:

| File | What to change |
| --- | --- |
| `DuckDb.pq` | Connector name, `DataSource.Kind`, user-facing parameters and credential handling, default URI scheme, and the options record passed to `Adbc.Connection`. |
| `FlightSqlAdbcConfig.pqm` | Server-specific ADBC metadata: `Name`, catalog/schema support flags, identifier quoting, and supported table types. The driver location and entry point stay the same for FlightSQL. |
| `SqlGenerator.pqm` | The source's dialect rules: type facets, LIMIT/OFFSET syntax, function remapping, typed-literal formats, and supported cast rules. |
| `TypeInfo.pqm` | The source's native-type to M-type mappings. |
| `SqlGeneratorCommon.pqm` | Typically reused as-is. This file is the shared SQL92 base. |
| `resources.resx` | Display strings (connector name, descriptions, error messages). |

## Related content

- [Developing an ADBC-based Power Query connector: fundamentals](adbc.md)
- [SQL generator for ADBC connectors](adbc-sql-generator.md)
- [Transition from ODBC to ADBC drivers in Power BI and Fabric](transition-to-adbc.md)
- [Enabling DirectQuery for an ODBC-based connector](odbc.md)
- [Handling native query support](native-query-sdk.md)
- [DuckDB FlightSQL sample on GitHub](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL)
