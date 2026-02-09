Here is a professional, clean, and developer-friendly `README.md` version of your homework. I’ve polished the language to sound natural—like a developer documenting their work—while keeping the technical details accurate.

I also added a strategic image tag for the partitioning section, as that concept is much easier to grasp visually.

---

# Module 3 Homework: Data Warehousing & BigQuery

This repository contains my solution for **Module 3** of the Data Engineering Zoomcamp.

In this module, I focused on building a Data Warehouse using **BigQuery** and **Google Cloud Storage (GCS)**. The tasks involved setting up external tables, optimizing query performance through partitioning and clustering, and understanding BigQuery internals like columnar storage and cost estimation.

## 📂 Dataset

I worked with the **NYC Yellow Taxi Trip Records** for the first half of 2024.

* **Period:** January 2024 – June 2024
* **Format:** Parquet
* **Source:** [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

## 🛠️ Setup & Prerequisites

To reproduce these steps, you need:

* A Google Cloud Platform (GCP) Project.
* The Google Cloud SDK installed (`gcloud`, `gsutil`, `bq`).
* A BigQuery dataset created in the **US** region.
* A GCS bucket for staging the data.

**Authentication:**

```bash
gcloud auth application-default login

```

---

## 🚀 Workflow

### 1. Download the Data

I used a simple bash loop to download the Parquet files for Jan–Jun 2024.

```bash
# Download Yellow Taxi data (Months 01-06)
for i in {01..06}; do
  wget https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-${i}.parquet
done

```

### 2. Upload to GCS

Next, I organized the files and uploaded them to my GCS bucket.

```bash
# Create directory structure
gsutil mkdir -p gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data

# Upload files
gsutil cp yellow_tripdata_2024-*.parquet gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data/

```

### 3. Create External Table

I created an external table in BigQuery that reads directly from the Parquet files in GCS without moving the data.

```bash
bq mk \
  --external_table_definition=PARQUET=gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data/yellow_tripdata_2024-*.parquet \
  <YOUR_PROJECT_ID>:taxi_data_2024.external_yellow_taxi

```

### 4. Create Managed (Materialized) Table

For performance comparison, I also created a native BigQuery table from the external source.

```sql
CREATE OR REPLACE TABLE `<YOUR_PROJECT_ID>.taxi_data_2024.yellow_taxi_managed`
AS
SELECT *
FROM `<YOUR_PROJECT_ID>.taxi_data_2024.external_yellow_taxi`;

```

---

## 📝 Homework Questions & Solutions

### Question 1: Record Count

**Objective:** count the total rows in the external table.

```sql
SELECT COUNT(*) 
FROM `<YOUR_PROJECT_ID>.taxi_data_2024.external_yellow_taxi`;

```

**Answer:** `20,332,093`

### Question 2: Data Read Estimation

**Objective:** Compare the estimated bytes read for counting distinct `PULocationID` between the External Table and the Materialized Table.

```sql
SELECT COUNT(DISTINCT PULocationID) FROM `<YOUR_PROJECT_ID>.taxi_data_2024.external_yellow_taxi`;
SELECT COUNT(DISTINCT PULocationID) FROM `<YOUR_PROJECT_ID>.taxi_data_2024.yellow_taxi_managed`;

```

**Answer:**

* External Table: **~2.14 GB** (Scans the Parquet files)
* Materialized Table: **0 MB** (BigQuery optimized metadata/caching)

### Question 3: Columnar Storage Behavior

**Objective:** Why does selecting two columns read more data than selecting one?

```sql
SELECT PULocationID FROM ...; 
-- vs --
SELECT PULocationID, DOLocationID FROM ...;

```

**Answer:**
BigQuery is a **columnar store**. It only reads the specific columns requested in the query. Retrieving two columns (`PULocationID`, `DOLocationID`) requires scanning more data blocks than retrieving just one.

### Question 4: Zero Fare Trips

**Objective:** Count how many records have a fare amount of 0.

```sql
SELECT COUNT(*)
FROM `<YOUR_PROJECT_ID>.taxi_data_2024.yellow_taxi_managed`
WHERE fare_amount = 0;

```

**Answer:** `8,333`

### Question 5: Partitioning and Clustering Strategy

**Objective:** Determine the best way to optimize the table for queries filtering by `tpep_dropoff_datetime` and ordering by `VendorID`.

**Answer:**

* **Partition by:** `tpep_dropoff_datetime` (To prune partitions based on the date filter).
* **Cluster by:** `VendorID` (To sort data within partitions for faster retrieval).

**SQL Implementation:**

```sql
CREATE OR REPLACE TABLE `<YOUR_PROJECT_ID>.taxi_data_2024.yellow_taxi_partitioned`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID
AS
SELECT *
FROM `<YOUR_PROJECT_ID>.taxi_data_2024.yellow_taxi_managed`;

```

### Question 6: Partitioning Benefits

**Objective:** Compare bytes processed when querying a specific date range (March 1-15, 2024).

**Query:**

```sql
SELECT DISTINCT VendorID
FROM `<TABLE_NAME>`
WHERE tpep_dropoff_datetime BETWEEN '2024-03-01' AND '2024-03-15';

```

**Answer:**

* Non-partitioned table: **~310.24 MB**
* Partitioned table: **~26.84 MB**

### Question 7: External Table Storage

**Objective:** Where is the data for an external table actually stored?

**Answer:**
**GCP Bucket** (Google Cloud Storage). The metadata is in BigQuery, but the actual bytes remain in the bucket.

### Question 8: Clustering Best Practices

**Objective:** Is it always best practice to cluster?

**Answer:**
**False**. Clustering adds overhead to data ingestion and re-clustering. For small tables (<1GB), the performance gain is negligible compared to the management cost.

---

## 📊 Summary

| Question | Answer |
| --- | --- |
| **Q1** | 20,332,093 |
| **Q2** | 0 MB (Managed) vs 2.14 GB (External) |
| **Q3** | BigQuery is a columnar store |
| **Q4** | 8,333 |
| **Q5** | Partition by `tpep_dropoff_datetime`, Cluster by `VendorID` |
| **Q6** | ~310 MB vs ~26 MB |
| **Q7** | GCP Bucket |
| **Q8** | False |

---

**Author:** Rahmatulloh

*Data Engineering Zoomcamp 2025*
