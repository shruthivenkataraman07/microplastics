# FTIR Microplastics: Raw Data Exploration

**Texas A&M University, Department of Geology & Geophysics**

`01_FTIR_Raw_Data_Exploration.ipynb` is a discovery notebook. It inventories our raw FTIR microplastics data, finds the spectra and metadata it contains, checks data quality, and assesses how ready the data is for the planned AWS pipeline (S3 Bronze/Silver/Gold → RDS PostgreSQL → Databricks).

The notebook **never modifies the source files** and **never fabricates data or labels**. Every number in its reports is computed from the files it reads.

## Contents

```
ftir_microplastics_eda/
├── 01_FTIR_Raw_Data_Exploration.ipynb   # the notebook
├── requirements.txt                      # dependencies (core + optional readers)
├── README.md
└── ftir_eda_outputs/                     # created/overwritten on every run
    ├── tables/    # CSV: inventory, parse log, spectra summary, QC issues, metadata catalogue, Bronze manifest …
    ├── figures/   # PNG plots (200 dpi)
    ├── reports/   # EDA_SUMMARY.md, eda_summary.json, preprocessing recommendations, draft SQL schema, lineage spec
    └── logs/      # ftir_eda.log: every warning and error, file by file
```

## Running in Google Colab (recommended)

1. **Add the data folder to your Drive.** Open the shared Drive folder in your browser. Right-click it → **Organize → Add shortcut** → choose **My Drive**.
   *Why:* Colab can only see files in the signed-in account's Drive. A shared-folder link is not a file path, and folders that were merely "shared with me" do not appear under `/content/drive`.
2. **Open the notebook in Colab.** Upload `01_FTIR_Raw_Data_Exploration.ipynb` at <https://colab.research.google.com> (**File → Upload notebook**), or open it from Drive with Colab.
3. **Run Section 1.2** (the Drive-mount cell). Google's sign-in window appears. Approve access for the account that has the shortcut.
   No password or token is stored in the notebook. Never paste credentials into a cell.
4. **Set `DATA_ROOT`** in Section 1.3 to the shortcut's path, for example:
   ```python
   DATA_ROOT = Path("/content/drive/MyDrive/<shortcut folder name>")
   ```
   If the path is wrong, the next cell lists the folders it can see in My Drive, which helps you find the right name.
5. **Runtime → Run all.**

In Colab, outputs are written to `/content/drive/MyDrive/ftir_eda_outputs/`, so they are kept after the session ends. The notebook refuses to write inside the data folder.

## Running locally (VS Code / Jupyter)

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt ipykernel
```

1. Download the Drive folder (right-click → **Download**) and unzip it.
2. Point the notebook at the unzipped folder in one of two ways: edit `DATA_ROOT` in Section 1.3, or set an environment variable before launching:
   ```bash
   export FTIR_DATA_ROOT="/path/to/unzipped/folder"
   export FTIR_OUTPUT_DIR="$PWD/ftir_eda_outputs"   # optional
   ```
3. Open the notebook in VS Code, select the `.venv` kernel, and click **Run All**.

## What runs without data

If `DATA_ROOT` cannot be found, the cells marked **⚠️ REQUIRES DATA** print a skip message instead of failing. In that case, only the setup cells, the generic draft SQL schema, and a summary stating that no data was examined are produced.

## Supported formats

| Format | Handling |
|---|---|
| CSV, TSV, TXT, DAT, PRN, ASC, XY, DPT | The notebook detects delimiter, decimal comma, instrument header lines (read as metadata), and the header row |
| XLSX / XLSM (XLS with `xlrd`) | Read sheet by sheet, with header-row detection |
| JCAMP-DX (`.jdx`, `.dx`) | `jcamp` |
| SPC (`.spc`) | `spc-io` (modern 0x4B/0x4C format) |
| Bruker OPUS (`.0`, `.1`, …) | `brukeropusreader` (AB block); identified by file signature, not by extension |
| Thermo OMNIC (`.spa`, `.spg`) | `spectrochempy`, **off by default** (`ENABLE_SPA_READER = True` to enable). Exporting CSV from OMNIC also works. |
| Images, PDFs, archives | Inventoried and checksummed only |

Binary files are never parsed as text. A file that cannot be read gets an explicit status in `tables/parse_log.csv` (`parse_error`, `reader_unavailable`, `unexpected_binary_content`, …). It is never skipped silently.

Supported spectral layouts are two-column x/y files, one spectrum per column, one spectrum per row (wavenumbers as headers), and long format (ID, wavenumber, value).

## Key settings (Section 1.3)

| Setting | Default | Purpose |
|---|---|---|
| `RANDOM_SEED` | 42 | All random sampling is reproducible |
| `SAMPLE_FILES_PER_CATEGORY` | 3 | Files shown in detail in Section 3 |
| `MAX_FILES_TO_PARSE` | `None` (all) | Set an integer for a seeded subset on very large datasets |
| `MAX_SPECTRA_FOR_PLOTS` | 300 | Spectra held in memory for plots; summary statistics cover **all** spectra |
| `HASH_ALL_FILES` | `True` | SHA-256 of every file (needed for the Bronze manifest) |
| `CLEAN_PREVIOUS_OUTPUTS` | `True` | Re-runs replace outputs instead of duplicating them |

## Reading the results

* Start with **`reports/EDA_SUMMARY.md`**.
* **`tables/qc_issues.csv`** separates `confirmed_error` (broken data) from `potential_anomaly` (possibly real chemistry, possibly an artefact). **Researchers need to review the anomalies.** The notebook does not delete anything.
* Interpretations of the y-axis quantity (absorbance vs. transmittance) are marked *declared* (from headers or metadata) or *heuristic* (from the shape of the values). Transmittance is **never** converted to absorbance automatically.
* The point spacing between data points is **not** the instrument's optical resolution; zero-filling makes it finer. Take the resolution from instrument metadata.
* `reports/draft_postgres_schema.sql` and `tables/draft_bronze_manifest.csv` are **drafts** for team review. No AWS service is contacted.
