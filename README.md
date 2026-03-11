# AstroDB Bot

Claude skills for interactive AstroDB ingestion workflows. Expects an input data table and will prepare it to be converted into a new database using the [AstroDB Template schema](https://github.com/astrodbtoolkit/astrodb-template-db) and conventions.

## Skills

- `parse-data-table` — summarizes table columns, descriptions, units, and types.
- `match-schema` — maps parsed columns to [AstroDB Template](https://github.com/astrodbtoolkit/astrodb-template-db) tables and fields

## Requirements

- an AI skill runner that reads `.claude/skills/`
- Python 3
- `astropy`
- `pandas` for fallback table parsing
- `uv` or `pip` to install Python packages

## License

BSD 3-Clause. See [LICENSE](LICENSE).
