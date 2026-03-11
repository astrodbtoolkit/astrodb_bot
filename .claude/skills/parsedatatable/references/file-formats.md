# Supported File Formats

`astropy.table.Table.read()` handles most of these automatically. Fall back to `pandas` if needed.

| Extension(s) | Format |
|---|---|
| `.fits`, `.fit`, `.fz` | FITS |
| `.csv`, `.tsv`, `.txt` | CSV / TSV |
| `.dat`, `.txt`, `.ascii` | ASCII / fixed-width |
| `.ecsv` | ECSV (Enhanced CSV with metadata) |
| `.hdf5`, `.h5` | HDF5 |
| `.xml`, `.vot` | VOTable |
| `.xlsx`, `.xls` | Excel |
| `.parquet` | Parquet |
| `.json` | JSON |

For CSV, TSV, and plain text, column descriptions usually aren't embedded in the file — leave description as "—".
