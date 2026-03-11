---
name: validate-schema-mapping
description: >
  Validate an AstroDB schema mapping by checking that data columns are compatible with the
  schema fields they've been mapped to. Use this skill whenever the user has completed a
  schema mapping (e.g., from the match-schema skill) and wants to check whether the actual
  data is safe to ingest — specifically: (1) are there null values in data columns that
  map to non-nullable schema fields, and (2) do data column types match the expected types
  in schema.yaml? Always use this skill proactively after match-schema produces a mapping,
  before the user proceeds to write ingestion code. Trigger on phrases like "validate",
  "check my mapping", "will this ingest cleanly", "are there any type mismatches", "null
  check", "nullable violations", or "verify schema compatibility".
compatibility: python, astropy, pyyaml
metadata:
  authors: ["Claude"]
---

# Validate Schema Mapping

Check that a completed column mapping is safe to ingest by comparing what the data actually
contains against what the schema requires.

Two classes of problems can block a clean ingest:
1. **Nullable violations** — the schema marks a field as `nullable: false`, but the data
   column has missing/null values. Inserting these rows will raise a database constraint error.
2. **Type mismatches** — the schema specifies a type (e.g., `double`, `string`, `boolean`)
   but the data column holds a different type (e.g., a float column mapped to a string field,
   or a string column mapped to a numeric field). These may cause silent truncation, coercion
   errors, or failed inserts.

## Inputs

You need three things:

1. **The mapping table** — the markdown table from match-schema, with columns:
   `Input Column | Description | Units | Type | DB Table | DB Field | Confidence | Notes`
   (or a similar structure — adapt if column names differ slightly)

2. **The data file path** — the original astronomical table (FITS, CSV, ECSV, HDF5, VOTable, etc.)

3. **The schema.yaml path** — the Felis-format schema file for the target database.
   Common locations to check if not provided: `schema/schema.yaml`, `schema.yaml`,
   `<db-name>/schema.yaml`. Ask the user if you can't find it.

If the user doesn't provide the data file or schema.yaml paths, ask for them before
proceeding — you cannot do this validation without both.

## Step 1: Parse the schema.yaml

Load the schema.yaml using PyYAML. For each table, extract every column's `name`, `datatype`,
`nullable` (defaults to `true` if absent), and any `length` constraint. Build a lookup
structure indexed as `schema_fields[table_name][field_name]` → `{datatype, nullable, length}`.

The Felis `datatype` values map to Python/numpy types roughly as:

| Felis datatype | Compatible Python/numpy types |
|---|---|
| `string` | `str`, `object`, `bytes` |
| `double` | `float64`, `float32`, `float16` |
| `float` | `float32`, `float64` |
| `int` / `long` | `int32`, `int64`, `int16`, `int8` |
| `boolean` | `bool` |
| `timestamp` | `datetime64`, `str` (ISO format) |

A float32 mapped to a `double` field is **compatible** (widening). A string mapped to a
`double` field is **incompatible**. Use broad compatibility rather than strict equality —
the goal is to catch real problems, not flag harmless precision differences.

## Step 2: Load the data and check each mapping

For each row in the mapping table where DB Table and DB Field are filled in (skip unmapped
rows and rows mapped to "—" or "N/A"):

Write a short Python script to:
- Load the data file (use `astropy.table.Table.read()` — it handles FITS, ECSV, CSV, VOTable, HDF5)
- For each mapped column:
  - Count null/missing values: `np.sum(np.isnan(col))` for floats, `np.sum(col == None)` / masked array checks for strings. For FITS masked arrays, check `.mask` if present.
  - Get the column's numpy dtype

Save the script to `/tmp/validate_mapping.py` and run it to get the per-column null counts and dtypes.

## Step 3: Produce the validation report

Output a markdown validation report. Use this exact structure:

```
## Schema Mapping Validation Report
Source: <filename> → <schema.yaml path>
Date: <today>

### Nullable Violations  (<N> issues)
| Data Column | Maps To | Null Count | Total Rows | % Null |
|---|---|---|---|---|
| col_name | Table.field | 42 | 1000 | 4.2% |

### Type Mismatches  (<N> issues)
| Data Column | Data Type | Maps To | Expected Type | Compatible? |
|---|---|---|---|---|
| col_name | float32 | Table.field | string | ❌ No |

### Clean Mappings  (<N> columns OK)
(List column → field pairs that passed both checks, one per line, as a collapsed summary)

### Summary
- X nullable violations found (columns with nulls in non-nullable fields)
- Y type mismatches found
- Z columns passed validation cleanly

**Next steps:**
- For nullable violations: either filter out null rows before ingest, fill with a default
  value, or ask the schema maintainer whether the field can be made nullable.
- For type mismatches: add a type-cast step in your ingestion script.
```

If there are no issues, say so clearly: "All N mapped columns passed validation."

Also write the full report to `/tmp/schema-validation-report.md` using the Write tool and
tell the user the path.

## Edge cases to handle gracefully

- **Column not in data file**: warn that the mapped column name wasn't found in the data —
  this usually means the mapping used a renamed/transformed column, not the raw column name.
- **Schema field not found**: warn if a DB Table/Field from the mapping doesn't exist in
  schema.yaml — possible if the mapping used a custom table.
- **All-null columns**: flag these explicitly — they'll violate any non-nullable constraint
  and may indicate a parsing problem upstream.
- **FITS masked arrays**: treat masked values as nulls.
- **String columns with empty strings**: count `""` as effectively null for non-nullable
  string fields — empty strings often slip through where None would be caught.
