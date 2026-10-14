# MarketPulse AI --- Data and Context Layer

The **Data and Context Layer** prepares retail sales, product, store,
and contextual data for the MarketPulse AI forecasting and
inventory-optimization pipeline. It provides dataset adapters, schema
validation, preprocessing, stockout handling, calendar/context features,
and cold-start priors.

> **Status:** Hackathon project component. Confirm the current data
> schemas and loader return signature with the other team members before
> integrating downstream modules.

## Contents

-   [Overview](#overview)
-   [Architecture](#architecture)
-   [Repository Structure](#repository-structure)
-   [Requirements](#requirements)
-   [Setup](#setup)
-   [Loading a Dataset](#loading-a-dataset)
-   [Data and Feature Pipeline](#data-and-feature-pipeline)
-   [Data Quality and Testing](#data-quality-and-testing)
-   [Data Files and Git](#data-files-and-git)
-   [Integration Notes](#integration-notes)
-   [Troubleshooting](#troubleshooting)

## Overview

This component is intended to provide downstream forecasting,
optimization, and evaluation modules with consistent retail data.

Key responsibilities include:

-   Loading supported datasets through a common loader.
-   Producing and validating sales, store, item/product, and context
    tables.
-   Generating synthetic retail data for development and reproducible
    testing.
-   Supporting public-dataset adapters, including Favorita and M5 where
    configured.
-   Handling stockout-related records carefully so that zero observed
    sales during unavailable inventory are not automatically interpreted
    as zero demand.
-   Creating calendar/context features and cold-start priors.
-   Keeping time-series feature engineering and evaluation splits
    chronological to reduce data leakage.

## Architecture

At a high level, the data flow is:

``` text
Raw or generated data
        |
        v
Dataset adapter / synthetic generator
        |
        v
Schema validation and data-quality checks
        |
        v
Cleaning, stockout handling, and context features
        |
        v
Processed tables and feature matrix
        |
        v
Forecasting, inventory optimization, and evaluation
```

## Repository Structure

``` text
data_and_context_layer/
├── config/
│   └── synthetic_config.yaml
├── data/
│   ├── raw/                 # Raw input CSVs or source data
│   └── processed/           # Generated feature tables and metadata
├── src/
│   ├── canonical_schemas.py
│   ├── synthetic_generator.py
│   ├── pipeline.py
│   ├── context_layer.py
│   ├── regional_adapters/
│   ├── cold_start.py
│   ├── public_dataset_adapters.py
│   └── loader.py
├── tests/
├── CONTRACT.md
├── DATA_QUALITY_REPORT.md
├── README.md
├── requirements.txt
├── export_processed.py
└── verify_high_class.py
```

The exact contents may change as the hackathon implementation evolves.

## Requirements

-   Python 3.10 or newer is recommended; use the version tested by your
    team.
-   Git
-   Python packages listed in `requirements.txt`

## Setup

### 1. Clone the repository

``` bash
git clone <YOUR_REPOSITORY_URL>
cd data_and_context_layer
```

Replace `<YOUR_REPOSITORY_URL>` with the URL of your repository.

### 2. Create and activate a virtual environment

**Windows PowerShell:**

``` powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

**Git Bash:**

``` bash
python -m venv venv
source venv/Scripts/activate
```

### 3. Install dependencies

``` bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## Loading a Dataset

The central entry point is `load_dataset` in `src/loader.py`.

The current implementation has supported names including:

-   `synthetic`
-   `synthetic_sparse`
-   `m5`
-   `favorita`

Check `src/loader.py` for the exact names, adapter requirements, and
current function signature before running a dataset. The implementation
may require local source data for public datasets.

Example pattern:

``` python
from src.loader import load_dataset

result = load_dataset("synthetic")
print(type(result))
```

Inspect the implementation to determine the returned tables and their
order. The current local implementation may return **four**
DataFrames---sales, stores, items, and context---so do not assume a
three-table return signature unless the team contract and implementation
have been aligned.

## Data and Feature Pipeline

The pipeline is designed to support the following areas:

### Dataset preparation

-   Load raw or generated data.
-   Normalize column names and data types.
-   Validate required fields and keys.
-   Check duplicate store--item--date records and other data-quality
    problems.

### Stockout handling

A recorded sale of zero does not necessarily mean demand was zero if the
product was unavailable. Stock availability should be considered when
interpreting sales and constructing demand-related features.

### Context features

Context may include calendar indicators such as weekdays, weekends,
paydays, market days, school terms, or local events, depending on the
adapter and available source data. When context data is unavailable,
fallback behavior should be explicitly flagged rather than silently
treated as observed data.

### Cold-start support

Cold-start priors can provide initial estimates for products or stores
with little or no historical sales. These estimates should be derived
only from information available at the forecast cutoff.

### Time-series safety

Feature generation and train/validation/test splitting should respect
time order. Rolling statistics, lags, and other predictive features must
not use future information.

Refer to `CONTRACT.md` for the agreed data contract and
`DATA_QUALITY_REPORT.md` for documented quality checks and known
limitations.

## Data Quality and Testing

The `tests/` directory contains tests for areas such as:

-   Canonical schemas
-   Reproducibility of synthetic data
-   Synthetic ground-truth behavior
-   Time leakage
-   Stockout handling
-   Regional adapters and fallback behavior
-   Cold-start priors

Run the test suite from the repository root:

``` bash
python -m pytest -q
```

If pytest is not installed in your environment:

``` bash
python -m pip install pytest
python -m pytest -q
```

Use the test results as the source of truth for what currently passes;
this README does not claim that every test has already passed in every
environment.

## Data Files and Git

The repository may contain raw CSV inputs and processed outputs such as:

-   `data/raw/sales.csv`
-   `data/raw/stores.csv`
-   `data/raw/items.csv`
-   `data/raw/context.csv`
-   `data/processed/full_ml_feature_matrix.csv`
-   `data/processed/cold_start_priors.csv`
-   `data/processed/items_catalog.csv`
-   `data/processed/stores_metadata.csv`

Before committing datasets, check:

1.  **Size:** GitHub blocks individual files larger than 100 MiB in
    ordinary Git pushes. Large files may need Git LFS or external
    storage.
2.  **Licensing:** Verify that any public dataset can be redistributed
    under its license.
3.  **Privacy:** Do not commit personal, confidential, or restricted
    data.
4.  **Reproducibility:** Document how generated outputs can be
    recreated, including relevant configuration and random seeds.
5.  **Repository scope:** Decide with the team whether processed CSVs
    should be versioned or generated locally. Avoid committing duplicate
    or unnecessary outputs.

To review what Git will commit:

``` bash
git status
git diff --cached --stat
```

To commit selected files:

``` bash
git add README.md CONTRACT.md DATA_QUALITY_REPORT.md
git commit -m "docs: add project documentation"
git push origin main
```

Adjust the branch name if your repository uses a different default
branch.

## Integration Notes

Before connecting this component to forecasting, optimization, or
evaluation modules, agree on:

-   The exact `load_dataset(name)` signature and return order.
-   Canonical names, data types, and nullability of columns.
-   The forecast target column (for example, `units_sold` if that is the
    approved team contract).
-   The meaning of unknown stock and missing context.
-   The handling of duplicate store--item--date records.
-   Which features are safe at prediction time.
-   How cold-start priors and fallback context are represented.

The contract and implementation should agree. If the team contract
specifies three tables but the loader returns four, resolve that
mismatch before integration rather than silently dropping a table.

## Troubleshooting

### `ModuleNotFoundError`

Run commands from the repository root and ensure the virtual environment
is active. Confirm dependencies are installed and that the project
package/import structure matches the command being run.

### Dataset adapter cannot find source data

Check the adapter documentation and expected source-data location.
Public datasets may require separate download, extraction, or
configuration steps.

### Tests fail

Run the failing test alone for a clearer error:

``` bash
python -m pytest tests/test_schemas.py -q
```

Replace the test path with the failing test. Do not mark a test as
passed until it has been run successfully.

### Processed files do not appear on GitHub

Check `git status` and `git diff --cached --stat`, commit the intended
files, and push the branch. Also check `.gitignore` rules and the size
of each file.

## Contributing

1.  Keep schema and contract changes documented.
2.  Add or update tests when behavior changes.
3.  Avoid time leakage in time-series features and splits.
4.  Document data provenance, assumptions, and fallback behavior.
5.  Run the relevant tests before submitting changes.

------------------------------------------------------------------------
