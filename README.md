# AstroDB Bot

Agent Skills for interactive AstroDB ingestion workflows. Expects an input data table and will prepare it to be converted into a new database using the [AstroDB Template schema](https://github.com/astrodbtoolkit/astrodb-template-db) and conventions. It utilizes the [Felis system](https://felis.lsst.io/index.html) for describing database schemas. Also uses `astrodbkit` and `astrodb_utils` packages for setting up and interacting with the database.

To install this in another agent, you can rename/copy the `.claude` directory to whatever is appropriate for the agent (eg, `.cursor` for Cursor) or configure your agent to use this directory.

## Skills

- [`parse-data-table`](.claude/skills/parse-data-table/SKILL.md) — summarizes table columns, descriptions, units, and types.
- [`match-schema`](.claude/skills/match-schema/SKILL.md) — maps parsed columns to [AstroDB Template](https://github.com/astrodbtoolkit/astrodb-template-db) tables and fields
- [`validate-schema-mapping`](.claude/skills/validate-schema-mapping/SKILL.md) — identifies problems with nulls and inconsistent data types
- [`generate-felis-schema`](.claude/skills/generate-felis-schema/SKILL.md) — creates a Felis-format schema.yaml file using outputs of previous skills
- [`create-astrodb`](.claude/skills/create-astrodb/SKILL.md) — Create an empty SQLite AstroDB database from a Felis-validated schema.yaml, following the astrodb-template-db file structure.

## Requirements

- an AI skill runner that reads `.claude/skills/`
- `uv` or `pip` to install Python packages
- Python 3.11+
  - `astropy`
  - `pandas` for fallback table parsing
  - `lsst-felis`
  - `astrodbkit`
  - `astrodb_utils`
  - `pytest`

## License

BSD 3-Clause. See [LICENSE](LICENSE).
