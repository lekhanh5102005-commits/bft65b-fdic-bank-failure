# BFT65B Final Report – A Big-Data Pipeline for Bank Failure Early Warning

Individual final project for **BFT65B – Big Data Analytics in Finance** (National Economics University).

- **Author:** LE KHANH (student ID 11230974)
- **Report:** submitted separately (PDF/DOCX)

The pipeline downloads 42.5 years of FDIC Call Reports, stores them as Parquet, audits data quality, builds a point-in-time bank–quarter panel with Polars (cross-checked with pandas and PySpark), and validates an early-warning model for bank failure within four quarters on a single, frozen test period.

## Repository contents

| File | Purpose |
| --- | --- |
| `Big_Data_Analysis_final.ipynb` | Full pipeline with all outputs (Parts 1–5 and the final test evaluation) |
| `fdic_download_starter.py` | Download script provided by the course instructor, used unmodified |
| `requirements.txt` | Pinned Python package versions |
| `README.md` | This file |

**No data files are committed.** All data are downloaded from the FDIC API by the script (see below).

## Data

- Source: FDIC BankFind Suite public API, `https://api.fdic.gov/banks` (no key needed; docs: https://api.fdic.gov/banks/docs/)
- Scope: Financials 1984Q1–2026Q2 (170 quarters, 1,682,738 bank-quarters, 24,064 banks); complete Failures (4,117 rows) and Institutions (27,835 rows) tables
- Download dates: Financials 2026-11-19; Failures and Institutions re-downloaded 2026-11-20

## How to reproduce

The notebook runs on Google Colab with Google Drive.

1. Open `Big_Data_Analysis_final.ipynb` in Google Colab.
2. In the second code cell, set `WORK_DIR` to a folder in your Google Drive (default: `/content/drive/MyDrive/bft65b_fdic_project`).
3. Upload `fdic_download_starter.py` to `WORK_DIR`.
4. Run all cells from top to bottom. The download cell runs:
   ```
   python fdic_download_starter.py --start 1984-03-31 --end 2026-06-30 --out <WORK_DIR>/data_raw/fdic
   ```
   A first download takes about one hour; reruns skip quarters already marked complete in `manifest_financials.json`.
5. Seeds are fixed (`SEED = 42` for scikit-learn models and the bootstrap).

To install the same package versions outside Colab: `pip install -r requirements.txt`.

### Note on the test set

The cell under **"Final test evaluation"** writes `TEST_USED.lock` after scoring the test period once, and refuses to run while that file exists. To reproduce the test evaluation from scratch, run the notebook on a clean copy of the working folder (without the lock file).

## Design summary

| Step | Choice |
| --- | --- |
| Storage | Unified-schema Parquet (Zstd), plus a Hive-partitioned copy by year (PyArrow) |
| Processing | Polars lazy query; benchmarked against pandas at 1×, 2×, 4× data; key steps reproduced in PySpark |
| Label | Bank in the FDIC failure list (RESTYPE = FAILURE) with failure date in quarters q+1 … q+4; mergers and assistance are not failures |
| Split | Development 1984Q1–2012Q4 (2,779 failed banks), gap 2013, test 2014Q1–2025Q2 (53 failed banks) |
| Validation | Walk-forward with expanding window and a 4-quarter purge; ROC-AUC and PR-AUC |
| Models | Logistic regression (baseline), HistGradientBoosting, Spark MLlib logistic regression (demo) |
| Threshold | Minimises expected cost on development out-of-fold predictions (C_FN from FDIC loss data, C_FP:C_FN = 1:100 with sensitivity analysis) |

Single test evaluation (2014Q1–2025Q2): logistic regression ROC-AUC 0.952, PR-AUC 0.353, recall 0.789 at a 0.91% alert rate.

## References

- FDIC. BankFind Suite API documentation and field dictionary. https://api.fdic.gov/banks/docs/
- Martin, D. (1977). Early warning of bank failure: A logit regression approach. *Journal of Banking & Finance*, 1(3), 249–276.
- Cole, R. A., & White, L. J. (2012). Déjà vu all over again: The causes of U.S. commercial bank failures this time around. *Journal of Financial Services Research*, 42(1–2), 5–29.
- Pedregosa, F., et al. (2011). Scikit-learn: Machine learning in Python. *Journal of Machine Learning Research*, 12, 2825–2830.
- Saito, T., & Rehmsmeier, M. (2015). The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE*, 10(3), e0118432.
- Zaharia, M., et al. (2016). Apache Spark: A unified engine for big data processing. *Communications of the ACM*, 59(11), 56–65.
- Polars documentation: https://docs.pola.rs · Apache Arrow (PyArrow) documentation: https://arrow.apache.org/docs/python/
- BFT65B course starter script `fdic_download_starter.py` (provided by the instructor; used unmodified).

## Use of AI tools

Claude (Anthropic, Claude Sonnet 5.5) and Gemini (Google) were used to assist with writing and debugging the notebook code. Claude was also used to review the code and to suggest a detailed report outline based on the notebook's outputs, and to point out the limitations of the analysis. All numbers were taken from the notebook outputs, and all design decisions, code runs and the final text were checked and revised by the author. The same statement appears in the report.
