# Data Lake & Lakehouse Design

## When to use
Use this skill when designing a data platform — from raw ingestion through curated, analytics-ready datasets. Covers the medallion (bronze/silver/gold) architecture, data pipeline patterns, governance, and query layer design.

## Medallion Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│  Policy Admin │ Claims System │ CRM │ External Feeds │ IoT     │
└──────┬────────┴──────┬────────┴──┬──┴───────┬────────┴──┬──────┘
       │               │           │          │           │
       ▼               ▼           ▼          ▼           ▼
┌─────────────────────────────────────────────────────────────────┐
│  BRONZE (Raw)                                                   │
│  • Exact copy of source data, append-only                       │
│  • Partitioned by ingestion_date                                │
│  • Format: Delta Lake (schema-on-read)                          │
│  • Retention: 7 years (regulatory)                              │
│  adls://datalake/bronze/{source}/{table}/                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Cleanse, deduplicate, type-cast
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  SILVER (Cleansed)                                              │
│  • Standardized schema, data types enforced                     │
│  • Deduplication applied (source PK + event timestamp)          │
│  • PII pseudonymized (hashed or tokenized)                      │
│  • SCD Type 2 for slowly changing dimensions                    │
│  adls://datalake/silver/{domain}/{entity}/                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Join, aggregate, business logic
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  GOLD (Curated)                                                 │
│  • Business-aligned dimensional models (star schema)            │
│  • Pre-aggregated KPIs and metrics                              │
│  • Optimized for BI tools (Power BI, Tableau)                   │
│  • Data quality checks passed (Great Expectations)              │
│  adls://datalake/gold/{business_domain}/{dataset}/              │
└─────────────────────────────────────────────────────────────────┘
```

## Storage Layout

```
adls://zurich-datalake-{env}/
├── bronze/
│   ├── policy_admin/
│   │   ├── policies/  (partitioned by ingestion_date=YYYY-MM-DD)
│   │   ├── endorsements/
│   │   └── premiums/
│   ├── claims/
│   │   ├── claims/
│   │   ├── payments/
│   │   └── reserves/
│   └── external/
│       ├── weather/
│       └── market_data/
├── silver/
│   ├── customer/
│   │   ├── dim_customer/
│   │   └── dim_broker/
│   ├── policy/
│   │   ├── dim_policy/
│   │   └── fact_premium/
│   └── claims/
│       ├── dim_claim/
│       └── fact_payment/
├── gold/
│   ├── finance/
│   │   ├── monthly_premium_summary/
│   │   └── loss_ratio_by_lob/
│   ├── actuarial/
│   │   ├── triangle_data/
│   │   └── exposure_base/
│   └── operations/
│       ├── claims_cycle_time/
│       └── sla_compliance/
└── sandbox/
    └── {user_id}/  (exploratory, auto-purged after 30 days)
```

## Pipeline Patterns

### Bronze ingestion (batch)

```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable
from datetime import date

spark = SparkSession.builder.getOrCreate()

def ingest_to_bronze(source_path: str, bronze_path: str):
    df = (spark.read
          .option("inferSchema", "true")
          .option("header", "true")
          .csv(source_path))

    (df.withColumn("_ingestion_date", F.lit(date.today().isoformat()))
       .withColumn("_source_file", F.input_file_name())
       .write
       .format("delta")
       .mode("append")
       .partitionBy("_ingestion_date")
       .save(bronze_path))
```

### Bronze to Silver (cleansing)

```python
from pyspark.sql import functions as F

def bronze_to_silver(bronze_path: str, silver_path: str, pk_cols: list[str]):
    bronze = spark.read.format("delta").load(bronze_path)

    silver = (bronze
        .dropDuplicates(pk_cols)
        .withColumn("customer_email", F.sha2(F.col("customer_email"), 256))
        .withColumn("effective_date", F.to_date("effective_date", "yyyy-MM-dd"))
        .filter(F.col("policy_number").isNotNull()))

    (silver.write
        .format("delta")
        .mode("overwrite")
        .option("overwriteSchema", "true")
        .save(silver_path))
```

### Silver to Gold (aggregation)

```python
def build_loss_ratio(silver_premium_path: str, silver_claims_path: str, gold_path: str):
    premiums = spark.read.format("delta").load(silver_premium_path)
    claims = spark.read.format("delta").load(silver_claims_path)

    loss_ratio = (premiums
        .groupBy("line_of_business", "underwriting_year")
        .agg(F.sum("written_premium").alias("total_premium"))
        .join(
            claims.groupBy("line_of_business", "underwriting_year")
                  .agg(F.sum("incurred_amount").alias("total_incurred")),
            on=["line_of_business", "underwriting_year"],
            how="left"
        )
        .withColumn("loss_ratio",
            F.round(F.col("total_incurred") / F.col("total_premium"), 4)))

    loss_ratio.write.format("delta").mode("overwrite").save(gold_path)
```

## Data Quality (Great Expectations)

```python
import great_expectations as gx

context = gx.get_context()

validator = context.sources.pandas_default.read_delta(silver_path)
validator.expect_column_values_to_not_be_null("policy_number")
validator.expect_column_values_to_be_between("premium_amount", min_value=0, max_value=10_000_000)
validator.expect_column_values_to_match_regex("policy_number", r"^ZUR-[A-Z]{2}-\d{8}$")
results = validator.validate()
```

## Data Governance

| Concern | Solution | Tool |
|---------|----------|------|
| **Data catalog** | Auto-scan and tag all Delta tables | Azure Purview |
| **Lineage** | Track column-level lineage from bronze → gold | Purview + Databricks Unity Catalog |
| **Access control** | Role-based access per zone and domain | Unity Catalog + Azure AD groups |
| **PII handling** | Pseudonymize in silver, restrict raw bronze access | Dynamic views + column masking |
| **Data quality** | Automated checks on every pipeline run | Great Expectations |
| **Retention** | Bronze: 7 years, Silver: 5 years, Gold: 3 years, Sandbox: 30 days | Delta Lake VACUUM + lifecycle policies |

## Gotchas

- Never grant direct access to bronze — it contains raw PII. All consumers should read from silver or gold.
- Delta Lake VACUUM default retention is 7 days. For regulatory data, set `delta.deletedFileRetentionDuration` to match your retention policy BEFORE running VACUUM.
- Partition by date for time-series data, by region for geo-distributed queries. Over-partitioning (e.g., by customer_id) creates small-file problems.
- Schema evolution: use `mergeSchema` for additive changes (new columns). Breaking changes (renamed/dropped columns) require a new table version and migration.
- Cost control: set auto-termination on Databricks clusters (15 min idle), use spot instances for bronze/silver jobs, reserved capacity for gold serving.
- Cross-region replication for DR must respect data residency — EU data cannot replicate to US regions even for backup.