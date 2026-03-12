# AstroDB Bot

Claude skills for interactive AstroDB ingestion workflows. Expects an input data table and will prepare it to be converted into a new database using the [AstroDB Template schema](https://github.com/astrodbtoolkit/astrodb-template-db) and conventions. It utilizes the [Felis system](https://felis.lsst.io/index.html) for describing database schemas.

## Skills

- `parse-data-table` — summarizes table columns, descriptions, units, and types.
- `match-schema` — maps parsed columns to [AstroDB Template](https://github.com/astrodbtoolkit/astrodb-template-db) tables and fields
- `validate-schema-mapping` - identifies problems with nulls and inconsistent data types
- `generate-felis-schema` - creates a Felis-format schema.yaml file using outputs of previous skills

## Requirements

- an AI skill runner that reads `.claude/skills/`
- `uv` or `pip` to install Python packages
- Python 3.11+
  - `astropy`
  - `pandas` for fallback table parsing
  - `lsst-felis`

## License

BSD 3-Clause. See [LICENSE](LICENSE).
