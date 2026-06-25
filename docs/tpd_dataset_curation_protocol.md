# TPD Dataset Curation and Standardization Protocol

## Purpose and scope

This document defines the publication-quality curation rules used to
standardize targeted protein degradation (TPD) activity records for downstream
modeling. The protocol covers target and ligase identifiers, cell-line
metadata, quantitative activity values, non-quantitative activity annotations,
inequality and range handling, final export headers, and the activity threshold
used to define active degraders.

The primary quantitative endpoints are:

- `DC50`: concentration that produces 50% degradation, standardized to
  nanomolar (nM).
- `Dmax`: maximum degradation, standardized as percent degradation.
- `pDC50`: negative base-10 logarithm of `DC50` expressed in molar units,
  used for potency distribution plots and summary statistics.

For reporting potency distributions, `pDC50` is calculated as:

```text
pDC50 = -log10(DC50 in molar concentration)
      = 9 - log10(DC50 in nM)
```

Example: `DC50 = 100 nM` corresponds to `pDC50 = 7.0`.

## Core curation principles

1. Each retained quantitative record must describe one compound, one protein of
   interest (POI), one E3 ligase, one cell line, and one activity endpoint.
2. Entries containing multiple targets within a single row are excluded from
   quantitative `DC50` and `Dmax` training to avoid ambiguous target-label
   assignments.
3. Quantitative concentrations are standardized to nM before modeling.
4. Non-quantitative or assay-status annotations are not imputed as potency
   values, except for the specific `Dmax = NDE` rule described below.
5. Inequality operators and range bounds are preserved in auxiliary audit
   fields so that train/test splitting and evaluation filtering remain
   reproducible.
6. Rows excluded from quantitative activity modeling may still be retained for
   cell-line-only training when they contain valid compound, target, ligase, and
   cell-line metadata.

## Required final modeling headers

The final long-format modeling table must contain the following required
columns, in this order:

```text
serial_id
TPD_ID
SMILES
InChIKey
POI Uniprot ID
Ligase Uniprot ID
Parameter
Cell Line
Activity
```

### Column definitions

| Column | Definition |
| --- | --- |
| `serial_id` | Unique row-level identifier assigned after curation. |
| `TPD_ID` | Source TPD record identifier for traceability. |
| `SMILES` | Canonical or source-provided molecular SMILES string. |
| `InChIKey` | Molecular InChIKey. |
| `POI Uniprot ID` | Curated UniProt identifier for the protein of interest. |
| `Ligase Uniprot ID` | Curated UniProt identifier for the E3 ligase. |
| `Parameter` | Standardized activity endpoint. Allowed quantitative values are `DC50_nM` and `Dmax_percent`; `pDC50` may be exported for analysis/visualization when explicitly required. |
| `Cell Line` | Experimental cell line. |
| `Activity` | Numeric standardized activity value corresponding to `Parameter`. |

### Standard parameter vocabulary

Use the following controlled values in the `Parameter` column:

| `Parameter` value | Standardized unit | Use |
| --- | --- | --- |
| `DC50_nM` | nM | Quantitative potency modeling and active-label assignment. |
| `Dmax_percent` | percent degradation | Quantitative degradation-efficacy modeling and active-label assignment. |
| `pDC50` | unitless logarithmic value | Derived reporting and visualization endpoint; include in final exports only when explicitly required for potency distribution analysis. |

### Auxiliary audit fields

The required final modeling headers define the minimal table used for model
input. During curation, the following auxiliary fields should be retained in a
separate audit table or appended after the required columns when a single flat
file is needed:

| Auxiliary field | Definition |
| --- | --- |
| `activity_raw` | Original unmodified activity entry from the source. |
| `activity_operator` | Inequality operator parsed from the source value: `<`, `>`, `<=`, `>=`, or blank if no operator was present. |
| `activity_range_lower` | Lower numeric bound for range values, after unit conversion. |
| `activity_range_upper` | Upper numeric bound for range values, after unit conversion. |
| `activity_unit_raw` | Original source unit. |
| `activity_unit_standardized` | Standardized unit, either `nM` for `DC50_nM` or `%` for `Dmax_percent`. |
| `quantitative_use` | Whether the row is eligible for quantitative model training. |
| `evaluation_use` | Whether the row is eligible for validation/test evaluation. Rows with inequality operators should be excluded from evaluation sets. |
| `cell_line_training_use` | Whether the row is retained for cell-line-only training despite exclusion from quantitative endpoint modeling. |
| `exclusion_reason` | Controlled reason for exclusion, if applicable. |

## Identifier and target curation

### POI and E3 ligase requirements

Rows must be excluded from the final quantitative dataset when:

- `POI Uniprot ID` is missing,
- `Ligase Uniprot ID` is missing, or
- both identifiers are missing.

These exclusions apply before quantitative activity modeling because the model
label cannot be assigned to a unique POI-ligase pair without both identifiers.

### Multiple target handling

Entries containing multiple targets in a single source row are excluded from
quantitative `DC50` and `Dmax` training for consistency. Such rows may be
retained only for non-quantitative metadata tasks if a downstream task can
handle multi-target annotations explicitly.

Source-specific TPD exception: if a TPD record contains exactly two
`POI_Uniprot_ID` values separated by a slash, retain the second identifier as
the curated `POI Uniprot ID` and discard the first. For example:

```text
Raw POI_Uniprot_ID: P12345/Q67890
Curated POI Uniprot ID: Q67890
```

If a slash-delimited entry contains more than two identifiers or cannot be
resolved unambiguously, treat it as a multi-target row and exclude it from
quantitative activity modeling.

## Quantitative value standardization

### DC50 concentration conversion

All `DC50` values are converted to nM before modeling. When source values are
reported as `pDC50`, convert them to `DC50_nM` using:

```text
DC50_nM = 10^(9 - pDC50)
```

Examples:

| Source value | Standardized value |
| --- | --- |
| `0.1 uM` | `100 nM` |
| `1 uM` | `1000 nM` |
| `100 nM` | `100 nM` |
| `pDC50 = 7` | `100 nM` |
| `pDC50 = 8` | `10 nM` |

### Dmax standardization

`Dmax` values are standardized as percent degradation. Numeric percentage
values are retained as reported after removing percent symbols and whitespace.

Examples:

| Source value | Standardized value |
| --- | --- |
| `85%` | `85` |
| `80` | `80` |
| `0%` | `0` |

## Inequality operators

Entries with inequality operators are retained as numeric values for training
where appropriate, but the operator is stored separately. This allows the
numeric component to contribute to training while preventing censored values
from contaminating evaluation sets.

Allowed parsed operators are:

```text
<
>
<=
>=
```

Examples:

| Source value | `Activity` | `activity_operator` | Evaluation use |
| --- | ---: | --- | --- |
| `>100 nM` | `100` | `>` | Exclude from validation/test evaluation |
| `<10 nM` | `10` | `<` | Exclude from validation/test evaluation |
| `>=80%` | `80` | `>=` | Exclude from validation/test evaluation |
| `<=20%` | `20` | `<=` | Exclude from validation/test evaluation |

For final model evaluation, all rows with non-empty `activity_operator` should
be excluded from validation and test sets. They may remain in training sets as
censored observations when the modeling workflow explicitly permits them.

## Range values

Range values are converted to their arithmetic mean for the standardized
`Activity` value, while the original lower and upper bounds are retained in
auxiliary audit fields.

Examples:

| Source value | `Activity` | `activity_range_lower` | `activity_range_upper` |
| --- | ---: | ---: | ---: |
| `10-100 nM` | `55` | `10` | `100` |
| `0.01-0.1 uM` | `55` | `10` | `100` |
| `40-80%` | `60` | `40` | `80` |

Range-derived values should be flagged in the audit table. If strict evaluation
requires exact point estimates only, range-derived records should be excluded
from validation/test evaluation.

## Non-quantitative and assay-status annotations

The following controlled rules must be applied to source activity annotations.

### DC50-specific rules

| Source annotation | Quantitative `DC50` action | Cell-line-only action |
| --- | --- | --- |
| `NDE` or `No determinable effect` | Exclude from `DC50` training and evaluation. | Retain for cell-line-only training if metadata are complete. |
| `ND` | Exclude from `DC50` training and evaluation. | Retain for cell-line-only training if metadata are complete. |
| `NT` or `Not tested` | Exclude from `DC50` training and evaluation. | Retain for cell-line-only training if metadata are complete. |
| `-` | Exclude from `DC50` training and evaluation. | Retain for cell-line-only training only if interpreted as an assay-status marker and metadata are complete. |
| `A`, `B`, `C`, `D`, `E` | Exclude from `DC50` training and evaluation because patent-specific mappings are required to infer quantitative values. | Retain for cell-line-only training if metadata are complete. |
| `+`, `++`, `+++`, `-`, `--`, `---`, `----` | Exclude from `DC50` training and evaluation because these are qualitative activity labels. | Retain for cell-line-only training if metadata are complete. |

### Dmax-specific rules

| Source annotation | Quantitative `Dmax` action | Cell-line-only action |
| --- | --- | --- |
| `NDE` or `No determinable effect` | Convert to `Dmax_percent = 0`. | Also eligible for cell-line-only training if metadata are complete. |
| `NT` or `Not tested` | Exclude from `Dmax` training and evaluation. | Retain for cell-line-only training if metadata are complete. |
| `ND` | Exclude from `Dmax` training and evaluation. | Retain for cell-line-only training if metadata are complete. |
| `-` | Exclude from `Dmax` training and evaluation. | Retain for cell-line-only training only if interpreted as an assay-status marker and metadata are complete. |
| `A`, `B`, `C`, `D`, `E` | Exclude from `Dmax` training and evaluation because patent-specific mappings are required to infer quantitative values. | Retain for cell-line-only training if metadata are complete. |
| `+`, `++`, `+++`, `-`, `--`, `---`, `----` | Exclude from `Dmax` training and evaluation because these are qualitative activity labels. | Retain for cell-line-only training if metadata are complete. |

### Patent activity grades

Categorical grades such as `A`, `B`, `C`, `D`, and `E`, which are common in
patent disclosures, are excluded from quantitative `DC50` and `Dmax` modeling.
These labels require patent-specific grade-to-value mappings and should not be
pooled across patents as if they were comparable numeric measurements.

## Active degrader definition

A data point is defined as active only when both quantitative conditions are
met:

```text
Dmax > 80%
DC50 < 100 nM
```

This follows the thresholds used by Li et al. The active label must be assigned
at the compound-POI-ligase-cell-line level after `DC50` and `Dmax` have both
been standardized.

Examples:

| DC50 | Dmax | Active label | Reason |
| ---: | ---: | --- | --- |
| `50 nM` | `90%` | Active | `DC50 < 100 nM` and `Dmax > 80%`. |
| `100 nM` | `90%` | Not active | The threshold is strict; `100 nM` is not `< 100 nM`. |
| `50 nM` | `80%` | Not active | The threshold is strict; `80%` is not `> 80%`. |
| `150 nM` | `90%` | Not active | Potency threshold is not met. |
| `50 nM` | `70%` | Not active | Degradation threshold is not met. |
| `>100 nM` | `90%` | Training-only censored record | Numeric value may be used for training if allowed, but exclude from evaluation because the `DC50` operator is censored. |

When either `DC50` or `Dmax` is missing, non-quantitative, or excluded by the
rules above, no active/inactive classification should be assigned for
quantitative model evaluation.

## Recommended curation workflow

1. **Load source records** and preserve raw activity strings, raw units, source
   identifiers, compound identifiers, target identifiers, ligase identifiers,
   and cell-line annotations.
2. **Normalize target and ligase identifiers**:
   - apply the TPD slash-delimited POI rule where applicable;
   - remove rows missing POI or E3 ligase UniProt identifiers;
   - remove unresolved multi-target rows from quantitative modeling.
3. **Parse activity parameters** into a long-format table with one row per
   endpoint (`DC50_nM` or `Dmax_percent`).
4. **Apply non-quantitative annotation rules**:
   - remove `DC50 = NDE` from quantitative `DC50` modeling;
   - convert `Dmax = NDE` to `0`;
   - remove `NT`, `ND`, standalone negative signs, qualitative plus/minus
     labels, and patent activity grades from quantitative endpoint modeling.
5. **Parse operators and ranges**:
   - store `<`, `>`, `<=`, or `>=` in `activity_operator`;
   - store range lower and upper bounds in standardized units;
   - set `Activity` to the numeric component for inequalities and to the
     arithmetic mean for ranges.
6. **Standardize units**:
   - convert all `DC50` concentrations to nM;
   - convert `pDC50` to `DC50_nM`;
   - standardize `Dmax` as percent degradation.
7. **Assign usage flags**:
   - `quantitative_use = true` only for records eligible for `DC50` or `Dmax`
     training;
   - `evaluation_use = false` for inequality records and, when strict point
     estimates are required, range-derived records;
   - `cell_line_training_use = true` for excluded activity rows that still have
     valid cell-line and identifier metadata.
8. **Generate final exports**:
   - export the required nine-column modeling table;
   - export or retain the auxiliary audit table for reproducibility;
   - compute `pDC50` from standardized `DC50_nM` for potency distribution
     plots.

## Consistent exclusion-reason vocabulary

Use the following controlled values in `exclusion_reason` to keep filtering
reproducible:

| Value | Meaning |
| --- | --- |
| `missing_poi_uniprot` | POI UniProt identifier is missing. |
| `missing_ligase_uniprot` | E3 ligase UniProt identifier is missing. |
| `missing_poi_and_ligase_uniprot` | Both POI and E3 ligase UniProt identifiers are missing. |
| `multiple_targets` | Row contains unresolved multiple targets. |
| `dc50_nde` | `DC50` is `NDE` or `No determinable effect`. |
| `dc50_nt` | `DC50` is `NT` or `Not tested`. |
| `dmax_nt` | `Dmax` is `NT` or `Not tested`. |
| `activity_nd` | `DC50` or `Dmax` is `ND`. |
| `negative_sign_only` | Activity value is a standalone `-` sign. |
| `qualitative_plus_minus_label` | Activity value is a qualitative plus/minus label such as `++` or `---`. |
| `patent_grade_label` | Activity value is a categorical grade such as `A`, `B`, `C`, `D`, or `E`. |
| `non_numeric_unmapped` | Activity value cannot be parsed into a numeric value under the rules above. |

## Case examples

### Example 1: inequality DC50

Raw record:

```text
Parameter = DC50
Activity = >100 nM
```

Curated record:

```text
Parameter = DC50_nM
Activity = 100
activity_operator = >
evaluation_use = false
```

### Example 2: pDC50 conversion

Raw record:

```text
Parameter = pDC50
Activity = 7.5
```

Curated record:

```text
Parameter = DC50_nM
Activity = 31.62
pDC50 = 7.5
```

### Example 3: range DC50

Raw record:

```text
Parameter = DC50
Activity = 10-100 nM
```

Curated record:

```text
Parameter = DC50_nM
Activity = 55
activity_range_lower = 10
activity_range_upper = 100
```

### Example 4: Dmax with no determinable effect

Raw record:

```text
Parameter = Dmax
Activity = NDE
```

Curated record:

```text
Parameter = Dmax_percent
Activity = 0
quantitative_use = true
```

### Example 5: qualitative patent grade

Raw record:

```text
Parameter = DC50
Activity = B
```

Curated outcome:

```text
quantitative_use = false
cell_line_training_use = true, if compound, POI, ligase, and cell-line metadata are complete
exclusion_reason = patent_grade_label
```

### Example 6: missing target identifier

Raw record:

```text
POI Uniprot ID = missing
Ligase Uniprot ID = Q13616
```

Curated outcome:

```text
Exclude from final quantitative dataset
exclusion_reason = missing_poi_uniprot
```

### Example 7: active-label assignment

Raw paired activity records for the same compound, POI, ligase, and cell line:

```text
DC50 = 50 nM
Dmax = 90%
```

Curated outcome:

```text
active = true
```

The row is active because `DC50 < 100 nM` and `Dmax > 80%`.

## Reporting statement

For publication methods, the following concise statement can be used:

> TPD activity records were curated into a standardized long-format table with
> one compound, POI, E3 ligase, cell line, and activity endpoint per row. Rows
> with missing POI or E3 ligase UniProt identifiers, unresolved multiple
> targets, non-quantitative patent grades, qualitative plus/minus labels, `ND`,
> `NT`, or standalone negative signs were excluded from quantitative `DC50` and
> `Dmax` modeling. `DC50` values were converted to nM, `pDC50` values were
> back-converted to `DC50`, and `Dmax` values were standardized as percent
> degradation. Inequality operators were parsed into separate fields while the
> numeric component was retained, enabling censored observations to be excluded
> from validation and test evaluation. Range values were replaced by their
> arithmetic mean and their original bounds were retained. `Dmax = NDE` was
> encoded as 0% degradation, whereas `DC50 = NDE` was excluded from quantitative
> `DC50` modeling and retained only for cell-line-only training when metadata
> were complete. Potency distributions were reported as `pDC50 = -log10(DC50
> [M])`. Active degraders were defined as records with `Dmax > 80%` and `DC50 <
> 100 nM`, following Li et al.
