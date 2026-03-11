---
name: match-schema
description: Match columns from an astronomical data table to fields in the AstroDB template database schema. Use this skill whenever the user wants to ingest, import, or load a data table (FITS, CSV, ECSV, etc.) into an AstroDB database, wants to know which database table or field a column belongs to, asks about schema mapping, column mapping, or data ingestion, or has output from the parse-data-table skill and wants to figure out where each column goes. This skill should also trigger when the user shares a table of columns (with names, descriptions, units, types) and asks about AstroDB, SIMPLE, or any astrodb-toolkit database. Always use this skill proactively after parse-data-table runs if the user seems to be working toward database ingestion.
compatibility: python, astropy
metadata:
  authors: ["Claude"]

---

# Match Schema

Map columns from an astronomical data table to the AstroDB template database schema, so you know
exactly which table and field each column belongs to before ingesting data.

## Input

Accept input in either form:

1. **A markdown table** (e.g., output from the `parse-data-table` skill) with columns: Column,
   Description, Units, Type
2. **A data file path** — run the `parse-data-table` skill on it first, then proceed with its output

If given a file path, invoke `parse-data-table` first and wait for its output before continuing.

## The AstroDB Template Schema

The full table and field listing is in `references/schema.md` — read it now before proceeding.
It covers all Lookup Tables, Main Tables, and Data Tables with every field name.

## Matching Strategy

Read `references/column-patterns.md` for the full matching rules. It covers three layers in order:
1. **Column name patterns** — specific known aliases for each field (strongest signal)
2. **Units** — unit-to-field lookup when the name is ambiguous
3. **Description text** — keyword scanning as a tiebreaker

It also documents how to handle uncertainty columns (`_error`, `_error_upper`, `_error_lower`) and catch-all tables (`ModeledParameters`, `CompanionParameters`) for unmapped physical parameters.

## Output

Render a table directly in your response (not in a code block) with one row per input column:

| Input Column | Description | Units | DB Table | DB Field | Confidence | Notes |
|---|---|---|---|---|---|---|

**Confidence levels:**
- **High**: Name clearly matches a known pattern, or name + units together are unambiguous
- **Medium**: Units or description match but name is generic; or name matches but units are unexpected
- **Low**: Only a weak contextual signal; flagging as possible match with uncertainty

After the table, add two sections:

### Unmatched columns
List any columns with no match and briefly explain why. Suggest whether `ModeledParameters` or `CompanionParameters` could accommodate them.

### Ingestion notes
Point out any important issues:
- Columns that need **unit conversion** before ingestion (e.g., parallax in arcsec → mas, period in days → hours)
- Columns where the same input column might map to **multiple rows** in the DB (e.g., one photometry column per band becomes a separate row in the Photometry table)
- Columns that appear to be **duplicates** of each other (e.g., two RA columns)
- Any **required fields** that are missing from the input (e.g., `source` identifier is required in every data table row — flag if no obvious source column exists)
