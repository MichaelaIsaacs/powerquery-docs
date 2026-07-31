---
title: SQL generator for ADBC connectors
description: Describes how to implement query folding for an ADBC-based Power Query connector by using the SqlView.Generator engine helper and a layered SQL generator.
author: simplywilson
ms.topic: concept-article
ms.date: 7/22/2026
ms.author: tinglee
ms.subservice: custom-connectors
---

# SQL generator for ADBC connectors

A connector built on [Arrow Database Connectivity (ADBC)](adbc.md) achieves query folding through the engine helper `SqlView.Generator` and a *SQL generator*, a record that describes how M operations translate into your source's SQL dialect. This article explains the SQL generator architecture used by the [DuckDB FlightSQL sample](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL) and how to adapt it for other backends.

The pattern is built for reuse. A shared SQL92 base supplies the common infrastructure, and each connector layers a small set of dialect-specific overrides on top, so most of the folding logic is written once and shared across ADBC connectors.

> [!TIP]
> For the end-to-end ADBC connector overview, see [Developing an ADBC-based Power Query connector: fundamentals](adbc.md) and [Developing an ADBC-based Power Query connector: query folding and advanced patterns](adbc-advanced-patterns.md).

> [!NOTE]
> This article assumes you're familiar with [Developing an ADBC-based connector](adbc.md) and [query folding](query-folding-basics.md). It complements the ODBC approach described in [Parameters for Odbc.DataSource](odbc-parameters.md), where folding is customized through `AstVisitor` and `SqlCapabilities` records.

## The SqlView.Generator helper

`SqlView.Generator` is the engine-provided function that plugs a SQL dialect into M's query-folding pipeline. It takes three arguments:

```powerquery-m
SqlView.Generator(id, generator, getData)
```

- **`id`** is a record that uniquely identifies the source. The sample builds it with `MakeUniqueIdentifier(ConnectionString)`, producing `[Module, Signature]` where `Signature` is the connection string. The engine uses this identity to tell one source instance apart from another.
- **`generator`** is the SQL generator record exported by `SqlGenerator.pqm`. It's produced by merging DuckDB's dialect overrides onto the shared SQL92 base with `MergeOverrides`, then layering in the extended `AstVisitor`. It holds the source's dialect details: capabilities, function translations, type facets, and literal formats.
- **`getData`** is a callback that executes a generated SQL string and returns a table. In the sample, this callback runs the SQL through the ADBC `ExecuteQuery` function and reshapes the result to the expected column types.

The table returned by `SqlView.Generator` folds downstream operations such as `Table.SelectRows`, `Table.SelectColumns`, `Table.Group`, and `Table.Sort` into the dialect's SQL, instead of evaluating them in the M engine. In the sample, the leaf table for each navigation node is produced like this:

```powerquery-m
// SqlView.Generator returns a generator function, which is stored on the shared context record.
SqlGen = SqlView.Generator(UniqueIdentifier, DuckDbSqlGenerator[SqlGenerator], GetData),
...
context = [ ..., SqlGenerator = SqlGen, ... ],
...
// Each leaf table invokes that function with a table reference and the table's type.
withSqlView = context[SqlGenerator](tableReference, tableType, [])
```

## Two-layer override architecture

The generator uses two layers:

| Layer | File | Role |
| --- | --- | --- |
| **SQL92 base** | `SqlGeneratorCommon.pqm` | Shared infrastructure: SQL92 capabilities, AST helper functions, type validation, and a large set of base function translations. |
| **Dialect overrides** | `SqlGenerator.pqm` | Source-specific behavior: type facets, LIMIT/OFFSET syntax, function remapping, typed-literal generation, and supported cast rules. |

Any ADBC connector can reuse `SqlGeneratorCommon.pqm` as-is. It exposes a `MergeOverrides` function, a library of AST construction `Helpers` (for example `Invocation`, `CastSqlExpression`, `BinaryOperation`, `Literal`), `Constants`, and `FunctionNames`. It also registers the SQL92 base configuration under the ID `Sql92`.

`SqlGenerator.pqm` loads the base with `Extension.LoadExpression()`, defines an `Override` record, and calls `MergeOverrides` to produce the final generator:

```powerquery-m
SqlGeneratorHelpers = Extension.LoadExpression("SqlGeneratorCommon.pqm"),
...
Override = [
    SqlGetTypeInfo = SqlGetTypeInfo,
    SqlCapabilities = [LimitClauseKind = LimitClauseKind.LimitOffset, FractionalSecondsScale = 6],
    TimestampFunctionOverrides = AdbcTimestampFunctionOverrides,
    FunctionOverrides = AdbcFunctionOverrides,
    BinaryOperatorOverrides = [],
    UnaryOperatorOverrides = [],
    DefaultTypes = DefaultTypes,
    SupportedConversions = SupportedConversions,
    SqlTypesCategories = SqlTypesCategories
],
SqlGenerator = SqlGeneratorHelpers[MergeOverrides]("Sql92", Override, false)
```

`MergeOverrides(generatorId, overrides, applyAssert)` merges your override record field-by-field onto the named base configuration. The `applyAssert` flag enables type validation of the overrides. Set it to `false` in production code.

## What you can override

The override record that you pass to `MergeOverrides` accepts the following fields. Each field merges onto the SQL92 base, so you only need to specify the parts that differ from standard SQL92.

### SqlCapabilities

A record of capability flags. The most common override is the `LimitClauseKind`, which tells the engine how to express row limits. The SQL92 base defaults to `LimitClauseKind.None`. The DuckDB sample sets `LimitClauseKind.LimitOffset` so that `Table.FirstN` and `Table.Skip` fold to `LIMIT n OFFSET m`. Other capability fields include `FractionalSecondsScale`, `IdentifierQuoteChar`, `Sql92Conformance`, and the supported predicates and join operators.

### FunctionOverrides

A record that maps M functions to SQL. Two styles are supported:

- **Simple overrides** map a function to a SQL function name (or a `[Name, Type]` record). For example, the base maps `Text.Upper` to `UPPER` and `Date.Year` to `year`.
- **Complex overrides** are functions of the form `(visitor, rowType, groupKeys, ast) => ...` that inspect the abstract syntax tree (AST) and build SQL using the helper functions from `SqlGeneratorCommon.pqm`. Complex overrides are required when a translation isn't a straight name swap.

The DuckDB sample uses complex overrides for cases such as:

- **Function remapping**: `Text.PositionOf` to `INSTR` (with an offset correction because M is 0-based and SQL `INSTR` is 1-based), `Text.StartsWith` to `starts_with`, and `Text.Middle` to `SUBSTRING`.
- **Aggregations**: folding `List.Sum`, `List.Average`, `List.Max`, and `List.Count` into `SUM`, `AVG`, `MAX`, and `COUNT` when group keys are present.
- **Date arithmetic**: folding `Date.AddMonths`, `Date.StartOfMonth`, `Date.EndOfYear`, and similar functions into DuckDB's `date_add`/`date_diff` with `INTERVAL` syntax.

### TimestampFunctionOverrides

A record that supplies dialect-specific implementations of `timestampadd` and `timestampdiff`. DuckDB doesn't have those functions, so the sample remaps them to `date_add` and `date_diff` with `INTERVAL` syntax.

### DefaultTypes and SupportedConversions

`DefaultTypes` maps M types to the source's default SQL type names (for example, `Int64.Type` to `BIGINT`, `Text.Type` to `VARCHAR`). `SupportedConversions` is a table declaring which source type can be cast to which other types, so the engine only folds casts the source actually supports.

### SqlTypesCategories

Categorizes type names into groups (such as `SoftBase2Types`, `VarCharTypes`, and `VarBinaryTypes`) used by the engine for numeric compatibility and concatenation-overflow handling.

### Extending the AstVisitor

`MergeOverrides` handles the structural translation of a query, but it doesn't decide the exact text used for literal values or for the row-limit clause. The sample fills those gaps by extending the generator's `AstVisitor` with two functions: a `Constant` visitor and a `LimitClause`.

#### Typed literals (the Constant visitor)

A *typed literal* is a constant value written in the form the source expects, rather than as a bare value. For example, when a query filters on a date, the engine can't emit `= '2023-01-01'`; DuckDB needs `= DATE '2023-01-01'`. The `Constant` visitor produces that text by formatting each value according to its type:

- `DATE '2023-01-01'`, `TIMESTAMP '...'`, and `TIME '...'` for temporal values.
- `CAST(value AS TYPE)` for numeric and string types.
- `true`/`false` for booleans.

The `Constant` visitor does a per-type lookup: it reads the value's native SQL type name and applies the matching formatter to the literal. So a filter such as `Table.SelectRows(employees, each [HireDate] = #date(2023, 1, 1))` folds the constant to `DATE '2023-01-01'` in the generated `WHERE` clause.

The type name comes from the *type facets*. Each M type is stamped with its source-native type name (and precision or scale where relevant) through `Type.ReplaceFacets`:

```powerquery-m
#"DATE"   = Type.ReplaceFacets(Date.Type, [NativeTypeName = "DATE"]),
#"DOUBLE" = Type.ReplaceFacets(Double.Type, [NativeTypeName = "DOUBLE", NumericPrecisionBase = 2, NumericPrecision = 53])
```

With these facets in place, the generator can pick the correct `CAST` target and literal format for each column. The DuckDB sample defines facets for all 24 supported types.

#### Row limits (the LimitClause)

The [`SqlCapabilities`](#sqlcapabilities) flag `LimitClauseKind.LimitOffset` declares *that* the source uses `LIMIT`/`OFFSET`. The `LimitClause` function supplies the actual text, and reports where the engine should place it through the `Location` field:

```powerquery-m
LimitClause = (skip, take) =>
    if skip = 0 and take = null then error "NoopSkipTakeError"
    else
        let
            limitPart = if take <> null then Text.Format("LIMIT #{0}", {take}) else "",
            offsetPart = if skip <> null and skip > 0 then Text.Format("OFFSET #{0}", {skip}) else "",
            delimiter = if limitPart <> "" and offsetPart <> "" then " " else ""
        in
            [Text = limitPart & delimiter & offsetPart, Location = "AfterQuerySpecification"]
```

Here `Table.FirstN` supplies `take` and `Table.Skip` supplies `skip`, so a `Table.FirstN(t, 5)` folds to a `LIMIT 5` appended after the query body.

## Adapting the generator for another source

Because `SqlGeneratorCommon.pqm` carries the shared SQL92 base, adapting the sample for a different backend mostly comes down to editing `SqlGenerator.pqm`:

1. Replace the **type facets** and `DefaultTypes` with your source's type system.
2. Set the **`LimitClauseKind`** (and `LimitClause` text, if needed) to match your source's row-limit syntax.
3. Replace **function and timestamp overrides** with your dialect's function names and behaviors.
4. Update **typed-literal formats** in the `Constant` visitor.
5. Adjust **`SupportedConversions`** to the casts your source allows.

Reuse `SqlGeneratorCommon.pqm` as-is unless your source needs changes to the shared SQL92 base.

## Related content

- [Developing an ADBC-based connector](adbc.md)
- [Implement Query Folding and Advanced Patterns in ADBC-based connectors](adbc-advanced-patterns.md)
- [Parameters for Odbc.DataSource](odbc-parameters.md)
- [Query folding basics](query-folding-basics.md)
- [Handling native query support](native-query-sdk.md)
- [DuckDB FlightSQL sample on GitHub](https://github.com/Microsoft/DataConnectors/tree/master/samples/DuckDbFlightSQL)
