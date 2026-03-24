# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
make                     # Release build (alias: make release, make opt)
make debug               # Debug build with ASAN/UBSAN
make reldebug            # RelWithDebInfo build
make relassert           # Release build with assertions enabled
make cldebug             # Debug build without ASAN/UBSAN
GEN=ninja make           # Use Ninja for faster builds
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make  # Limit parallel jobs
make clean               # rm -rf build
```

Extensions can be included via env vars: `BUILD_TPCH=1 make`, `BUILD_JSON=1 make`, `BUILD_ALL_EXT=1 make`, or `DUCKDB_EXTENSIONS="json;parquet" make`.

## Testing

DuckDB **strongly** prefers SQL logic tests (`.test` files) over C++ tests. Only use C++ tests for exotic behavior (concurrent connections, etc.).

```bash
make unit                # Fast unit tests (~1 min), builds debug first
make allunit             # All unit tests (~1 hour)
make unittest            # Run debug unittest binary
make smoke               # Smoke tests using test/smoke_tests.list
```

**Run a single test or filter tests:**
```bash
build/debug/test/unittest "test/sql/path/to/test.test"
build/debug/test/unittest --test-filter "pattern"
```

### SQL Logic Test Format (.test files)

```sql
# name: test/sql/function/blob/encode.test
# description: Test blob encode/decode functions
# group: [blob]

statement ok
PRAGMA enable_verification

query I
SELECT encode('ü')
----
\xC3\xBC

statement error
SELECT decode('\xFF'::BLOB)
----
```

- `statement ok` — SQL that should succeed
- `statement error` — SQL that should fail
- `query <types>` — query with expected results after `----` (I=integer, T=text, D=double, R=real)
- `.test_slow` suffix for slow tests; these run only in `allunit`

## Code Formatting

```bash
make format-fix          # Format all code (clang-format 11.0.1 + black)
make format-check        # Check formatting without modifying
make format-head         # Format only changes since HEAD
make format-main         # Format only changes vs main branch
```

Requires `clang-format==11.0.1` (`pip install clang-format==11.0.1`).

## Code Generation

```bash
make generate-files      # Run all code generators (C API, functions, serialization, settings, etc.)
```

Run this when modifying enums, settings, serializable classes, or the C API.

## Architecture

DuckDB is an in-process OLAP database. All code is in `namespace duckdb`.

### Query Processing Pipeline

```
SQL string → Parser → Planner/Binder → Optimizer → PhysicalPlanGenerator → Executor → QueryResult
```

| Stage | Directory | Description |
|-------|-----------|-------------|
| Parser | `src/parser/` | SQL parsing via libpg_query (PostgreSQL parser), produces `SQLStatement` AST |
| Planner | `src/planner/` | Binder resolves names/types against Catalog; produces `LogicalOperator` tree |
| Optimizer | `src/optimizer/` | 40+ rules: filter pushdown, join order optimization, CSE, late materialization, etc. |
| Execution | `src/execution/` | Push-based vectorized engine; produces `PhysicalOperator` pipeline |
| Parallel | `src/parallel/` | Pipeline-based parallelism with task scheduler and event-driven synchronization |

### Core Data Structures

- **Vector** (`src/include/duckdb/common/types/vector.hpp`): Column of values, the fundamental execution unit. Variants: Flat, Constant, Dictionary, List, Struct, String, FSST, Array, etc.
- **DataChunk** (`src/include/duckdb/common/types/data_chunk.hpp`): Collection of Vectors with same row count (up to STANDARD_VECTOR_SIZE=2048). The unit passed between operators.
- **LogicalType**: Type system including INT8-INT64, FLOAT, DOUBLE, VARCHAR, BLOB, DATE, TIMESTAMP, STRUCT, LIST, MAP, UNION, ARRAY.

### Key Components

| Component | Directory | Key Classes |
|-----------|-----------|-------------|
| Database/Connection | `src/main/` | `DatabaseInstance`, `Connection`, `ClientContext` |
| Catalog | `src/catalog/` | `Catalog`, `CatalogEntry`, `CatalogSet` |
| Storage | `src/storage/` | `DataTable`, `BufferManager`, column compression (RLE, Dictionary, Bitpacking, etc.) |
| Transactions | `src/transaction/` | MVCC via `DuckTransaction`, `UndoBuffer` |
| Functions | `src/function/` | `ScalarFunction`, `AggregateFunction`, `TableFunction`, `WindowFunction` |
| Core Functions | `src/core_functions/` | Built-in scalar and aggregate function implementations |

### Extension System

Extensions live in `extension/` (in-tree) or are loaded dynamically. Key in-tree extensions: parquet, json, icu, jemalloc, core_functions.

Extension entry point pattern:
```cpp
void MyExtension::Load(ExtensionLoader &loader) {
    loader.RegisterFunction(my_function);
    loader.RegisterType("my_type", my_type);
}
extern "C" { DUCKDB_CPP_EXTENSION_ENTRY(my_ext, loader) { LoadInternal(loader); } }
```

Configuration: `extension_config.cmake` (base), `extension_config_local.cmake` (local overrides).

### Function Registration Pattern

Function definitions in headers (often auto-generated in `src/include/duckdb/function/`):
```cpp
struct MyFun {
    static constexpr const char *Name = "my_func";
    static constexpr const char *Parameters = "arg1,arg2";
    static constexpr const char *Description = "Does something";
    static ScalarFunction GetFunction();
};
```

Implementation typically: BindData struct → Bind function → Execute function → register in function set.

Headers are in `src/include/duckdb/` mirroring the `src/` directory structure.

## C++ Conventions

- Use `unique_ptr` over `shared_ptr`; no raw `new`/`delete`/`malloc`
- Use `[u]int(8|16|32|64)_t` and `idx_t` (not `size_t` for indices/counts)
- Functions: `CamelCase`; variables: `snake_case`; files: `snake_case.cpp`
- Class layout: public constructor + public vars → public methods → private methods → private vars
- Tabs for indentation, spaces for alignment, 120 column limit
- Use `D_ASSERT` for programmer-error assertions (never triggered by user input)
- Exceptions only for query-terminating errors; use return values for expected errors
- Return early to avoid deep nesting; always use braces for if/loops
- Do not import namespaces (`using namespace std` is forbidden)
- Use `override`/`final` when overriding virtual methods
