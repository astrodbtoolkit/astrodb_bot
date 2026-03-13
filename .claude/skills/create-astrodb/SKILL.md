---
name: create-astrodb
description: Create an empty SQLite AstroDB database from a Felis-validated schema.yaml, following the astrodb-template-db file structure. Use this skill whenever the user wants to create a database, initialize a SQLite database, build an AstroDB, or has just finished generating a Felis schema and wants to turn it into a working database. Always trigger after generate-felis-schema completes, or when the user says "create database", "make database", "initialize database", "create sqlite", "make sqlite", "build the database", "create astrodb", "initialize astrodb", or "set up the database". Do NOT skip this skill just because a schema.yaml already exists — this skill is exactly what handles that case.
compatibility: python, astrodbkit, felis
---

# Create AstroDB Database

Take a Felis-validated `schema.yaml` and create an empty SQLite database following the
[astrodb-template-db](https://github.com/astrodbtoolkit/astrodb-template-db) file structure,
using `astrodbkit`.

## Prerequisites

This skill requires a schema.yaml that has **passed** `felis validate`. The generate-felis-schema
skill always runs this validation as its final step, so if the user just completed that workflow
the schema is already validated. If there is any doubt, validate before proceeding.

## Step 1: Locate the schema.yaml

Check (in order):
1. A path the user explicitly stated in the conversation
2. `/tmp/<schema-name>-schema.yaml` — the default output of generate-felis-schema
3. `schema.yaml` in the current working directory

If you cannot find the file, ask the user for the path before continuing.

## Step 2: Confirm felis validation

Run `felis validate` on the schema to confirm it is valid. If the project has a `.venv/`:

```bash
.venv/bin/felis validate <schema-path>
# or
felis validate <schema-path>
```

**If validation fails:** show the error to the user and stop — do not create the database from
a broken schema. Offer to go back to generate-felis-schema to fix the issue.

**If validation passes:** proceed.

## Step 3: Read the schema name

The database name comes from the top-level `name:` field in the schema YAML. For example:

```yaml
name: MyDataset
"@id": "#MyDataset"
```

This yields `MyDataset.sqlite` as the database file name.

Use Python or shell to extract it:
```bash
python -c "import yaml; d=yaml.safe_load(open('<schema-path>')); print(d['name'])"
```

## Step 4: Determine the project root

The project root is the directory where the new database will live — typically the current working
directory (where the user's project is). Confirm this is correct; if the user wants it elsewhere,
use that path instead.

The final layout will follow astrodb-template-db:
```
<project-root>/
├── schema.yaml              ← copy of the validated schema
├── <db-name>.sqlite         ← the new empty database
├── database.toml            ← config file
└── data/
    ├── reference/           ← lookup table JSON files (initially empty)
    └── source/              ← source JSON files (initially empty)
```

## Step 5: Create directory structure and config

Create the directories:
```bash
mkdir -p <project-root>/data/reference
mkdir -p <project-root>/data/source
```

Copy the schema.yaml to the project root (if it is not already there):
```bash
cp <schema-path> <project-root>/schema.yaml
```

Create `database.toml` if it does not already exist:

```toml
db_name = "<db-name>"
data_path = "data/"
felis_path = "schema.yaml"
lookup_tables = []
```

Do not overwrite an existing `database.toml`.

## Step 6: Create the empty SQLite database

Use the bundled script `scripts/create_db.py`. Run it with `uv run` so the project's virtualenv
is used:

```bash
uv run python <skill-dir>/scripts/create_db.py \
  --schema <project-root>/schema.yaml \
  --db-path <project-root>/<db-name>.sqlite
```

The script calls `astrodbkit.astrodb.create_database()` and prints success or an error traceback.

If `uv` is not available, try:
```bash
python <skill-dir>/scripts/create_db.py --schema ... --db-path ...
```

**If the script fails**, read the traceback carefully:
- `felis.datamodel` errors → schema is not valid Felis; offer to re-run generate-felis-schema
- `sqlite3` errors → check that the db path is writable
- `ImportError` for astrodbkit → astrodbkit is not installed; run `uv add astrodbkit`

## Step 7: Verify and report

After the script succeeds, confirm the database file exists and is non-empty:
```bash
ls -lh <project-root>/<db-name>.sqlite
```

Then tell the user:
- The database file path
- The schema.yaml location
- The database.toml location
- The data/ directory structure
- What to do next (e.g., run ingestion scripts to populate the database)

Example success message:
```
Database created: MyDataset.sqlite (32 KB)
Schema:           schema.yaml
Config:           database.toml
Data directory:   data/reference/  data/source/

Next steps: add JSON data files to data/source/ and run your ingestion scripts.
```
