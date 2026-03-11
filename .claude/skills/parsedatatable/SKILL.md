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

### Step 2: Figure out how to read in the file

The file may be in any common tabular format, including but not limited to:
- **FITS** (`.fits`, `.fit`, `.fz`)
- **CSV / TSV** (`.csv`, `.tsv`, `.txt`)
- **ASCII / fixed-width** (`.dat`, `.txt`, `.ascii`)
- **ECSV** (`.ecsv` — Enhanced CSV with metadata)
- **HDF5** (`.hdf5`, `.h5`)
- **VOTable** (`.xml`, `.vot`)
- **Excel** (`.xlsx`, `.xls`)
- **Parquet** (`.parquet`)
- **JSON** (`.json`)

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

### Step 3: Extract column information

For each column, extract:
- **Column name**
- **Description** (from metadata/comments; use "—" if not available)
- **Units** (use "—" if not specified)
- **Data type** (e.g. `float64`, `int32`, `str`)

**Important:** `t[col].description` may be empty even when descriptions exist in the file. Always check format-specific metadata sources:

#### FITS files
FITS BINTABLEs store column descriptions in `TCOMMn` header keywords, and units in `TUNITn`. Read them directly:

```python
from astropy.io import fits

with fits.open("$ARGUMENTS") as hdul:
    hdr = hdul[1].header  # BINTABLE is usually extension 1
    n_cols = hdr['TFIELDS']
    for i in range(1, n_cols + 1):
        name = hdr.get(f'TTYPE{i}', '')
        desc = hdr.get(f'TCOMM{i}', None)
        unit = hdr.get(f'TUNIT{i}', None)
```

Also check for embedded VOTable XML metadata — some FITS files store richer descriptions there. If the PRIMARY HDU header has `VOTMETA = T`, extract it:

```python
with fits.open("$ARGUMENTS") as hdul:
    if hdul[0].header.get('VOTMETA'):
        xml_str = hdul[0].data.tobytes().decode('utf-8', errors='replace')
        # parse <FIELD name="..."><DESCRIPTION>...</DESCRIPTION></FIELD> elements
```

#### ECSV files
Descriptions and units are in the YAML header at the top of the file. `astropy` usually populates `t[col].description` and `t[col].unit` for these.

#### Other formats
For CSV, TSV, and plain text, descriptions usually aren't embedded — leave as "—".

### Inferring missing descriptions

When a column has no description in the file metadata, try to infer one from context if you can do so confidently:

- Columns ending in `+` or `_plus` are typically upper uncertainties on the base column (e.g. `dmod+` → "Upper uncertainty on dmod")
- Columns ending in `-` or `_minus` are typically lower uncertainties (e.g. `dmod-` → "Lower uncertainty on dmod")
- Columns with `err`, `e_`, or `sig` prefixes/suffixes usually denote errors or standard deviations — use the base column's description to construct the inferred description

If you can't infer a description confidently, leave it as "—".

### Inferring missing units

When a column has no units in the file metadata, try to infer them from the column name and description. Use `astropy.units` string conventions. Common patterns:

| Clue in name or description | Unit |
|-----------------------------|------|
| velocity, `km/s`, `(km/s)` | `km / s` |
| magnitude, `mag`, surface brightness | `mag` or `mag / arcsec2` |
| half-light radius, `arcmin` | `arcmin` |
| `arcsec` | `arcsec` |
| proper motion, `mas/yr` | `mas / yr` |
| metallicity, `dex`, `[Fe/H]` | `dex` |
| solar masses, `M_sun`, `M☉` | `solMass` (or `1e+06 solMass` if scaled) |
| distance modulus | `mag` |
| reddening, `E(B-V)` | `mag` |
| position angle | `deg` |
| right ascension, declination (if decimal degrees) | `deg` |
| flag, index, coded integer values | `—` |
| dimensionless quantities (e.g. ellipticity, ratios) | `dimensionless_unscaled` |

If you can't infer units confidently from the description, fall back to inheriting from the base column: for uncertainty columns (ending in `+` or `-`), look up the already-resolved unit of the base column. Track resolved units as you process each column so they're available when processing the corresponding uncertainty columns.

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
