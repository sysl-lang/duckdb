# duckdb

[DuckDB](https://duckdb.org) for [sysl](https://github.com/sysl-lang/sysl): an in-process analytical
database, bound through its C API. Module `sh.sysl.duckdb`.

*An independent sysl package for DuckDB. It is not affiliated with or endorsed by DuckDB's authors or trademark holders.*

```hocon
dependencies {
  duckdb { git = "github.com/sysl-lang/duckdb", version = "0.1.0" }
}
```

DuckDB has to be installed (`brew install duckdb`; elsewhere the `libduckdb` archive from duckdb.org).
It ships no `.pc` file, so the header and library directories are passed:

```
sysl test . --include-path duckdb=/opt/homebrew/include --link-path /opt/homebrew/lib
```

Written against DuckDB 1.5.6 and sysl 0.1.0-alpha.11.

## Usage

Compiled as the test `the README's example`:

```sysl
import sh.sysl.duckdb.open

val db = open().unwrap()                  // ":memory:", or a path
val con = db.connect().unwrap()
val r = con.query("SELECT i, i * 2.5::DOUBLE AS x FROM range(10) t(i)").unwrap()
var total = 0.0

loop
    val chunk = r.next() match
        Some(ch) -> ch
        None -> break

    val ids = chunk.column(0).i64s().unwrap()   // DuckDB's own int64 array, not a copy
    val xs = chunk.column(1).f64s().unwrap()

    for i in 0..<chunk.size() do total += f64(ids[i]) + xs[i]
```

**A column is a view.** DuckDB hands results back as data chunks of up to 2048 rows, each column a flat
C array plus a validity bitmask. `Column.i64s()`, `f64s()`, `i128s()` (HUGEINT), `dates()`,
`timestamps()` and the rest answer a slice pointing at that array, after checking the column's type;
`valid(row)`/`is_null(row)` read the mask. VARCHAR and BLOB are read per element (`bytes` is a view,
`text` a copy), decoding `duckdb_string_t` — inline up to 12 bytes, a pointer past that. DECIMAL is
read at whichever of i16/i32/i64/i128 DuckDB stored it in. LIST, STRUCT, MAP, ARRAY and UNION vectors
reach their children through `lists()`, `elements()` and `field(i)`, and their logical types describe
themselves through `LogicalType`.

**A view is valid only while its chunk is, and sysl does not enforce that.** The slices are made from
the `*T` DuckDB returns, and a slice of a `*T` region has no owner, so one kept past its chunk reads
freed memory (the gap is reported upstream). Copy out what must outlive the chunk.

Also bound: prepared statements with typed binds (`bind_i64`, `bind_f64`, `bind_text`, `bind_blob`,
`bind_i128`, `bind_decimal`, `bind_date`, `bind_timestamp`, `bind_interval`, `bind_null`, …), the
appender (`add_*`, `end_row`, `flush`), settings at open, and **scalar functions written in sysl**:
`Connection.register_scalar(name, params, returns, body)`, where `body` is a `&sync Fn(&Chunk, Column)
-> Result[unit, string]` that DuckDB calls from its worker threads.

Every handle is a `&T` whose destructor runs DuckDB's `duckdb_close`/`_disconnect`/`_destroy_*`
exactly once; each holds what it was made from (a connection its database, a result its connection), so
nothing is destroyed before something depending on it. Every fallible call answers `Result[_, Error]`
carrying DuckDB's own sentence and its error class.

## Tests

`sysl test .` (with the two path flags): 25 tests, each against an oracle that could disagree —

- query results cell by cell against the `duckdb` CLI (`duckdb -csv -noheader -c …`) on the same SQL,
  for every integer width, doubles, booleans, strings with commas and quotes, HUGEINT's and DECIMAL's
  edges at all four storage widths, a CSV written by sysl and a Parquet written by DuckDB's `COPY`;
- the sum of 1..1,000,000 over the views against N(N+1)/2, and DuckDB's own `SUM` as an i128;
- NULL in 25 column types; strings at 0, 1, 4, 11, 12, 13 and 38 bytes against DuckDB's own
  `duckdb_string_is_inlined`; dates and timestamps against the CLI's `date_diff`/`epoch_us`;
- 1,000,000 appended rows equal to the same rows inserted by SQL (`EXCEPT ALL` both ways);
- a sysl scalar function over 10,000,000 rows with `threads = 4`: the result, every row seen, and more
  than one thread calling it;
- error paths with exact sentences: a syntax error (also against the CLI's stderr), a bind of the wrong
  type, a bind past the last parameter, a missing file (also against the CLI), a closed connection, a
  missing table and a short row for the appender, an unknown setting, a scalar function's `Err`;
- the layout of every struct crossing to C, against the header's `sizeof`/`offsetof`.

## The survey: what is bound, what is absent, and why

`duckdb.h` (1.5.6) declares **543 functions in 25 modules**, every one of them exported by
`libduckdb.dylib` (checked with `nm -gU`). Classified:

- **expressible: 543.** Every handle is a pointer to an opaque struct; every aggregate passed by value
  (`duckdb_result`, `duckdb_hugeint`, `duckdb_decimal`, `duckdb_interval`, the date/time structs) is a
  struct sysl lays out identically (pinned by the layout test); every callback type takes handles,
  pointers and scalars only, so a named sysl function's `&f` is its address — no `@export` needed.
- **shim-able: 0.** No C is carried. `duckdb_string_t` is an untagged union, which sysl cannot spell;
  the pointer half is declared as a struct and DuckDB's own `duckdb_string_t_data` answers where the
  inline half's bytes are.
- **needing the language to move: 0 to call, 2 to expose safely.** `duckdb_vector_get_data` and
  `duckdb_vector_get_validity` are bound and answer zero-copy views, but nothing ties a view's lifetime
  to the chunk that owns the memory (see above).

| module | functions | bound | notes |
|---|---|---|---|
| database (open, close, config, instance cache) | 12 | 6 | open with settings; the instance cache is absent — one `Database` per path is the caller's arrangement |
| connection | 10 | 3 | connect, disconnect, interrupt; client context and Arrow options absent |
| query (materialized result) | 32 | 14 | columns, names, logical types, rows changed, errors, `fetch_chunk`; 7 deprecated |
| data_chunk | 7 | 4 | read side; building chunks (for `append_data_chunk`) is absent |
| vector | 26 | 11 | data, validity, list/struct/array children, string assignment for UDF output |
| logical_type | 31 | 17 | inspection of every nested kind; *constructing* nested types is absent (UDFs here take scalar types) |
| prepared_statement | 44 | 20 | every scalar `bind_*` but `bind_value`; 2 deprecated |
| appender | 40 | 15 | row-wise appends of the scalar types, `end_row`, `flush`, error data; 1 deprecated |
| datetime_helpers | 19 | 6 | date/timestamp split and join, hugeint/decimal to double |
| scalar_function | 33 | 10 | name, parameters, return type, the function, extra info, errors; bind/init callbacks, varargs, sets absent |
| value (`duckdb_value`, and the row-wise `duckdb_value_*`) | 101 | 0 | **the 24 deprecated row-wise accessors are left out deliberately**: they convert one cell per call and DuckDB has deprecated them in favour of chunks, which this binding reads directly. The 77 `duckdb_value` constructors/getters are absent — typed binds and appends cover the scalar types without allocating a value per cell |
| table_function | 36 | 0 | expressible with the scalar function's pattern (a C-allocated slot holding a `&sync Fn`, freed by DuckDB's destructor); not in 0.1.0 |
| replacement_scan | 4 | 0 | same |
| aggregate_function, cast_function, copy_function | 59 | 0 | extension-authoring surface; same pattern, not in 0.1.0 |
| pending, threading | 15 | 0 | incremental and caller-driven execution; not in 0.1.0 |
| arrow | 19 | 0 | the Arrow C data interface — wants an Arrow binding beside it; 14 deprecated |
| catalog, config_option, expression, file_system, log_storage, table_description | 55 | 0 | extension and introspection surface; not in 0.1.0 |

**106 of 543 bound.**

## Licence

ISC for the binding; DuckDB is MIT and is not carried here.
