# UrbanMart — Medallion Lakehouse on Databricks

End-to-end data engineering capstone: raw retail CSV files are loaded into a **Bronze → Silver → Gold** lakehouse on Databricks (Unity Catalog, Delta Lake, SQL) and modelled as a **Kimball star schema**, with data-quality rules, automated validation, orchestration and analytics.

**Author:** Efkleia Karagkiozi

## Business questions

UrbanMart is a Greek retail chain with 12 physical stores and 1 online store. The sales data mart answers five questions:

1. What are monthly revenue, gross margin and units sold, by region and store?
2. Which product categories and brands drive the most margin?
3. Who are the top customers per segment?
4. What is the return rate by category and store type?
5. How do weekdays vs weekends and online vs physical stores compare?

## Source data

| File | Rows | Description |
|---|---:|---|
| customers.csv | 508 | Customer master |
| customer_changes.csv | 25 | City, region and segment changes (used for SCD2) |
| products.csv | 76 | Catalogue with cost and list price |
| stores.csv | 13 | 12 physical stores + 1 online store |
| orders.csv | 8,000 | Order headers, 2024-01-01 → 2025-12-30 |
| order_items.csv | 24,111 | Order lines with quantity, unit price and discount |

The data contains deliberate data-quality defects, and finding and handling them was part of the project.

## Architecture

| Layer | Schema | Content |
|---|---|---|
| Bronze | `bronze_efkleia_urbanmart` | 6 raw CSVs stored as STRING + lineage columns (`_source_file`, `_ingested_at`, `_source_row_hash`) |
| Silver | `silver_efkleia_urbanmart` | Cleansed, typed, de-duplicated tables + `etl_rejects` |
| Gold | `gold_efkleia_urbanmart` | Star schema: `fact_sales` + `dim_date`, `dim_customer`, `dim_product`, `dim_store`, `dim_customer_scd2` |

**Pipeline (Lakeflow Job):** `00_setup → 01a_bronze → 02_silver → 03_gold → 04_validation`
Each task depends on the previous one, and `04_validation` acts as the final gate: any failing assertion fails the job.

## Notebooks

| Notebook | What it does |
|---|---|
| `00_setup` | Creates the catalog widget, the three schemas and the landing volume |
| `01a_bronze` | Ingests the CSVs with `read_files`, keeps all values as STRING, adds lineage and row hashes |
| `01b_profiling` | Profiles the bronze data, records the Data Quality Log and the dimensional design (not part of the job) |
| `02_silver` | Cleanses and types the data, de-duplicates it, and routes invalid rows to `etl_rejects` |
| `03_gold` | Builds the dimensions (surrogate keys, SCD1 `MERGE`, unknown member −1) and `fact_sales` (PK/FK and CHECK constraints) |
| `04_validation` | Reconciliation and integrity assertions, a failure test with time travel/`RESTORE`, and an idempotency proof |
| `05_analytics` | Business questions Q1–Q5 that feed the dashboard |
| `06_stretch_scd2` | Overview of the SCD Type 2 stretch goal |

## Highlights

- **Data quality:** 13 documented issues, including duplicates, mixed timestamp formats, negative quantities, orphan records and guest orders, each with an explicit handling rule.
- **Reconciliation:** 24,111 bronze lines − 114 rejected = 23,997 fact rows. Silver and gold net amounts match exactly.
- **Idempotent loads:** the dimension `MERGE`s and `INSERT OVERWRITE` loads give identical results on every re-run, which Delta time travel verifies.
- **SCD Type 2:** `dim_customer_scd2` tracks changes to city, region and segment, and `fact_sales` uses a point-in-time join to link each sale to the customer version valid on the order date.
- **Analytics:** monthly revenue and margin by store, top categories and brands, top customers per segment, return rates, and weekday vs weekend / online vs physical sales.

## Key findings

- Physical stores earn about the same per day on weekdays and on weekends.
- Electronics sold in Mall stores has the highest return rate.
- The SCD2 history shows 10 customers upgrading their loyalty segment and 6 leaving the loyalty programme.

## Documentation

[`docs/Documentation_Capstone.pdf`](docs/Documentation_Capstone.pdf) explains each notebook, its results and the design decisions, with screenshots of the ERD, the dashboard, the validation failure test and the Lakeflow Job.

## Tech stack

Databricks · Unity Catalog · Delta Lake · Spark SQL · Lakeflow Jobs · AI/BI Dashboards

## How to run

1. Upload the source CSVs to `/Volumes/<catalog>/bronze_efkleia_urbanmart/landing/`.
2. Run the notebooks in order, `00_setup` → `05_analytics`, or schedule them as a job with the parameter `catalog = <your catalog>`.

The source data comes from the bootcamp and is not included in this repository.
