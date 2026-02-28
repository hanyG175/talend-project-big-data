# Data Warehouse Alimentation — College Project

> **AI & Data Science — ETL Pipeline with Talend Open Studio & PostgreSQL**

This repository documents a college project I built as part of my AI and Data Science curriculum. The goal was to design and implement a complete data warehouse solution from scratch — ingesting raw CSV sales data, processing it through an ETL pipeline, and ultimately landing it in a dimensional data warehouse optimized for analytics. It was one of the more technically demanding projects I've worked on so far, and it genuinely shifted how I think about data engineering.

---

## What This Project Does

The system reads CSV files containing sales data (clients, products, categories, transactions, etc.), processes them through a multi-stage ETL pipeline built in **Talend Open Studio for Data Integration**, and loads the results into a **PostgreSQL** database organized across two schemas: an Operational Data Store (ODS) for staging and cleansing, and a dimensional Data Warehouse (DWH) for analytical querying.

It follows the classic three-layer warehouse architecture — Source → ODS → DWH — which sounds straightforward until you're actually building it.

---

## What I Learned

This project was my first real hands-on encounter with ETL pipeline design, and I came away with a much deeper understanding of why data engineering is its own discipline.

**Understanding the ETL logic** was the first wall I hit. It wasn't immediately obvious how to think about data flow across layers — when to validate, when to deduplicate, when to transform versus when to just stage. I spent a significant amount of time mapping out the pipeline on paper before touching any Talend component, which ultimately saved me from several design mistakes down the line. Grasping how `tFileList` orchestrates file iteration, how routing works between components, and how each job fits into the broader orchestration graph (`jChargeODS → jChargeDWH → jAlimentationBDD`) was genuinely instructive.

**Debugging the Java components** was the second — and more frustrating — challenge. Talend's `tJava` components require you to write Java code snippets that execute within the Talend runtime, and when something goes wrong, the error messages are not always forthcoming. I learned to read generated Java stack traces more carefully, to isolate failing rows using logging (`routines.system.ResumeUtil`), and to be far more deliberate about null handling and type casting than I would have been otherwise.

Beyond those two core struggles, I also developed a clearer intuition for dimensional modelling — understanding why fact tables and dimension tables are structured the way they are, and how the time dimension (generated programmatically via `jGenerateCalendar`) connects everything together. The project also reinforced good habits around environment configuration: using context variables for database credentials and paths meant that the pipeline could be adapted to different environments without touching job logic.

---

## Project Structure

```
TalendProjectBigData/
├── codeForOds.java                     # Java snippets for tJava components
├── customCode.java                     # Custom utility routines
├── jGenerateCalendar.zip               # Calendar generation job
├── joursFeries.xlsx                    # Holiday reference data
├── README.md                           # This file
├── ALIMENTATION_DATA_WAREHOUSE/        # Main Talend project
│   ├── .project                        # Eclipse project configuration
│   ├── talend.project                  # Talend project definition
│   ├── code/                           # Custom routines and system utilities
│   ├── metadata/                       # Database connections and schemas
│   ├── poms/                           # Maven POMs and generated Java code
│   └── process/                        # ETL job definitions
├── csvFiles/                           # Source CSV files
│   ├── ICOM_20250605_CATEGORY.csv
│   ├── ICOM_20250605_CUSTOMER.csv
│   ├── ICOM_20250605_PRODUIT.csv
│   ├── ICOM_20250605_SOUS_CATEGORIE.csv
│   └── ICOM_20250605_TYPE_CLIENT.csv
├── scriptSQL_partie_1/                 # Database schema scripts
└── scriptSQL_partie_2/                 # Additional SQL scripts
```

---

## Architecture

### Source Layer
Raw CSV files following the naming convention `ICOM_YYYYMMDD_[ENTITY].csv`, located in `csvFiles/`. Entities include categories, customers, products, sub-categories, customer types, and sales transactions.

### ODS Layer (Operational Data Store)
The staging area. Data arrives here from the CSV files, gets validated, cleansed, and deduplicated before any transformation takes place. Jobs use `tFileList` to iterate over files and `routines.system.ResumeUtil` for comprehensive logging. This layer exists precisely so that the DWH never receives dirty data — a principle I came to appreciate only after seeing what happens when you skip it.

### DWH Layer (Data Warehouse)
The dimensional model. Fact tables (sales transactions) and dimension tables (customers, products, time) are built here from the clean ODS data. This layer is what you'd actually run analytical queries against.

---

## Database Setup

### PostgreSQL Configuration

```sql
CREATE DATABASE vente_db;
```

Three logical connections are used within the project:

- **VENTE_ODS** — ODS schema operations
- **VENTE_DWH** — Data warehouse schema operations
- **PARAMS_LOG** — Logging and pipeline parameter tracking

**Connection defaults:**

| Parameter | Value |
|-----------|-------|
| Host | localhost |
| Port | 5432 |
| Database | vente_db |
| Username | *(your username)* |
| Password | *(your password)* |

---

## Talend Setup

### Context Variables

All environment-specific configuration is handled through Talend context variables, which keeps job logic clean and portable:

| Variable | Purpose |
|----------|---------|
| `schema_ods` | ODS schema name |
| `schema_dwh` | DWH schema name |
| `database` | Database name (`vente_db`) |
| `ServerName` | Database host (`localhost`) |
| `port` | Database port (`5432`) |
| `utilisateur` | Database username |
| `password` | Database password |
| `projectFolder` | Path to the `csvFiles/` directory |

### Importing the Project

1. Open Talend Open Studio for Data Integration
2. Import the `ALIMENTATION_DATA_WAREHOUSE/` project
3. Navigate to Metadata → Db Connections and configure the three database connections found in `metadata/connections/`
4. Update context variables to match your local environment
5. Copy the `routines.customCode` class from `customCode.java` into your Talend project's routines

---

## Job Overview

### ODS Jobs (`/process/ODS/`)

These jobs extract data from CSV files and load it into the ODS staging layer:

- **jOdsCategorie** — Category data
- **jOdsClient** — Customer data
- **jOdsProduit** — Product data
- **jOdsSousCategorie** — Sub-category data
- **jOdsTypeClient** — Customer type data
- **jOdsVente** — Sales transaction data

### DWH Jobs (`/process/DWH/`)

These jobs transform ODS data into the dimensional model:

- **jDwhClient** — Customer dimension
- **jDwhProduit** — Product dimension
- **jDwhVente** — Sales fact table
- **jGenerateCalendar** — Time dimension (requires `joursFeries.xlsx` for holiday data via `tFileInputExcel`)

### Orchestration Jobs (`/process/Orechestration/`)

- **jChargeODS** — Runs all ODS loading jobs in sequence
- **jChargeDWH** — Runs all DWH transformation jobs in sequence
- **jAlimentationBDD** — The top-level job; runs the entire pipeline end-to-end

---

## Running the Pipeline

Once setup is complete:

1. Open `jAlimentationBDD` in Talend
2. Run the job — it will orchestrate the full pipeline automatically
3. Monitor the console output for logs and progress indicators

To verify results, query:
- `VENTE_ODS` schema for staged and cleansed data
- `VENTE_DWH` schema for the dimensional model

---

## File Notes

- `codeForOds.java` contains Java snippets meant to be pasted into `tJava` components — comments in the file indicate where each block belongs
- `jGenerateCalendar.zip` should be imported as a separate Talend job; it depends on `joursFeries.xlsx`
- Part 3 of the video series (build & scheduling) is not covered in the repository, but the process is straightforward — refer to the final video in the playlist linked below

---

## Video References

- [Part 1](https://www.youtube.com/watch?v=2Ncg1ieyyZQ)
- [Part 2](https://www.youtube.com/watch?v=-CZILJ4swGk)
- [Part 3](https://www.youtube.com/watch?v=lGTMhZ22-jM) *(build & scheduling — not in repo)*
