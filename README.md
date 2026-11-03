# HCP Segmentation & Call Targeting for a Pharma Sales Force

> End-to-end analytics project: segment prescribers (HCPs), score their potential, and reallocate sales rep calls to maximize brand impact at constant effort.
> Built on **Databricks (PySpark + Delta Lake)**, **Python**, **SQL**, **Excel** and **Tableau**.

---

## 1. Business Problem

A pharma company sells **Brand X**, a branded Type 2 diabetes therapy. The field force makes roughly the same number of calls to every doctor, regardless of how much each doctor prescribes or could prescribe.

- High-potential prescribers are under-called.
- Low-potential prescribers are over-called.
- About 7% of HCPs have never been called.

**Question:** Which HCPs should reps prioritize, and how should the same total call capacity be redistributed to grow Brand X prescriptions?

## 2. Solution Summary

| Step | What was done | Tool |
|---|---|---|
| 1 | Cleaned and validated raw HCP data; logged every fix | Databricks, PySpark, SQL |
| 2 | EDA with Lorenz curve, Gini coefficient and decile analysis | Python (pandas, matplotlib) |
| 3 | Engineered features: brand share, growth, calls per Rx, headroom | PySpark, SQL |
| 4 | Segmented HCPs with K-Means and GMM; validated with silhouette and BIC | Python (scikit-learn) |
| 5 | Built a potential vs. current-effort quadrant and a call reallocation plan | Python, Excel |
| 6 | Delivered an interactive dashboard and target list | Tableau |

## 3. Key Results

> Replace the placeholders with your actual numbers after running the notebooks.

- Top 20% of HCPs account for **[X]%** of Brand X Rx (Gini = **[x.xx]**).
- Call volume vs. Rx volume correlation is **[r]**, confirming effort is not aligned with potential.
- Final model: **[K-Means / GMM]** with **k = [k]** segments (silhouette = **[x.xx]**).
- Reallocating **[N]** calls from low-potential to high-potential HCPs at constant total calls projects **[X]%** incremental Brand X Rx (see assumptions in `excel/`).

## 4. Data

The data is **fully synthetic**, modeled on the structure of the CMS Medicare Part D Prescriber public dataset. No real patient or prescriber information is used.

| File | Rows | Description |
|---|---|---|
| `hcp_master.csv` | 6,025 | One row per HCP: profile, patients, prescribing, access, engagement |
| `hcp_quarterly_rx.csv` | 48,000 | 8 quarters of Brand X Rx and rep calls per HCP |
| `generate_hcp_dataset.py` | n/a | Script that regenerates both files (seed 42) |

**Column groups in `hcp_master.csv`:**

- **Profile:** `hcp_id`, `npi`, `gender`, `specialty`, `practice_type`, `years_in_practice`, `city`, `state`, `region`, `urban_flag`
- **Patients:** `total_medicare_patients`, `dm_patients`, `avg_patient_age`, `avg_patient_risk_score`, `pct_medicaid_dual`
- **Prescribing (12 months):** `total_dm_rx_12m`, `generic_rx_12m`, `branded_class_rx_12m`, `brand_x_rx_12m`, `competitor_rx_12m`, `new_to_brand_rx_12m`, `brand_x_yoy_growth`
- **Access:** `formulary_access_score`
- **Engagement:** `rep_calls_12m`, `samples_dropped_12m`, `days_since_last_call`, `emails_sent_12m`, `email_open_rate`, `speaker_program_attended`

## 5. Data Cleaning Log

| # | Issue | Rows affected | Fix |
|---|---|---|---|
| 1 | Duplicate HCP records | 25 | Dropped duplicates on `hcp_id` |
| 2 | Inconsistent specialty label ("Internal Med") | 54 | Standardized to "Internal Medicine" |
| 3 | Lowercase state codes | 70 | Converted to uppercase |
| 4 | Missing `years_in_practice` | 69 | Imputed with specialty median |
| 5 | Missing `avg_patient_risk_score` | 256 | Imputed with specialty median |
| 6 | Missing `formulary_access_score` | 185 | Imputed with state median |
| 7 | Missing `email_open_rate` | 315 | Imputed with median; added missing flag |
| 8 | Missing `days_since_last_call` | 400 | Not imputed; these HCPs have 0 calls. Set `never_called = 1` |
| 9 | Extreme outliers in `total_dm_rx_12m` | 6 | Detected with IQR and domain check; winsorized at the 99.5th percentile |

## 6. Architecture (Databricks)

```
CSV upload ──> Bronze (raw Delta)  ──> Silver (cleaned Delta)  ──> Gold (features + segments)
                   hcp_bronze             hcp_silver                 hcp_features, hcp_segments
                                                                          │
                                         Excel call plan  <───────────────┤
                                         Tableau dashboard <──────────────┘
```

- **Bronze:** raw files loaded as-is into Delta tables.
- **Silver:** deduplicated, standardized, imputed (SQL + PySpark).
- **Gold:** engineered features, cluster labels, potential/effort scores and recommended call counts.

## 7. Methodology

### EDA
- Lorenz curve and Gini coefficient of Brand X Rx across HCPs.
- Decile analysis: Rx share and call share by Rx decile.
- Specialty, region and practice-type breakdowns.

### Feature Engineering

| Feature | Definition |
|---|---|
| `brand_share_of_branded` | Brand X Rx / branded class Rx |
| `brand_share_of_total` | Brand X Rx / total diabetes Rx |
| `rx_per_dm_patient` | Total diabetes Rx / diabetes patients |
| `calls_per_rx` | Rep calls / Brand X Rx |
| `headroom_rx` | Branded class Rx minus Brand X Rx (competitor Rx the brand could win) |
| `momentum` | Brand X YoY growth, plus 4-quarter slope from `hcp_quarterly_rx.csv` |
| `engagement_score` | Email open rate, speaker program, sample drops (scaled) |

### Segmentation
- Log-transform skewed features and standardize.
- Choose k using elbow, silhouette score and GMM BIC.
- Profile each segment (volume, share, growth, specialty mix) and name it.

### Targeting Framework
- **Potential score** = weighted combination of current Brand X Rx, headroom, diabetes patient volume and growth.
- **Current effort** = rep calls in the last 12 months.
- Quadrants:

| | Low effort | High effort |
|---|---|---|
| **High potential** | **Invest:** increase calls | **Maintain:** keep current level |
| **Low potential** | **Monitor:** low-cost digital touch | **Reduce:** cut calls, shift to email |

### Call Reallocation
- Constraint: **total calls stay constant**.
- Calls move from Reduce to Invest segments using a per-segment call cap.
- Impact estimated from segment-level response (Rx lift per additional call), with assumptions documented in the Excel model.

## 8. Deliverables

| Deliverable | Location |
|---|---|
| Databricks notebooks | `notebooks/` |
| SQL scripts | `sql/` |
| Excel call plan and scenario model | `excel/hcp_call_plan.xlsx` |
| Tableau dashboard | `tableau/` ([Tableau Public link](#)) |
| 1-page client recommendation | `docs/client_recommendation.pdf` |

**Tableau dashboard views:** segment overview, geography map, potential vs. effort quadrant, target HCP list with filters (region, specialty, segment).

## 9. Repository Structure

```
hcp-segmentation-call-targeting/
├── data/
│   ├── hcp_master.csv
│   ├── hcp_quarterly_rx.csv
│   └── generate_hcp_dataset.py
├── notebooks/
│   ├── 01_data_cleaning.py
│   ├── 02_eda_lorenz_deciles.py
│   ├── 03_feature_engineering.py
│   ├── 04_segmentation.py
│   └── 05_targeting_reallocation.py
├── sql/
│   ├── 01_silver_cleaning.sql
│   ├── 02_features.sql
│   └── 03_segment_summary.sql
├── excel/
│   └── hcp_call_plan.xlsx
├── tableau/
│   └── hcp_dashboard.twbx
├── docs/
│   └── client_recommendation.pdf
├── images/
└── README.md
```

## 10. How to Run

1. Upload `data/hcp_master.csv` and `data/hcp_quarterly_rx.csv` to Databricks (Catalog > Create table, or a Volume).
2. Import the notebooks from `notebooks/` into your workspace.
3. Run them in order (01 to 05). Each notebook reads the previous layer's Delta table.
4. Export the gold tables (`hcp_segments`, `hcp_call_plan`) to CSV for Excel and Tableau.
5. Open `excel/hcp_call_plan.xlsx` for the scenario model and `tableau/hcp_dashboard.twbx` for the dashboard.

To regenerate the data:

```bash
python data/generate_hcp_dataset.py
```

## 11. Tech Stack

- **Databricks:** PySpark, Delta Lake, notebooks
- **Python:** pandas, NumPy, scikit-learn, matplotlib, seaborn
- **SQL:** Spark SQL for cleaning, feature tables and segment summaries
- **Excel:** call plan, scenario analysis, summary tables
- **Tableau:** dashboard and target list

## 12. Assumptions & Limitations

- Data is synthetic. The patterns demonstrate the method, not real-world market results.
- Rx response to additional calls is estimated at the segment level and is correlational, not causal. A real engagement would validate it with a test-and-control pilot.
- The potential score weights are judgment-based and should be calibrated with client input.
- Results are a single-snapshot view; production use would require periodic refresh.

## 13. Author

**Rahul** | [GitHub](https://github.com/rahulpatil0001) | [LinkedIn](https://linkedin.com/in/rahulpatil0001)
