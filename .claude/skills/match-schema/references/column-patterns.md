# Column Name Matching Patterns

## Layer 1: Column name patterns (strongest signal)

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

**Photometry band → SVO filter ID:**
The `band` value stored in `PhotometryFilters` must be an SVO Filter Profile Service ID
(`Facility/Instrument.Band`). Map common band names as follows:

| Bare band name | SVO filter ID |
|---|---|
| J (2MASS) | `2MASS/2MASS.J` |
| H (2MASS) | `2MASS/2MASS.H` |
| K / Ks (2MASS) | `2MASS/2MASS.Ks` |
| W1 | `WISE/WISE.W1` |
| W2 | `WISE/WISE.W2` |
| W3 | `WISE/WISE.W3` |
| W4 | `WISE/WISE.W4` |
| u (SDSS) | `SDSS/SDSS.u` |
| g (SDSS) | `SDSS/SDSS.g` |
| r (SDSS) | `SDSS/SDSS.r` |
| i (SDSS) | `SDSS/SDSS.i` |
| z (SDSS) | `SDSS/SDSS.z` |
| G (Gaia DR3) | `Gaia/Gaia3.G` |
| BP (Gaia DR3) | `Gaia/Gaia3.Gbp` |
| RP (Gaia DR3) | `Gaia/Gaia3.Grp` |
| B (Johnson) | `Generic/Johnson.B` |
| V (Johnson) | `Generic/Johnson.V` |
| R (Cousins) | `Generic/Cousins.R` |
| I (Cousins) | `Generic/Cousins.I` |

For any band not in this table, search https://svo2.cab.inta-csic.es/theory/fps3/ to find the
correct ID. If the telescope or instrument is ambiguous, flag the band in the output and ask the
user to confirm the SVO ID.

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

## Layer 2: Units (use when name is ambiguous)

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

## Layer 3: Description text (use as tiebreaker or when name+units both unclear)

Scan description for key words:
- "magnitude", "flux", "photometry" → `Photometry`
- "parallax", "distance" → `Parallaxes`
- "radial velocity", "line-of-sight velocity" → `RadialVelocities`
- "proper motion", "sky-plane motion" → `ProperMotions`
- "spectral type", "classification" → `SourceTypes`
- "rotation", "spin", "period" → `RotationalParameters`
- "separation", "companion", "binary" → `CompanionRelationships` or `CompanionParameters`
- "association", "membership", "moving group" → `Associations`

## Uncertainty columns

When you identify that a column is an uncertainty on a value you've already mapped:
- Single symmetric error → `<field>_error`
- Upper bound / `+` suffix / `_upper` / `_plus` → `<field>_error_upper`
- Lower bound / `-` suffix / `_lower` / `_minus` → `<field>_error_lower`

## Catch-all tables for unmapped physical parameters

If a column clearly represents a physical quantity but doesn't fit any specific table field:
- Fitted or modeled parameters (effective temperature `Teff`, luminosity `L`, mass `M`, radius `R`, surface gravity `logg`, metallicity `[Fe/H]`, age) → `ModeledParameters` (using the generic `value` + `unit` fields; put the parameter name in `parameter`)
- Companion-derived measurements → `CompanionParameters`
