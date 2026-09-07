# HDB Resale Price ETL Pipeline

## Overview

This is a Python-based ETL pipeline for HDB resale flat price dataset.

The pipeline:

1. Downloads source files from data.gov.sg via API using API Key
2. Combines the source files into one master dataset [Filter by date Jan 2012 to Dec 2016]
3. Data Profiles using ydata_profiling
4. Quarantines records that [fails data profile rules & has similar transactions for the same month, other identifiers]
5. Calculates the remaining lease
6. Creates the Resale Identifier [Using Mentioned Transformation Logics]
7. Creates the hashed identifier [Using Salted hash]
8. Saves the final outputs

The main notebook to run is:

**000_Run_Pipeline.ipynb**

---

## Setup

Python 3.10+ is recommended.

Create a virtual environment.

### Windows

```bash
python -m venv venv
venv\Scripts\activate
```

Install the required packages:

```bash
pip install -r requirements.txt
pip install notebook
```

`notebook` is installed separately because it provides the `jupyter notebook` command used below. `requirements.txt` covers everything the pipeline itself needs, but not the Jupyter web interface.

The `.env` file is already included in this repository at the project root. No additional setup is needed.

An internet connection is required, `001_Download.ipynb` fetches the source files from data.gov.sg.

---

## How to Run

Start Jupyter:

```bash
jupyter notebook
```

Open:

```text
000_Run_Pipeline.ipynb
```

Run the main execution cell.

The pipeline will run all four stages in order.

A successful run produces:

```text
data/Raw/
data/Combined/resale_flat_prices_2012_01_to_2016_12.parquet
data/Cleaned/cleaned_records.parquet
data/Quarantined/quarantined_records.parquet
data/Transformed/Transformed.parquet
data/Hashed/Hashed.parquet
DataProfile/HDB_Profiling_Report.html
pipeline_execution.log
```

The pipeline execution notebook also shows a final table listing the output locations and their status.

## Project Structure

```text
├── 000_Run_Pipeline.ipynb
├── 001_Download.ipynb
├── 002_File_Process.ipynb
├── 003_DataProfile_Validation.ipynb
├── 004_DataTransformation.ipynb
├── requirements.txt
├── .env
├── Data Platform Architecture.png
├── DataProfile/
│   └── HDB_Profiling_Report.html
└── data/
    ├── Raw/
    ├── Combined/
    ├── Cleaned/
    ├── Quarantined/
    ├── Transformed/
    └── Hashed/
```

`data/` is not commited to this repository. Its created when `000_Run_Pipeline.ipynb` runs and hold the outputs of each stage.

## What Each Notebook Does

### 000_Run_Pipeline.ipynb

This is the main entry point for the pipeline

It runs the notebooks in this order

```text
001_Download.ipynb
        ↓
002_File_Process.ipynb
        ↓
003_DataProfile_Validation.ipynb
        ↓
004_DataTransformation.ipynb
```

It also checks the required environment file, shows the progress of each step, records the execution time and writes the execution output to `pipeline_execution.log`.

### 001_Download.ipynb

Downloads the source datasets from data.gov.sg .

The raw files are saved under:

```text
data/Raw/
```

The script discovers the available datasets from the collection rather than manually listing each download URL.

### 002_File_Process.ipynb

Combines the raw CSV files into one master dataset and converts to .parquet

Not all the source files contain same columns, so the union of the columns is kept and missing values are left blank. Datafiles from Jan 2015 has an additional column *'remaining_lease'*

Only records from January 2012 to December 2016 are filtered and processed further.

Output:

```text
data/Combined/resale_flat_prices_2012_01_to_2016_12.parquet
```

### 003_DataProfile_Validation.ipynb

Handles data profiling and validation.

It:

- Generates the profiling report using *ydata_profiling* library
- Validates Date, Town, Flat Type, Flat Model and Storey Range
- Uses January 2012 as the baseline
  - When it fails, it checks for relevant values like new house types and new towns
- Checks for duplicates using the composite key
- Keeps the higher resale price when duplicate records are found
- Calculates the remaining lease using 99-year lease period
- Identifies potential resale price anomalies using IQR
- Saves clean and quarantined records

Outputs:

```text
DataProfile/HDB_Profiling_Report.html
data/Cleaned/cleaned_records.parquet
data/Quarantined/quarantined_records.parquet
```

### 004_DataTransformation.ipynb

Creates `resale_identifier` based on the rules specified in the PDF. then creates hashed identifier using SHA256 and pepper from .env file

Outputs:

```text
data/Transformed/Transformed.parquet
data/Hashed/Hashed.parquet
```

---

## Data Directory

Quick reference for every file the pipeline reads or writes:

| File / Folder        | Path                                                          | Description                                                                                                                                              |
| -------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Raw Data             | `data/Raw/`                                                   | Original CSV files downloaded from data.gov.sg, as is                                                                                                    |
| Combined Data        | `data/Combined/resale_flat_prices_2012_01_to_2016_12.parquet` | All raw files unioned together and filtered to Jan 2012 to Dec 2016 and converted to parquet                                                             |
| Cleaned Data         | `data/Cleaned/cleaned_records.parquet`                        | Records that passed every validation check                                                                                                               |
| Quarantined Data     | `data/Quarantined/quarantined_records.parquet`                | Records that failed validation, duplicates and flagged as anomalous                                                                                      |
| Transformed Data     | `data/Transformed/Transformed.parquet`                        | Cleaned records plus the Resale Identifier                                                                                                               |
| Hashed Data          | `data/Hashed/Hashed.parquet`                                  | Transformed records plus the hashed identifier                                                                                                           |
| Architecture Diagram | `Data Platform Architecture.png`                              | AWS Architecture (Part 2)                                                                                                                                |
| Environment Config   | `.env`                                                        | Holds `DATAGOV_API_KEY` and `APP_PEPPER`. **<u>Included in this submission for the tester visiblity only</u>**, *see the Environment File section below* |

## Other Output Files

### Profiling Report

```text
DataProfile/HDB_Profiling_Report.html
```

The automated data profiling report generated during the validation step.

### Pipeline Execution Log

```text
pipeline_execution.log
```

Contains the output from the pipeline run and can be used to check the execution history and troubleshooting.

---

## Environment File

The pipeline uses values stored in the `.env` file for the data.gov.sg API key and the hashing pepper.

The file contains variables such as:

```text
DATAGOV_API_KEY=...
APP_PEPPER_V2=...
```

For this technical test, the `.env` file is included with the submission so the tester can run the pipeline seemlesly

> **Production note:** The `.env` file wwill not be committed to source control or stored together with the application code in a production environment. Secrets will be retrieved from AWS Secrets Manager 

---

## Architecture

`Data Platform Architecture.png` 

AWS data solution architecture for data ingestion and data exploration (Part 2), covering the batch ingestion path from data.gov.sg and Tableau analytics.



## Processing Flow

```text
000_Run_Pipeline.ipynb
      |
      v
     data.gov.sg
      |
      v
001_Download.ipynb
      |
      v
      data/Raw/
      |
      v
002_File_Process.ipynb
      |
      v
      data/Combined/
      |
      v
003_DataProfile_Validation.ipynb
      |
      +-------------------------+
      |                         |
      v                         v
      data/Cleaned/        data/Quarantined/
      |
      v
004_DataTransformation.ipynb
      |
      +-------------------------+
      |                         |
      v                         v
 data/Transformed/        data/Hashed/
```

## Notes

- Run the notebooks from the project root because the pipeline uses relative paths.
- The raw source files are not manually edited.
- Each stage can also be opened and run separately when troubleshooting.
- `000_Run_Pipeline.ipynb` is the recommended entry point for a normal run.
