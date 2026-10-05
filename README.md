# Automated Data Cleaning & Quality Suite

> An automated, reusable Python-based data cleaning and quality validation suite designed to transform messy tabular datasets into cleaner, analysis-ready CSV files while maintaining an audit trail of the cleaning operations performed.

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue?logo=python)](https://www.python.org/)
[![Pandas](https://img.shields.io/badge/Pandas-Data%20Processing-150458?logo=pandas)](https://pandas.pydata.org/)
[![NumPy](https://img.shields.io/badge/NumPy-Numerical%20Computing-013243?logo=numpy)](https://numpy.org/)
[![Matplotlib](https://img.shields.io/badge/Matplotlib-Visualization-11557c)](https://matplotlib.org/)
[![Seaborn](https://img.shields.io/badge/Seaborn-Visualization-4c72b0)](https://seaborn.pydata.org/)

## Overview

Real-world datasets are rarely clean.

They may contain:

* Missing values
* Duplicate records
* Unnecessary whitespace
* Columns with extremely high missingness
* Numerical outliers
* Inconsistent data quality

Manually performing these operations repeatedly can make data-preparation workflows slow, inconsistent, and difficult to reproduce.

This project provides an object-oriented **`DataCleanerSuite`** designed to automate several common data-cleaning operations through a reusable pipeline.

The project demonstrates how a data-cleaning workflow can be converted from a collection of individual preprocessing steps into a structured, chainable cleaning engine.

The notebook uses an EEG dataset as the demonstration dataset and produces a cleaned CSV file ready for subsequent analysis or machine-learning workflows.

---

## Project Goals

The primary goals of this project are to:

1. Automate common data-cleaning operations.
2. Reduce repetitive preprocessing code.
3. Handle missing values according to column type.
4. Detect and cap numerical outliers using the IQR method.
5. Remove exact duplicate records.
6. Remove columns with excessive missing data.
7. Normalize whitespace in textual columns.
8. Maintain an audit log of cleaning operations.
9. Export the cleaned dataset as a CSV file.
10. Demonstrate how the cleaning engine can be packaged as an importable Python module.

---

## Key Features

### 1. Whitespace Normalization

The suite automatically identifies columns containing `object` or `string` data types and removes leading and trailing whitespace.

For example:

```text
"  Male  " → "Male"
" Patient 01 " → "Patient 01"
```

This helps prevent apparently identical categorical values from being treated as different values because of accidental spaces.

Implemented through:

```python
trim_whitespace()
```

---

### 2. Exact Duplicate Removal

The suite removes completely duplicated rows from the dataset.

```python
remove_duplicates()
```

Before removing duplicates, the original number of rows is recorded so the audit log can report how many records were removed.

Example audit message:

```text
Removed 15 exact duplicate rows.
```

Only exact duplicate rows are removed. The implementation does not attempt fuzzy or similarity-based duplicate detection.

---

### 3. High-Missingness Column Removal

Columns whose missing-value percentage reaches or exceeds the configured threshold are removed.

The default threshold is:

```python
threshold = 0.9
```

This means columns with **90% or more missing values** are removed.

The threshold can be changed:

```python
cleaner.drop_empty_columns(threshold=0.8)
```

This makes the operation configurable rather than permanently hard-coded to one value.

---

### 4. Automatic Missing-Value Imputation

Missing values are handled according to the detected data type.

#### Numerical Columns

By default, numerical missing values are replaced using the column median.

```python
num_strategy='median'
```

The implementation also supports mean imputation:

```python
num_strategy='mean'
```

For example:

```python
cleaner.impute_missing(num_strategy='median')
```

#### Categorical / Non-Numerical Columns

Missing categorical values are replaced with:

```text
Missing
```

This behavior can be customized:

```python
cleaner.impute_missing(cat_fill='Unknown')
```

The goal is to avoid silently discarding rows simply because some values are missing.

---

### 5. IQR-Based Outlier Capping

The main notebook pipeline uses the **Interquartile Range (IQR)** method to identify extreme numerical values.

For each numerical column:

```text
Q1 = 25th percentile
Q3 = 75th percentile
IQR = Q3 - Q1
```

The default boundaries are calculated as:

```text
Lower Bound = Q1 - 1.5 × IQR
Upper Bound = Q3 + 1.5 × IQR
```

Values outside these boundaries are capped rather than removed.

Implemented through:

```python
cap_outliers_iqr(factor=1.5)
```

For example:

```python
cleaner.cap_outliers_iqr(factor=1.5)
```

This approach preserves the rows while limiting the influence of extreme numerical observations.

> **Important:** The implementation performs outlier **capping**, not row deletion.

---

### 6. Audit Logging

The main `DataCleanerSuite` maintains an internal audit log.

Every major cleaning operation records a description of what happened.

For example:

```text
Normalized whitespace across 4 string columns.
Removed 12 exact duplicate rows.
Dropped 2 column(s) with missing rate >= 90%.
Imputed 35 missing values in 'age' using median (24.00).
Capped 17 outliers in 'value' to range [10.25, 89.75].
Exported cleaned dataset to /kaggle/working/cleaned_eeg_dataset.csv
```

The audit information can be retrieved using:

```python
cleaner.get_audit_report()
```

This provides basic transparency into the transformations applied to the dataset.

---

### 7. CSV Export

The cleaned dataset can be exported directly to a CSV file:

```python
cleaner.export_csv('/kaggle/working/cleaned_eeg_dataset.csv')
```

The resulting file contains the transformed DataFrame without the pandas index.

---

### 8. Chainable Cleaning Pipeline

The cleaning methods return `self`, allowing multiple operations to be chained together.

The main notebook uses:

```python
df_clean = (cleaner
            .trim_whitespace()
            .remove_duplicates()
            .drop_empty_columns(threshold=0.9)
            .impute_missing(num_strategy='median')
            .cap_outliers_iqr(factor=1.5)
            .export_csv('/kaggle/working/cleaned_eeg_dataset.csv')
            .df)
```

This creates a readable sequential workflow:

```text
Raw Data
   ↓
Trim Whitespace
   ↓
Remove Duplicates
   ↓
Remove Highly Missing Columns
   ↓
Impute Missing Values
   ↓
Cap Numerical Outliers
   ↓
Export Clean Dataset
```

---

# Dataset

The demonstration notebook uses:

**EEG.machinelearing_data_BRMH.csv**

The dataset is loaded from the Kaggle input directory:

```python
raw_df = pd.read_csv(
    '/kaggle/input/datasets/zuhaibalisays/messydata/EEG.machinelearing_data_BRMH.csv'
)
```

The notebook first displays the shape of the raw dataset:

```python
print(f"Raw Dataset Shape: {raw_df.shape}")
```

The cleaning suite itself is designed around tabular pandas DataFrames, so the cleaning class is not inherently limited to this particular EEG dataset.

---

# Data Quality Visualization

Before running the cleaning pipeline, the notebook visualizes missingness using the `missingno` library.

```python
msno.matrix(
    raw_df,
    sparkline=False,
    figsize=(10, 3)
)
```

This provides a visual representation of missing values in the original dataset.

The purpose is to inspect the dataset's missing-data structure before transformations are applied.

---

# Architecture

The core implementation is organized around a single class:

```text
DataCleanerSuite
```

The class receives a pandas DataFrame and creates an internal copy:

```python
self.df = df.copy()
```

This means the original DataFrame passed to the class is not directly modified by the cleaning operations.

The class also maintains:

```python
self.audit_log = []
```

for tracking cleaning operations.

---

# Main Class API

## `DataCleanerSuite(df)`

Creates a new cleaning engine from an existing pandas DataFrame.

```python
cleaner = DataCleanerSuite(raw_df)
```

### Parameters

| Parameter | Type               | Description            |
| --------- | ------------------ | ---------------------- |
| `df`      | `pandas.DataFrame` | Input dataset to clean |

---

## `trim_whitespace()`

Removes leading and trailing whitespace from string/object columns.

```python
cleaner.trim_whitespace()
```

Returns:

```python
self
```

allowing method chaining.

---

## `remove_duplicates()`

Removes exact duplicate rows.

```python
cleaner.remove_duplicates()
```

The number of removed rows is recorded in the audit log.

---

## `drop_empty_columns(threshold=0.9)`

Removes columns whose missing-value ratio is greater than or equal to the specified threshold.

```python
cleaner.drop_empty_columns(threshold=0.9)
```

### Example

To remove columns containing at least 80% missing values:

```python
cleaner.drop_empty_columns(threshold=0.8)
```

---

## `impute_missing(num_strategy='median', cat_fill='Missing')`

Automatically handles missing values.

### Numerical data

Supported strategies:

```text
median
mean
```

Example:

```python
cleaner.impute_missing(num_strategy='median')
```

### Categorical data

Default replacement:

```text
Missing
```

Custom replacement:

```python
cleaner.impute_missing(cat_fill='Unknown')
```

---

## `cap_outliers_iqr(factor=1.5)`

Detects extreme numerical values using IQR boundaries and caps them to those boundaries.

```python
cleaner.cap_outliers_iqr(factor=1.5)
```

A different factor can be supplied:

```python
cleaner.cap_outliers_iqr(factor=2.0)
```

---

## `export_csv(file_path)`

Exports the cleaned DataFrame as a CSV file.

```python
cleaner.export_csv(
    '/kaggle/working/cleaned_dataset.csv'
)
```

---

## `get_audit_report()`

Returns the list of recorded cleaning operations.

```python
report = cleaner.get_audit_report()

for item in report:
    print(item)
```

---

# Complete Notebook Workflow

The demonstration notebook follows four major stages.

## Step 1 — Load Raw Data

The dataset is loaded using pandas:

```python
raw_df = pd.read_csv(...)
```

The original dataset dimensions are displayed.

---

## Step 2 — Inspect Missingness

The notebook creates a missing-value matrix using `missingno`.

```python
msno.matrix(raw_df)
```

This provides an initial visual quality check.

---

## Step 3 — Execute Cleaning Pipeline

The cleaning engine is initialized:

```python
cleaner = DataCleanerSuite(raw_df)
```

The operations are then executed sequentially:

```python
df_clean = (cleaner
            .trim_whitespace()
            .remove_duplicates()
            .drop_empty_columns(threshold=0.9)
            .impute_missing(num_strategy='median')
            .cap_outliers_iqr(factor=1.5)
            .export_csv('/kaggle/working/cleaned_eeg_dataset.csv')
            .df)
```

---

## Step 4 — Review Audit Log

The notebook prints the operations performed:

```python
for idx, step in enumerate(cleaner.get_audit_report(), 1):
    print(f"{idx}. {step}")
```

This gives a simple execution summary of the cleaning process.

---

# Standalone Cleaning Module

The notebook also generates a separate Python module:

```text
cleaner_module.py
```

The module contains another implementation of:

```python
DataCleanerSuite
```

and provides a simplified:

```python
run_full_clean()
```

method.

The module performs:

1. Whitespace trimming
2. Duplicate removal
3. Removal of columns with ≥90% missing values
4. Median imputation for numerical columns
5. CSV export

Example:

```python
from cleaner_module import DataCleanerSuite

cleaner = DataCleanerSuite(df)

clean_df = cleaner.run_full_clean(
    export_path='cleaned_dataset.csv'
)
```

### Difference Between Notebook and Standalone Module

The notebook's primary `DataCleanerSuite` and the generated standalone module are intentionally not identical.

The **notebook implementation** includes:

* Individual chainable cleaning methods
* Configurable numerical imputation strategy
* Configurable categorical fill value
* Configurable missing-column threshold
* IQR outlier capping
* Audit logging
* CSV export

The **standalone `cleaner_module.py`** currently provides a more compact workflow through `run_full_clean()` and does not include the notebook's IQR outlier-capping or audit-log functionality.

---

# Installation

Install the required Python libraries:

```bash
pip install numpy pandas matplotlib seaborn missingno
```

If using Jupyter Notebook:

```bash
pip install jupyter
```

---

# Requirements

The project uses the following main Python libraries:

| Library      | Purpose                                   |
| ------------ | ----------------------------------------- |
| `pandas`     | DataFrame manipulation and CSV processing |
| `numpy`      | Numerical operations                      |
| `matplotlib` | Visualization                             |
| `seaborn`    | Plot styling                              |
| `missingno`  | Missing-data visualization                |

---

# Usage Outside Kaggle

The core cleaning class can be adapted for a local Python environment.

Example:

```python
import pandas as pd

from cleaner_module import DataCleanerSuite

df = pd.read_csv("messy_dataset.csv")

cleaner = DataCleanerSuite(df)

clean_df = cleaner.run_full_clean(
    export_path="cleaned_dataset.csv"
)

print(clean_df.head())
```

The current notebook uses Kaggle-specific paths such as:

```text
/kaggle/input/
/kaggle/working/
```

When running locally, replace those paths with local file paths.

---

# Example: Full Custom Pipeline

The main cleaning class can be used as a configurable pipeline:

```python
cleaner = DataCleanerSuite(df)

clean_df = (
    cleaner
    .trim_whitespace()
    .remove_duplicates()
    .drop_empty_columns(threshold=0.9)
    .impute_missing(
        num_strategy='median',
        cat_fill='Missing'
    )
    .cap_outliers_iqr(factor=1.5)
    .export_csv('cleaned_dataset.csv')
    .df
)
```

Then inspect the cleaning history:

```python
for step in cleaner.get_audit_report():
    print(step)
```

---

# Cleaning Pipeline at a Glance

```text
                 RAW DATASET
                      │
                      ▼
             ┌─────────────────┐
             │ Missingness     │
             │ Visualization   │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Trim Whitespace │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Remove Exact    │
             │ Duplicates      │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Remove Columns  │
             │ ≥ 90% Missing   │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Impute Missing  │
             │ Values          │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ IQR Outlier     │
             │ Capping         │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Audit Log       │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ Clean CSV       │
             └─────────────────┘
```

---

# Design Decisions

## Why Median Imputation?

The default numerical imputation strategy is the median.

Median imputation is less sensitive to extreme values than mean imputation, making it a practical default for messy numerical datasets.

The implementation also allows mean imputation:

```python
cleaner.impute_missing(num_strategy='mean')
```

---

## Why Remove Highly Missing Columns?

A column containing an extremely high proportion of missing values may provide limited usable information.

The default threshold is:

```text
90%
```

This is configurable and should be adjusted according to the characteristics and requirements of the dataset.

---

## Why Cap Instead of Remove Outliers?

The main pipeline caps numerical outliers rather than deleting entire rows.

This preserves observations while limiting the influence of extreme numerical values.

The approach is implemented using:

```python
numpy.clip()
```

after calculating IQR-based lower and upper bounds.

---

# Important Limitations

This project is an automated **general-purpose cleaning utility**, not a universal data-validation framework.

The current implementation has several limitations.

### 1. No Semantic Type Detection

The suite relies primarily on pandas data types.

It does not automatically determine whether a column represents:

* Age
* Income
* Date
* Medical measurement
* Identifier
* Geographic coordinate
* Target variable

Therefore, domain-specific validation is still necessary.

### 2. No Date Parsing

The current implementation does not automatically detect or convert date/time columns.

### 3. No Fuzzy Duplicate Detection

Only exact duplicate rows are removed.

Similar but non-identical records are not automatically identified.

### 4. Outlier Detection Is Numerical Only

IQR-based outlier capping is applied to numerical columns.

Categorical anomalies are not automatically detected.

### 5. No Domain-Specific Validation

The suite does not automatically determine whether a value makes sense in a particular domain.

For example, a negative age would not automatically be identified as invalid simply because it is numerical.

### 6. No Automatic Data-Type Conversion

The implementation does not perform general automatic conversion such as:

```text
string → datetime
string → numeric
integer → categorical
```

unless handled separately by the user.

### 7. Missingness Threshold Is a Heuristic

The 90% threshold is a configurable heuristic, not a universally correct rule.

Different datasets may require different thresholds.

---

# Reproducibility

The pipeline is deterministic for a fixed input dataset and configuration.

The main cleaning operations do not use random sampling or random initialization.

To reproduce the workflow:

1. Use the same input dataset.
2. Use the same cleaning parameters.
3. Run the pipeline in the same sequence.

---

# Project Structure

A suggested repository structure is:

```text
automated-data-cleaning-quality-suite/
│
├── README.md
├── cleaner_module.py
├── notebook/
│   └── automated_data_cleaning_quality_suite.ipynb
│
└── data/
    └── README.md
```

The actual Kaggle notebook can remain hosted on Kaggle while the reusable Python module can be maintained in the GitHub repository.

---

# Kaggle Notebook

The original Kaggle notebook is available here:

**Automated Data Cleaning & Quality Suite**

https://www.kaggle.com/code/zuhaibalisays/automated-data-cleaning-quality-suite

---

# Example Output

After executing the pipeline, the project generates:

```text
cleaned_eeg_dataset.csv
```

and an audit summary similar to:

```text
Pipeline Execution Summary

1. Normalized whitespace across X string columns.
2. Removed X exact duplicate rows.
3. Dropped X column(s) with missing rate >= 90%.
4. Imputed X missing values in numerical columns.
5. Capped X outliers in numerical columns.
6. Exported cleaned dataset to /kaggle/working/cleaned_eeg_dataset.csv
```

The exact values depend on the input dataset.

---

# Use Cases

This cleaning suite can serve as a starting point for:

* Exploratory Data Analysis (EDA)
* Machine-learning preprocessing
* Data-quality experiments
* Kaggle projects
* Data-science portfolios
* Reusable preprocessing pipelines
* Educational demonstrations of data cleaning
* Rapid preprocessing of tabular datasets

It is especially useful when the same basic cleaning operations need to be applied repeatedly.

---

# Future Improvements

Potential improvements for future versions include:

* Automatic data-type inference
* Date/time detection and normalization
* Configurable outlier strategies
* Z-score outlier detection
* Isolation Forest-based anomaly detection
* Data validation rules
* Column-level quality reports
* Before/after quality metrics
* Missing-value statistics
* Duplicate statistics
* Outlier summaries
* JSON audit reports
* HTML data-quality reports
* Logging to files
* Configuration through YAML/JSON
* Unit tests
* Command-line interface
* Support for Excel and Parquet files
* More comprehensive standalone module functionality
* Schema validation
* Dataset profiling
* Automated quality scoring

---

# Security & Data Handling

This project operates on the DataFrame supplied by the user.

The cleaning engine does not include any external API calls, cloud database integrations, or remote data-upload functionality.

When using sensitive datasets, users should still follow the privacy and security requirements applicable to their data.

---

# Technical Summary

| Component                 | Implementation         |
| ------------------------- | ---------------------- |
| Programming Language      | Python                 |
| Data Processing           | Pandas                 |
| Numerical Operations      | NumPy                  |
| Missingness Visualization | Missingno              |
| Visualization             | Matplotlib / Seaborn   |
| Architecture              | Object-oriented        |
| Cleaning Style            | Chainable pipeline     |
| Duplicate Handling        | Exact duplicates       |
| Missing Column Threshold  | 90% by default         |
| Numerical Imputation      | Median by default      |
| Categorical Imputation    | `"Missing"` by default |
| Outlier Method            | IQR                    |
| Default IQR Factor        | 1.5                    |
| Outlier Treatment         | Capping                |
| Audit Trail               | In-memory audit log    |
| Output                    | CSV                    |
| Demonstration Environment | Kaggle Notebook        |

---

# What This Project Demonstrates

This project goes beyond simply calling individual pandas functions.

It demonstrates several important data-science engineering concepts:

* Object-oriented data preprocessing
* Reusable cleaning components
* Method chaining
* Configurable preprocessing
* Missing-data handling
* Statistical outlier detection
* Data-quality visualization
* Audit logging
* CSV data export
* Converting notebook logic into a reusable Python module

The goal is to make common data-cleaning operations more structured, readable, and reusable.

---

# Conclusion

**Automated Data Cleaning & Quality Suite** provides a compact foundation for automating common data-quality operations on tabular datasets.

The main notebook demonstrates a complete workflow from raw-data inspection through cleaning, outlier treatment, audit logging, and CSV export.

The project also demonstrates how notebook-based preprocessing logic can be separated into a reusable Python module, providing a foundation for developing a more complete data-quality framework in future iterations.

---

## Author

**Zuhaib Ali**

Data Science Student | Python | Machine Learning | Data Analytics

GitHub: `zuhaibalisays`

Kaggle: `zuhaibalisays`

---

## License

Distributed under the MIT License. See `LICENSE` for more information.
