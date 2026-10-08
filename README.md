# exo_enzyme_assay_repository
Repository for the collection of analysis scripts for both hydrolase and oxidase enzyme assay data from 96-well plates.

Two scripts turn raw plate reader files into enzyme activities per sample:
- **Hydrolases** (fluorometric, MUF substrates): `scripts/script_hydrolases_preprocessing.Rmd`
- **Oxidases** (photometric, TMB substrate; phenol oxidase and peroxidase): `scripts/script_oxidases_preprocessing.Rmd`

Both run on any number of plates and any experimental design: samples are described by an `id` and generic factor columns (`factor_1`, `factor_2`, ...). The repository ships with example plates for both assays.

## Repository structure
```
data/
├── hydrolases/
│   ├── raw/                 Plate reader exports, one .csv file per plate
│   ├── metadata/            df_hydrolases_metadata.xlsx: one row per sample on a plate
│   └── processed/           df_hydrolases_processed.csv (created by the script)
├── oxidases/
│   ├── raw/                 Plate reader exports, one .xlsx file per plate
│   ├── metadata/            df_oxidases_metadata.xlsx: one row per sample on a plate
│   └── processed/           df_oxidases_processed.csv (created by the script)
└── om/                      df_om.xlsx: crucible weights for dry matter and organic matter (hydrolases)
scripts/
├── script_hydrolases_preprocessing.Rmd
└── script_oxidases_preprocessing.Rmd
```

## Workflow
1. Copy the plate reader exports into `data/hydrolases/raw/` or `data/oxidases/raw/`. The file name without extension is the plate name (e.g. `plate_hydrolase_001`).
2. Describe each sample in the metadata file of the assay. For hydrolases, also enter the crucible weights in `data/om/df_om.xlsx`.
3. Open `exo_enzyme_assay_repository.Rproj` in RStudio and run the script of the assay.
4. The result is saved to the `processed/` folder of the assay.

Paths and assay constants are set in Step 1c of each script. Step 4 shows a set of checks; each should return an empty table. Both scripts add a `qc_flag` to every sample; samples are flagged, never removed.

---

## Hydrolases

### Plate layout
Raw files are comma-separated (`.csv`) with a header line (`1` ... `12`) followed by eight rows of fluorescence values. Each plate row is one technical replicate; the four replicates of a sample are averaged.

| Rows | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1-4 | B | S | SC | HC 1 | Q 1 | A 1 | HC 2 | Q 2 | A 2 | HC 3 | Q 3 | A 3 |
| 5-8 | B | S | SC | HC 4 | Q 4 | A 4 | HC 5 | Q 5 | A 5 | HC 6 | Q 6 | A 6 |

B = blank, S = standard, SC = substrate control, HC = homogenate control, Q = quench, A = assay; numbers are the sample `position` (1-6). The reference wells (B, S, SC) are shared by the three samples in the same rows. A different layout is set in Step 3a of the script.

### Metadata: `df_hydrolases_metadata.xlsx`
One row per sample on a plate. The same `id` can appear on several plates, e.g. when a sample is measured with different substrates.

| Column | Description |
|---|---|
| `id` | Sample name, links to the OM data |
| `factor_1`, `factor_2`, ... | Experimental factors, any number |
| `sampling_date` | Date of sampling |
| `plate` | Plate name, equal to the raw file name without `.csv` |
| `position` | Sample position on the plate (1-6) |
| `enzyme` | Enzyme or substrate measured on the plate |
| `incubation_start` | Start of incubation (date and time) |
| `incubation_end` | End of incubation (date and time) |
| `sample_g` | Fresh sample mass used for the homogenate [g] |
| `buffer_ml` | Buffer volume of the homogenate [mL] |

Further columns are passed through to the output unchanged.

### OM data: `df_om.xlsx`
One row per `id`. All weights are gross weights including the crucible [g].

| Column | Description |
|---|---|
| `id` | Sample name, as in the metadata |
| `crucible_id` | Crucible label |
| `crucible_weight` | Empty crucible |
| `crucible_fw` | Crucible with fresh sample |
| `crucible_dw_105` | Crucible after drying at 105 °C |
| `crucible_loi_550` | Crucible after ignition at 550 °C |
| `comment` | Free text |

### Calculations
Per sample, from the mean of the four technical replicates (German et al. 2011):

```
emission_coef    = standard / standard_nmol
quench_coef      = (quench - homogenate_control) / standard
net_fluorescence = (assay - homogenate_control) / quench_coef - substrate_control
activity_fw      = net_fluorescence * buffer_ml / (emission_coef * homogenate_ml * incubation_h * sample_g)

dw_fraction      = (crucible_dw_105 - crucible_weight) / (crucible_fw - crucible_weight)
om_fraction      = (crucible_dw_105 - crucible_loi_550) / (crucible_fw - crucible_weight)
activity_dw      = activity_fw / dw_fraction
activity_om      = activity_fw / om_fraction
```

Activities are in nmol per g fresh sample (`fw`), dry sample (`dw`) or organic matter (`om`) per hour. `standard_nmol` (5 nmol) and `homogenate_ml` (0.2 mL) are set in Step 1c. Dry matter and OM fractions outside 0-1 are shown by the script and set to `NA`.

| `qc_flag` | Meaning |
|---|---|
| `ok` | Valid activity |
| `missing assay data` | Activity cannot be calculated, e.g. missing `sample_g` or `buffer_ml` |
| `activity <= 0` | Net fluorescence not above the controls |
| `activity above limit` | `activity_om` above `max_activity_om` (default 10000) |

### Example data
Ten plates of a β-glucosidase (`GLU`) assay with 60 soil samples. The `id` is `plot_subplot_depth` (e.g. `121_C2_0-5`) and the factors are:

| Column | Meaning | Values |
|---|---|---|
| `factor_1` | Plot | 120, 121, 370, ... |
| `factor_2` | Subplot | A2, A6, C2, G5 |
| `factor_3` | Depth [cm] | 0-5, 5-10, 20-30, 40-50, 60-80, 80-100 |
| `factor_4` | Zone | PZ (pioneer zone), HM (high marsh) |
| `factor_5` | Warming treatment | ambient, +1.5°C, +3.0°C |

Known gaps in this dataset:
- `plate_hydrolase_004` position 6 and all of `plate_hydrolase_005` have no `sample_g` and `buffer_ml`, so they are flagged `missing assay data`.
- `370_A6_40-50` has `crucible_loi_550` = 19725, probably a typo for 19.725, so its OM fraction is set to `NA`.
- `390_G5_20-30` has no OM sample, so only `activity_fw` is available.

---

## Oxidases

### Plate layout
Raw files are Excel exports with one 8 x 12 block per read. The last column names the read (`Read 1:630`, `Read 2:630`, ...). Read 1 is the start (t0); the following reads are `read_interval_min` apart (default 15 min).

Each plate row is one sample (`position` 1-8 = row A-H). Each treatment has three technical replicates side by side:

| Columns | 1-3 | 4-6 | 7-9 | 10-12 |
|---|---|---|---|---|
| Wells | Assay | Homogenate control | Assay + H₂O₂ | Homogenate control + H₂O₂ |
| Content | Suspension + buffer + TMB | Suspension + buffer | Suspension + buffer + TMB + H₂O₂ | Suspension + buffer + H₂O₂ |

A negative control (buffer instead of suspension) can take any row; give it the `id` `negative_control` in the metadata.

### Metadata: `df_oxidases_metadata.xlsx`

| Column | Description |
|---|---|
| `id` | Sample name; `negative_control` for negative controls |
| `factor_1`, `factor_2`, ... | Experimental factors, any number |
| `sampling_date` | Date of sampling |
| `plate` | Plate name, equal to the raw file name without `.xlsx` |
| `position` | Plate row of the sample (1-8 = A-H) |
| `sample_g` | Sample mass in the suspension [g]; protocol: fresh soil equivalent to 0.5 g dry weight, entered as 0.5 |
| `buffer_ml` | Volume of the suspension [mL] |

Further columns are passed through to the output unchanged.

### Calculations
Per sample and read, from the mean of the three technical replicates (protocol section 3.7.2):

```
phenol_oxidase = assay - homogenate_control
total_oxidase  = assay_h2o2 - homogenate_control_h2o2
peroxidase     = total_oxidase - phenol_oxidase
```

For each enzyme, a linear regression of this net absorbance on time (t0 to `max_time_min`) gives `slope_per_min` and `r_squared`. Then:

```
sample_per_well_g = sample_g * homogenate_ml / buffer_ml
activity          = slope_per_min * 60 / sample_per_well_g
```

`activity` is in E630 g⁻¹ h⁻¹ (change of absorbance at 630 nm per g sample and hour); with `sample_g` as dry weight equivalent, as in the protocol, this is per g dry soil. `homogenate_ml` (0.1 mL), `read_interval_min` (15 min), `max_time_min` (60 min) and `min_r_squared` (0.98) are set in Step 1c.

| `qc_flag` | Meaning |
|---|---|
| `ok` | Valid activity |
| `negative control` | Negative control; its slope should be close to zero |
| `missing assay data` | Activity cannot be calculated, e.g. missing `sample_g` or `buffer_ml` |
| `activity <= 0` | Absorbance does not increase |
| `r2 below limit` | Absorbance does not increase linearly; consider lowering `max_time_min` |

### Example data
One plate (`plate_oxidase_001`) with five reads (0, 15, 30, 45, 60 min). Rows A-G are soil samples (row G is another soil type with low activity), row H is the negative control. The metadata are placeholders: ids `sample_1` ... `sample_7` and the protocol amounts (0.5 g soil in 50 mL). Replace them with the real sample names and masses.
- Phenol oxidase increases linearly in all samples (r² ≥ 0.986).
- In the wells with H₂O₂ the absorbance levels off at about 2.5-3 within the hour, so total oxidase and peroxidase of samples 1-6 are flagged `r2 below limit`. Shorter time windows do not solve this.
- The negative control shows no increase.
