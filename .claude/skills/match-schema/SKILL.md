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

## Photometry Filter IDs

Read `references/photometry-filters.md` for the full rules on resolving band names to SVO Filter
Profile Service IDs before populating `PhotometryFilters.band`.

## Output

Write results to `/tmp/schema-match-result.html` using the `Write` tool. Follow the full visual
spec in `references/html-output.md` — read it now before writing the file.

Tell the user the file path and suggest opening it with VSCode's built-in HTML preview
(right-click the file → Open Preview) or in a browser.

As part of the HTML file, also generate a **Lookup Table Checklist** section — one mini-table
per lookup table that will need new entries before ingestion can proceed. See
`references/html-output.md` for the visual spec and the rules for which lookup tables to check.

After writing the file, give a short plain-text summary in the chat (2–4 sentences) noting how
many columns matched at each confidence level and flagging anything critical.

**Confidence levels:**
- **High**: Name clearly matches a known pattern, or name + units together are unambiguous
- **Medium**: Units or description match but name is generic; or name matches but units are unexpected
- **Low**: Only a weak contextual signal; flagging as possible match with uncertainty
