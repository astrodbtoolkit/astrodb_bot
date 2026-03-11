# Format-Specific Metadata

## FITS

FITS BINTABLEs store column descriptions in `TCOMMn` header keywords, and units in `TUNITn`. Read them directly:

```python
from astropy.io import fits

with fits.open("file.fits") as hdul:
    hdr = hdul[1].header  # BINTABLE is usually extension 1
    n_cols = hdr['TFIELDS']
    for i in range(1, n_cols + 1):
        name = hdr.get(f'TTYPE{i}', '')
        desc = hdr.get(f'TCOMM{i}', None)
        unit = hdr.get(f'TUNIT{i}', None)
```

Also check for embedded VOTable XML metadata — some FITS files store richer descriptions there. If the PRIMARY HDU header has `VOTMETA = T`, extract it:

```python
with fits.open("file.fits") as hdul:
    if hdul[0].header.get('VOTMETA'):
        xml_str = hdul[0].data.tobytes().decode('utf-8', errors='replace')
        # parse <FIELD name="..."><DESCRIPTION>...</DESCRIPTION></FIELD> elements
```

## ECSV

Descriptions and units are in the YAML header at the top of the file. `astropy` usually populates `t[col].description` and `t[col].unit` automatically for these.

## CSV / TSV / Plain text

Descriptions are not embedded in these formats — leave as "—".
