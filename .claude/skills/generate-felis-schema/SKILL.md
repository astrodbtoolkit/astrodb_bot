---
name: generate-felis-schema
description: Generate a Felis YAML schema for a user-provided astronomical data file, using the output of the match-schema and validate-schema-mapping skills. Produces a standards-compliant schema.yaml covering each mapped table and column, with proper Felis syntax (@id references, datatypes, nullable flags, units, and foreign key constraints). Always use this skill when the user has completed a schema mapping (with or without validation) and wants to produce a Felis YAML, create a schema file for their data, generate schema.yaml, export their mapping as a schema, or document their database tables. Also trigger when the user says "generate schema", "create felis yaml", "make a schema file", or "turn this mapping into a schema".
compatibility: python, pyyaml
metadata:
  authors: ["Claude"]
---

# Generate Felis Schema

Take a completed AstroDB schema mapping — and optionally a validation report — and produce a
well-formed Felis YAML schema file covering all the tables and columns that will be populated
by the user's data.

## What is Felis YAML?

Felis is the schema description language used by AstroDB and LSST. A Felis YAML file defines
tables, columns, datatypes, constraints, and metadata in a format that tools can use to create
and validate databases. The canonical spec is at https://felis.lsst.io/.

Read `references/felis-syntax.md` for the exact syntax rules and examples before writing any YAML.

## Inputs

You need at minimum:

1. **The mapping table** from `match-schema` — rows like:
   `Input Column | Description | Units | Type | DB Table | DB Field | Confidence | Notes`

2. **Schema name** — what to call the new schema (e.g. the dataset name or survey name).
   If not provided, derive it from the data file name or ask the user.

Optionally also accept:

3. **The validation report** from `validate-schema-mapping` — identifies nullable violations
   and type mismatches. If provided, use it to set `nullable` flags and resolve type conflicts.

4. **The data file path** — used to infer datatypes for any columns that need them.

If the user hasn't run `validate-schema-mapping` yet, note that the schema will be generated
without null/type checks, and suggest they validate before ingesting.

## Step 1: Identify unmatched and problematic columns

Before generating any YAML, audit the mapping table:

**Unmatched columns**: rows where DB Table or DB Field is blank, "—", "N/A", or "unmatched".

**Problematic columns** (if validation report is present):
- Nullable violations: a column has nulls but its mapped field is `nullable: false`
- Type mismatches: column dtype is incompatible with the schema field's datatype

For each category, **ask the user** how to handle them before proceeding. Present all
unmatched/problematic columns in a single message — one question per category, not one per column.

### Unmatched column options (ask the user to choose for each, or apply a default):
- **Skip it** — don't include it in the schema (good for metadata columns, row numbers, etc.)
- **Add to an existing table** — user specifies which table and what field name to use
- **Create a new table** — user provides a table name and any other columns it should have
- **Flag only** — include it as a commented-out stub in the YAML so it's documented but inactive

### Problematic column options:
- **Nullable violation**: mark the field `nullable: true` in the new schema, OR exclude that column
- **Type mismatch**: use the data's actual type in the new schema, OR use the schema's expected type
  (with a note that a cast will be needed at ingest)

If there are no unmatched or problematic columns, skip this step and proceed directly to generation.

## Step 2: Group columns by table

Group all columns that will appear in the schema by their DB Table. Each group becomes one
table block in the YAML.

For each table:
- Collect: table name, all mapped columns (with their DB Field names, types, units, descriptions)
- Note which column(s) form the primary key. If not obvious from the mapping, ask the user.
  Common defaults: `source` or `source_id` for data tables; the main identifier column otherwise.

## Step 3: Generate the Felis YAML

Follow the syntax in `references/felis-syntax.md` exactly. Key rules:
- Schema-level `@id` = `"#<SchemaName>"`
- Table `@id` = `"#<TableName>"`
- Column `@id` = `"#<TableName>.<columnName>"`
- `primaryKey` is a list of column `@id` strings
- Use the data column's actual numpy/astropy type, mapped to a Felis datatype
- Include `fits:tunit` for any column that has units
- Include `ivoa:ucd` where you can infer it confidently (see `references/felis-syntax.md`)
- Add foreign key constraints for any `source`, `reference`, `telescope`, `instrument`
  columns that link to standard AstroDB lookup tables
- **Always include stub definitions for lookup tables that are referenced via FK** —
  even if the data file doesn't populate them directly. At minimum define `Sources`
  (with `source` column) and `Publications` (with `reference` column) whenever those
  FKs appear. Without a table definition in the same schema file, the FK constraint
  references a table that doesn't exist in the document, which will fail Felis validation.

### Datatype mapping (data type → Felis datatype):
| Data type | Felis datatype |
|---|---|
| float64, float32 | `double` |
| float32 only | `float` |
| int64, int32 | `long` / `int` |
| int16, int8 | `short` / `byte` |
| bool | `boolean` |
| str, object, bytes | `string` (add `length`) |
| datetime64 | `timestamp` |

For `string` columns, set `length` to the observed max string length rounded up to a
reasonable ceiling (e.g., 30, 50, 100, 256). If you can't observe the data, use 100 as a default.

### Nullable rules:
- Default: `nullable: true` (omit the field — Felis treats absent as true)
- Set `nullable: false` only for primary key columns and columns the user confirms are required
- If validation found a nullable violation and the user chose to relax it: set `nullable: true`

## Step 4: Write the schema file

Save the generated YAML to `/tmp/<schema-name>-schema.yaml` using the Write tool.

Also print the full YAML in a code block in the chat so the user can review it inline.

Tell the user:
- The file path
- How many tables and columns were included
- Which columns were skipped or flagged (if any)
- Whether any assumptions were made (e.g., inferred primary keys, default string lengths)

Suggest next steps: validating the schema with `felis validate`, then using it to create the database.
