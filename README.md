# 🚀 End-to-End Batch & Streaming Data Pipeline
### Real-Time Product Recommendation System on AWS

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-purple?logo=terraform)](https://www.terraform.io/)
[![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)](https://python.org)
[![PostgreSQL](https://img.shields.io/badge/VectorDB-pgvector-336791?logo=postgresql)](https://github.com/pgvector/pgvector)

---

## 📌 Overview

This project implements a **full end-to-end data engineering pipeline** that covers both **batch processing** and **real-time streaming** to power a product recommendation system. Built on AWS, the architecture ingests raw user and product data, transforms it for ML training, stores embeddings in a vector database, and delivers real-time recommendations through a streaming pipeline.

> **Course:** Data Engineering Specialization — Week 4 Assignment  
> **Theme:** Thinking like a Data Engineer — translating stakeholder requirements into functional architectures

---

## 🏗️ Architecture

### Batch Pipeline
```
Amazon RDS (MySQL)
        │
        ▼
  AWS Glue ETL Job
        │
        ▼
  Amazon S3 (Data Lake)
  └── ratings-ml-training/
      └── customerNumber=<N>/   ← partitioned data
```

### Streaming Pipeline
```
Kinesis Data Streams (live user activity)
        │
        ▼
Kinesis Data Firehose
        │
        ├──► Lambda (stream-transformation)
        │         │
        │         ├── S3 (ML model artifacts)
        │         └── PostgreSQL + pgvector (embeddings)
        │
        ▼
  Amazon S3 (Recommendations Bucket)
  └── year/month/day/hour/
```

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| **Infrastructure as Code** | Terraform |
| **Source Database** | Amazon RDS MySQL (classicmodels) |
| **ETL** | AWS Glue (PySpark) |
| **Data Lake** | Amazon S3 |
| **Vector Database** | Amazon RDS PostgreSQL + pgvector |
| **Streaming** | Amazon Kinesis Data Streams + Firehose |
| **Serverless Compute** | AWS Lambda |
| **ML Artifacts** | Amazon S3 |
| **Monitoring** | Amazon CloudWatch |

---

## 📁 Project Structure

```
.
├── data/
│   └── mysqlsampledatabase.sql        # Source DB schema
├── images/
│   ├── de-c1w4-diagram-batch.drawio.png
│   ├── de-c1w4-diagram-stream.drawio.png
│   ├── schema_after_ETL.png
│   └── terraform_plan.png
├── scripts/
│   └── setup.sh                       # Environment setup
├── sql/
│   └── embeddings.sql                 # Loads embeddings into vector DB
└── terraform/
    ├── main.tf                        # Root Terraform config
    ├── outputs.tf
    ├── variables.tf
    ├── backend.tf
    └── modules/
        ├── etl/                       # AWS Glue + S3 resources
        │   ├── glue.tf
        │   ├── iam_roles.tf
        │   ├── network.tf
        │   ├── outputs.tf
        │   ├── policies.tf
        │   └── variables.tf
        ├── vector-db/                 # PostgreSQL + pgvector
        │   ├── rds.tf
        │   ├── iam_roles.tf
        │   ├── network.tf
        │   ├── outputs.tf
        │   ├── policies.tf
        │   └── variables.tf
        └── streaming-inference/       # Kinesis + Lambda
            ├── kinesis.tf
            ├── lambda.tf
            ├── iam_roles.tf
            ├── outputs.tf
            ├── policies.tf
            └── variables.tf
```

---

## 🚀 Deployment Guide

### Prerequisites
- AWS CLI configured with appropriate permissions
- Terraform >= 1.0
- MySQL client
- psql client

---

### Step 1 — Setup Environment

```bash
source ./scripts/setup.sh
cd terraform
```

---

### Step 2 — Deploy the Batch Pipeline (AWS Glue + S3)

Uncomment `module "etl"` in `terraform/main.tf` (lines 1–15) and `terraform/outputs.tf` (lines 2–8), then run:

```bash
terraform init
terraform plan
terraform apply
```

Start the Glue ETL job:

```bash
# Start job
aws glue start-job-run --job-name de-c1w4-etl-job | jq -r '.JobRunId'

# Monitor status (replace <JobRunID>)
aws glue get-job-run --job-name de-c1w4-etl-job \
  --run-id <JobRunID> \
  --output text \
  --query "JobRun.JobRunState"
```

✅ Wait for status: `SUCCEEDED` (~2–3 min)

---

### Step 3 — Deploy the Vector Database (PostgreSQL + pgvector)

Uncomment `module "vector_db"` in `terraform/main.tf` (lines 17–27) and `terraform/outputs.tf` (lines 11–27), then apply:

```bash
terraform init
terraform plan
terraform apply
```

⏳ Database creation takes ~7 minutes.

Retrieve credentials:

```bash
terraform output vector_db_master_username
terraform output vector_db_master_password
terraform output vector_db_host
```

Load embeddings into the vector DB:

```bash
# Connect to PostgreSQL
psql --host=<VectorDBHost> --username=postgres --password --port=5432

# Switch to postgres DB and run SQL script
\c postgres;
\i '../sql/embeddings.sql'

# Verify tables
\dt *.*
\q
```

> ⚠️ Before running `embeddings.sql`, replace `<BUCKET_NAME>` with your actual S3 bucket name (`de-c1w4-<AWS-ACCOUNT-ID>-us-east-1-ml-artifacts`) in **two places** in the file.

---

### Step 4 — Configure the Model Inference Lambda

In AWS Console → Lambda → `de-c1w4-model-inference` → Configuration → Environment Variables:

| Variable | Value |
|---|---|
| `VECTOR_DB_HOST` | Output of `terraform output vector_db_host` |
| `VECTOR_DB_USER` | Output of `terraform output vector_db_master_username` |
| `VECTOR_DB_PASSWORD` | Output of `terraform output vector_db_master_password` |

---

### Step 5 — Deploy the Streaming Pipeline (Kinesis + Lambda + S3)

Uncomment `module "streaming_inference"` in `terraform/main.tf` (lines 29–39) and `terraform/outputs.tf` (lines 30–32), then apply:

```bash
terraform init
terraform plan
terraform apply
```

Monitor the pipeline:
- **S3 Recommendations Bucket:** `de-c1w4-<ACCOUNT-ID>-recommendations`
- **CloudWatch Logs:** Lambda → `de-c1w4-transformation-lambda` → Monitor → View CloudWatch Logs

Data arrives partitioned as:
```
year/month/day/hour/de-c1w4-delivery-stream-<PLACEHOLDER>
```

> ⏳ Allow ~5 minutes for data to appear (Kinesis event interval ~10 seconds).

---

## 🗄️ Data Schema

### Source: `ratings` table (MySQL)
Added to the `classicmodels` database — contains user-to-product ratings (scale 1–5).

### After ETL Transformation
The Glue job produces training-ready data partitioned by `customerNumber`:

```
S3://de-c1w4-<ACCOUNT>-datalake/ratings-ml-training/
└── customerNumber=<N>/
    └── *.parquet
```

### ML Artifacts Structure (S3)
```
de-c1w4-<ACCOUNT>-us-east-1-ml-artifacts/
├── embeddings/
│   ├── item_embeddings.csv
│   └── user_embeddings.csv
├── model/
│   └── best_model.pth
└── scalers/
    ├── item_ohe.pkl
    ├── item_std_scaler.pkl
    ├── user_ohe.pkl
    └── user_std_scaler.pkl
```

---

## 🧠 How the Recommendation Works

1. User places a product in cart → event streamed via **Kinesis Data Streams**
2. **Kinesis Firehose** picks up the stream and triggers the **transformation Lambda**
3. Lambda extracts user & product features and runs inference using the trained `.pth` model
4. The model computes an **embedding vector** for the item
5. **pgvector** in PostgreSQL finds the most similar item embeddings (nearest neighbor search)
6. Recommendations are written to the **S3 recommendations bucket**

---

## 📊 Key AWS Services Used

```
Amazon RDS MySQL     →  Source transactional database
AWS Glue             →  Serverless ETL (PySpark under the hood)
Amazon S3            →  Data lake, ML artifacts, recommendations output
Amazon RDS PostgreSQL →  Vector database with pgvector extension
AWS Lambda           →  Serverless model inference & stream transformation
Amazon Kinesis       →  Real-time data streaming
CloudWatch           →  Logging & monitoring
Terraform            →  Infrastructure provisioning (IaC)
```

---

## ⚠️ Notes

- The AWS Console URL expires every **15 minutes** — regenerate with:
  ```bash
  cat ~/.aws/aws_console_url
  ```
- Terraform `sensitive` outputs (passwords) must be accessed explicitly:
  ```bash
  terraform output vector_db_master_password
  ```
- Always run Terraform commands from inside the `terraform/` directory.

---

## 📄 License

This project was developed as part of a data engineering course assignment. All AWS infrastructure is provisioned ephemerally for the lab duration.

---

<p align="center">
  Built with ❤️ using AWS · Terraform · Python · pgvector
</p>