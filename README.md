# Module 3 Homework – Data Warehousing & BigQuery

This repository contains my solution for **Module 3 – Data Warehousing & BigQuery** from the Data Engineering Zoomcamp.

The goal of this homework is to practice working with **Google Cloud Storage (GCS)** and **BigQuery**, including:
- External tables
- Managed (materialized) tables
- Columnar storage behavior
- Partitioning and clustering
- Query cost optimization

---

## Dataset

We use **NYC Yellow Taxi Trip Records** for the period:

**January 2024 – June 2024**

Data source (Parquet format):  
https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

---

## Prerequisites

- Google Cloud Project
- `gcloud`, `gsutil`, and `bq` CLI installed
- BigQuery dataset created in **US region**
- GCS bucket created
- Authentication via:
  ```bash
  gcloud auth application-default login
````

---

## Step 1 – Download the data

Download the Parquet files (Jan–Jun 2024):

```bash
for i in {01..06}; do
  wget https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-${i}.parquet
done
```

---

## Step 2 – Upload data to GCS

Create a folder structure in GCS and upload the files:

```bash
gsutil mkdir -p gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data
```

```bash
gsutil cp yellow_tripdata_2024-*.parquet \
gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data/
```

Verify that all 6 files exist:

```bash
gsutil ls gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data/
```

---

## Step 3 – Create External Table in BigQuery

Create an external table using the Parquet files stored in GCS:

```bash
bq mk \
  --external_table_definition=PARQUET=gs://yellow_taxi_data_homework/yellow_taxi_data/taxi_parquet_data/yellow_tripdata_2024-*.parquet \
  qwiklabs-gcp-02-de81f5c9e9bf:taxi_data_2024.external_yellow_taxi
```

Verify record count:

```sql
SELECT COUNT(*) AS total_rows
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.external_yellow_taxi`;
```

**Result:** `20,332,093`

---

## Step 4 – Create Managed (Materialized) Table

Create a regular BigQuery table from the external table (no partitioning or clustering):

```sql
CREATE OR REPLACE TABLE
`qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`
AS
SELECT *
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.external_yellow_taxi`;
```

Verify row count:

```sql
SELECT COUNT(*)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

---

## Question 1 – Counting records

```sql
SELECT COUNT(*)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.external_yellow_taxi`;
```

**Answer:** `20,332,093`

---

## Question 2 – Data read estimation

Count distinct pickup locations:

```sql
SELECT COUNT(DISTINCT PULocationID)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.external_yellow_taxi`;
```

```sql
SELECT COUNT(DISTINCT PULocationID)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

**Estimated data read:**

* External table: **~2.14 GB**
* Managed table: **0 MB**

---

## Question 3 – Understanding columnar storage

```sql
SELECT PULocationID
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

```sql
SELECT PULocationID, DOLocationID
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

**Explanation:**
BigQuery uses columnar storage and only scans the columns requested. Querying two columns requires reading more data than querying one column.

---

## Question 4 – Zero fare trips

```sql
SELECT COUNT(*)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`
WHERE fare_amount = 0;
```

**Answer:** `8,333`

---

## Question 5 – Partitioning and clustering

Best optimization strategy:

* **Partition by** `tpep_dropoff_datetime`
* **Cluster by** `VendorID`

Create optimized table:

```sql
CREATE OR REPLACE TABLE
`qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_partitioned`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID
AS
SELECT *
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

---

## Question 6 – Partition benefits

Non-partitioned table:

```sql
SELECT DISTINCT VendorID
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`
WHERE tpep_dropoff_datetime
BETWEEN '2024-03-01' AND '2024-03-15';
```

Partitioned table:

```sql
SELECT DISTINCT VendorID
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_partitioned`
WHERE tpep_dropoff_datetime
BETWEEN '2024-03-01' AND '2024-03-15';
```

**Estimated bytes processed:**

* Non-partitioned: **~310 MB**
* Partitioned: **~26 MB**

---

## Question 7 – External table storage

**Answer:**
Data is stored in **Google Cloud Storage (GCS)**.

---

## Question 8 – Clustering best practices

**Answer:**
**False** – clustering should only be used when it matches query patterns.

---

## Question 9 – Understanding table scans

```sql
SELECT COUNT(*)
FROM `qwiklabs-gcp-02-de81f5c9e9bf.taxi_data_2024.yellow_taxi_managed`;
```

**Estimated bytes:** `0 MB`

**Reason:**
BigQuery uses table metadata to return row counts without scanning data.

---

## Summary of Answers

| Question | Answer                                    |
| -------- | ----------------------------------------- |
| Q1       | 20,332,093                                |
| Q2       | 2.14 GB & 0 MB                            |
| Q3       | Columnar storage behavior                 |
| Q4       | 8,333                                     |
| Q5       | Partition by dropoff, cluster by VendorID |
| Q6       | 310.24 MB & 26.84 MB                      |
| Q7       | GCP Bucket                                |
| Q8       | False                                     |

---

## Rahmatulloh

Homework completed as part of
**Data Engineering Zoomcamp – Module 3**



Tinggal bilang 👍
```
