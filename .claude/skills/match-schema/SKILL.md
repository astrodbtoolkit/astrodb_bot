---
name: match-schema
description: >
  Match columns from an astronomical data table to fields in the AstroDB template database schema.
  Use this skill whenever the user wants to ingest, import, or load a data table (FITS, CSV, ECSV,
  etc.) into an AstroDB database, wants to know which database table or field a column belongs to,
  asks about schema mapping, column mapping, or data ingestion, or has output from the parse-data-table
  skill and wants to figure out where each column goes. This skill should also trigger when the user
  shares a table of columns (with names, descriptions, units, types) and asks about AstroDB, SIMPLE,
  or any astrodb-toolkit database. Always use this skill proactively after parse-data-table runs if
  the user seems to be working toward database ingestion.
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

The schema follows the [Felis](https://felis.lsst.io) format. All tables and their fields:

### Lookup Tables
| Table | Fields |
|---|---|
| Publications | reference, bibcode, doi, description |
| Telescopes | telescope, description, reference |
| Instruments | instrument, mode, telescope, description, reference |
| PhotometryFilters | band, ucd, effective_wavelength_angstroms, width_angstroms |
| Versions | version, start_date, end_date, description |
| RegimeList | regime, description |
| AssociationList | association, association_type, comments, reference |
| ParameterList | parameter, description |
| CompanionList | companion, description |
| SourceTypeList | source_type, comments |

### Main Tables
| Table | Fields |
|---|---|
| Sources | source, ra_deg, dec_deg, epoch_year, equinox, reference, other_references, comments |
| Names | source, other_name |
| Positions | source, ra_deg, dec_deg, epoch_year, reference |

### Data Tables
| Table | Fields |
|---|---|
| Photometry | source, band, magnitude, magnitude_error, magnitude_error_upper, magnitude_error_lower, telescope, epoch, comments, reference, regime |
| Parallaxes | source, parallax_mas, parallax_error, parallax_error_upper, parallax_error_lower, adopted, comments, reference |
| RadialVelocities | source, rv_kms, rv_error, rv_error_upper, rv_error_lower, adopted, comments, reference |
| ProperMotions | source, pm_ra, pm_ra_error, pm_ra_error_upper, pm_ra_error_lower, pm_dec, pm_dec_error, pm_dec_error_upper, pm_dec_error_lower, adopted, comments, reference |
| RotationalParameters | source, period_hr, period_error, period_error_upper, period_error_lower, v_sin_i_kms, v_sin_i_error, v_sin_i_error_upper, v_sin_i_error_lower, inclination, inclination_error, inclination_error_upper, inclination_error_lower, adopted, comments, reference |
| Morphology | source, position_angle_deg, position_angle_error, position_angle_error_upper, position_angle_error_lower, ellipticity, ellipticity_error, ellipticity_error_upper, ellipticity_error_lower, half_light_radius_arcmin, half_light_radius_error, half_light_radius_error_upper, half_light_radius_error_lower, adopted, comments, reference |
| Spectra | source, access_url, original_spectrum, local_spectrum, regime, telescope, instrument, mode, observation_date, comments, reference, other_references |
| CompanionRelationships | source, companion, relationship, projected_separation_arcsec, projected_separation_error, projected_separation_error_upper, projected_separation_error_lower, comments, reference, other_companion_names |
| CompanionParameters | source, companion, parameter, value, error, error_upper, error_lower, unit, comments, reference |
| Associations | source, association, membership_probability, comments, adopted, reference |
| SourceTypes | source, source_type, comments, adopted, reference |
| ModeledParameters | source, model, parameter, value, error, error_upper, error_lower, unit, comments, reference |

## Matching Strategy

Work through three layers of evidence in order, combining signals to arrive at the best match.

### Layer 1: Column name patterns (strongest signal)

**Identifiers:**
- `source`, `name`, `id`, `designation`, `target`, `obj`, `object` → `Sources.source` (primary identifier) or `Names.other_name` (alternate name). If the table has multiple name-like columns, the primary/canonical one goes to Sources, the rest to Names.

**Coordinates:**
- `ra`, `ra_deg`, `RA`, `Right_Ascension`, `RAJ2000` → `Sources.ra_deg`
- `dec`, `dec_deg`, `Dec`, `Declination`, `DEJ2000` → `Sources.dec_deg`
- `epoch`, `epoch_year` (standalone, not with a measurement) → `Sources.epoch_year`

**Parallax:**
- `parallax`, `plx`, `parallax_mas`, `pi` → `Parallaxes.parallax_mas`
- `parallax_error`, `e_plx`, `plx_err`, `sigma_plx`, `eplx`, `e_Plx` → `Parallaxes.parallax_error`

**Proper motion:**
- `pm_ra`, `pmra`, `mu_ra`, `mu_alpha`, `pmRA` → `ProperMotions.pm_ra`
- `pm_dec`, `pmdec`, `mu_dec`, `mu_delta`, `pmDE` → `ProperMotions.pm_dec`
- Corresponding `_error`, `_err`, `e_` prefix variants → the matching `_error` field

**Radial velocity:**
- `rv`, `RV`, `radial_velocity`, `vrad`, `HRV`, `cz` → `RadialVelocities.rv_kms`
- Corresponding error variants → `RadialVelocities.rv_error`

**Photometry — single-band columns:**
Column names that *are* a band name (case-insensitive), or end/start with a band name:
- Standard band names: `u`, `g`, `r`, `i`, `z`, `y`, `J`, `H`, `K`, `Ks`, `W1`, `W2`, `W3`, `W4`, `G`, `BP`, `RP`, `NUV`, `FUV`, `B`, `V`, `R`, `I`, `L`, `M`, `N`, `Q`
- e.g. `Jmag`, `J_mag`, `Hmag`, `W1mag`, `G_mag`, `gmag`, `rmag` → `Photometry.magnitude`
- Corresponding errors (`eJmag`, `J_err`, `e_Jmag`, `Jmag_err`, `eW1`, etc.) → `Photometry.magnitude_error`
- If there are asymmetric upper/lower error columns, map them to `magnitude_error_upper` / `magnitude_error_lower`

**Rotational parameters:**
- `period`, `rot_period`, `Prot`, `P_rot`, `rotation_period` → `RotationalParameters.period_hr` (note: units matter — convert to hours if in days)
- `vsini`, `v_sin_i`, `vrot`, `vsini_kms` → `RotationalParameters.v_sin_i_kms`
- `inclination`, `incl`, `i_rot` → `RotationalParameters.inclination`

**Morphology:**
- `pa`, `position_angle`, `PA_deg` → `Morphology.position_angle_deg`
- `ellipticity`, `ellip`, `e=1-b/a`, `epsilon` → `Morphology.ellipticity`
- `half_light_radius`, `r_eff`, `r_h`, `Re` → `Morphology.half_light_radius_arcmin` (check units)

**Spectra:**
- `access_url`, `spectrum_url`, `url`, `fits_url` → `Spectra.access_url`
- `telescope`, `obs_telescope` → `Spectra.telescope` (or `Telescopes.telescope`)
- `instrument` → `Spectra.instrument`
- `obs_date`, `date_obs`, `observation_date` → `Spectra.observation_date`

**Classification:**
- `spectral_type`, `spt`, `SpT`, `sp_type`, `sp_type_adopted` → `SourceTypes.source_type`
- `association`, `moving_group`, `young_association`, `cluster` → `Associations.association`

**References:**
- `reference`, `ref`, `bibcode`, `citation` → `Publications.reference`

### Layer 2: Units (use when name is ambiguous)

| Units | Likely field |
|---|---|
| `mas` | `Parallaxes.parallax_mas` |
| `mas/yr` | `ProperMotions.pm_ra` or `pm_dec` |
| `km/s` | `RadialVelocities.rv_kms` or `RotationalParameters.v_sin_i_kms` |
| `mag` | `Photometry.magnitude` |
| `deg` + RA/Dec context | `Sources.ra_deg` / `Sources.dec_deg` |
| `hr` or `d` + periodic context | `RotationalParameters.period_hr` |
| `arcsec` + separation context | `CompanionRelationships.projected_separation_arcsec` |
| `arcmin` + size context | `Morphology.half_light_radius_arcmin` |

### Layer 3: Description text (use as tiebreaker or when name+units both unclear)

Scan description for key words:
- "magnitude", "flux", "photometry" → `Photometry`
- "parallax", "distance" → `Parallaxes`
- "radial velocity", "line-of-sight velocity" → `RadialVelocities`
- "proper motion", "sky-plane motion" → `ProperMotions`
- "spectral type", "classification" → `SourceTypes`
- "rotation", "spin", "period" → `RotationalParameters`
- "separation", "companion", "binary" → `CompanionRelationships` or `CompanionParameters`
- "association", "membership", "moving group" → `Associations`

### Uncertainty columns

When you identify that a column is an uncertainty on a value you've already mapped:
- Single symmetric error → `<field>_error`
- Upper bound / `+` suffix / `_upper` / `_plus` → `<field>_error_upper`
- Lower bound / `-` suffix / `_lower` / `_minus` → `<field>_error_lower`

### Catch-all tables for unmapped physical parameters

If a column clearly represents a physical quantity but doesn't fit any specific table field:
- Fitted or modeled parameters (effective temperature `Teff`, luminosity `L`, mass `M`, radius `R`, surface gravity `logg`, metallicity `[Fe/H]`, age) → `ModeledParameters` (using the generic `value` + `unit` fields; put the parameter name in `parameter`)
- Companion-derived measurements → `CompanionParameters`

## Output

Produce a markdown table with one row per input column:

```
| Input Column | Description | Units | DB Table | DB Field | Confidence | Notes |
|---|---|---|---|---|---|---|
```

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
