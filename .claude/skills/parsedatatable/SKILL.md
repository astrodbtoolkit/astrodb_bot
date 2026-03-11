---
name: parsedatatable
description: Parse a data table file and extract column information (name, description, units, type). Supports FITS, CSV, ECSV, HDF5, VOTable, Parquet, Excel, and more. Generates a markdown table summarizing the columns.
compatibility: python, astropy, pandas
metadata:
    authors: ["Claude"]

---

# Parse Data Table

## Instructions
Parse the data table file `$ARGUMENTS` and extract column information.

### Step 1: Make sure Python is installed and the necessary libraries are available

Work through these options in order — stop at the first one that succeeds:

**Option 1: astropy is already available**
```bash
python3 -c "import astropy; print('ok')"
```
If this prints `ok`, proceed directly to Step 2.

**Option 2: use `uv` (fast, no project directory needed)**
```bash
uv run --with astropy --with pandas python3 script.py
```
If `uv` is installed, this handles everything — no separate install step required. Use this form to run all scripts in Steps 2–3.

**Option 3: `uv` is not installed — use `pip`**
```bash
pip install astropy pandas
python3 script.py
```

**Option 4: nothing works**

If none of the above work, tell the user you're unable to install the required libraries and ask them to run in an environment that has either `uv` or `pip` available.

### Step 2: Read the file

Use `astropy.table.Table.read()` first, which handles most formats automatically. Fall back to `pandas` if needed:

```python
from astropy.table import Table

try:
    t = Table.read("$ARGUMENTS")
    for col in t.columns:
        print(col, t[col].description, t[col].unit)
except Exception:
    import pandas as pd
    df = pd.read_csv("$ARGUMENTS")  # adjust reader as needed
    for col in df.columns:
        print(col, df[col].dtype)
```

See `references/file-formats.md` for the full list of supported formats.

### Step 3: Extract column information

For each column, extract:
- **Column name**
- **Description** (from metadata/comments; use "—" if not available)
- **Units** (use "—" if not specified)
- **Data type** (e.g. `float64`, `int32`, `str`)

**Important:** `t[col].description` may be empty even when descriptions exist in the file. Check format-specific metadata sources described in `references/format-specific-metadata.md`.

#### Inferring missing descriptions

When a column has no description in the file metadata, try to infer one from context:

- Columns ending in `+` or `_plus` → upper uncertainty on the base column (e.g. `dmod+` → "Upper uncertainty on dmod")
- Columns ending in `-` or `_minus` → lower uncertainty (e.g. `dmod-` → "Lower uncertainty on dmod")
- Columns with `err`, `e_`, or `sig` prefixes/suffixes → errors or standard deviations; use the base column's description to construct the inferred description

If you can't infer a description confidently, leave it as "—".

#### Inferring missing units

When a column has no units in the file metadata, consult `references/units-inference.md` for the lookup table and uncertainty-column inheritance logic.

### Step 4: Ask the user to fill in any remaining gaps

After exhausting file metadata and inference, if there are still columns with missing descriptions or units, ask the user to fill them in — but only if the number is manageable (fewer than 10). Present each missing column one at a time and wait for the user's response before moving to the next.

For example:
> Column `vrot_s` has no description. Do you know what this column represents?

> Column `e=1-b/a` has no units. What units should this column have, or is it dimensionless?

If there are 10 or more columns still missing descriptions or units, output the table as-is with "—" placeholders and note at the end how many are missing, so the user can address them separately.

### Step 5: Output the results

Output the results as a markdown table:

| Column | Description | Units | Type |
|--------|-------------|-------|------|
