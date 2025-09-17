

# 📊 PoC-AWS-MirrorBI-PDF – AWS Serverless Project

## 📌 Project Overview

The **Dashboard Analyzer** is a serverless solution built on **AWS** to automatically analyze dashboard reports (in PDF format).

The workflow is:

1. A user **uploads a dashboard PDF** into an **S3 bucket**.
2. The upload **triggers a Lambda function**.
3. Lambda sends the PDF to **Claude (via Amazon Bedrock)** with a predefined question.
4. Claude analyzes the dashboard and produces a **summary in JSON format**.
5. The summary is stored back in the **same S3 bucket** under `output/`.

This allows teams to quickly extract **key insights, KPIs, charts, and filters** from dashboards and reuse them when migrating to another BI tool (like Power BI, Tableau, or QuickSight).

---

## ⚙️ Architecture

```
           ┌────────────────────────┐
           │    User Uploads PDF    │
           └──────────┬─────────────┘
                      │
                      ▼
          ┌─────────────────────────┐
          │       Amazon S3         │
          │   (bucket /input/)      │
          └──────────┬──────────────┘
                     │ Trigger
                     ▼
           ┌────────────────────────┐
           │       AWS Lambda       │
           │ - Downloads PDF        │
           │ - Encodes as Base64    │
           │ - Sends to Bedrock     │
           │ - Parses response      │
           └──────────┬─────────────┘
                      │
                      ▼
          ┌─────────────────────────┐
          │ Amazon Bedrock (Claude) │
          │ - Analyzes dashboard    │
          │ - Returns JSON summary  │
          └──────────┬──────────────┘
                     │
                     ▼
          ┌─────────────────────────┐
          │       Amazon S3         │
          │   (bucket /output/)     │
          └─────────────────────────┘
```

---

## 🚀 How It Works

1. **Upload a dashboard PDF** into:

   ```
   s3://dashboard-analysis-bucket/input/my_dashboard.pdf
   ```
2. **S3 triggers Lambda**, which:

   * Downloads the PDF.
   * Encodes it in Base64.
   * Sends it with a question to **Claude via Bedrock**.
   * Parses the response.
3. **Claude returns JSON** with key insights.
4. **Lambda uploads the JSON** into:

   ```
   s3://dashboard-analysis-bucket/output/my_dashboard.json
   ```

---

## 🔑 Key Components

* **Amazon S3** → Stores input PDFs and output JSON results.
* **AWS Lambda** → Orchestrates the workflow (download → Bedrock → upload).
* **Amazon Bedrock (Claude)** → Analyzes the dashboard and extracts insights.
* **IAM Role** → Grants Lambda access to S3, Bedrock, and CloudWatch Logs.

---


---

## 📖 How to Deploy

1. **Create S3 bucket** → `dashboard-analysis-bucket`

   * Folders: `input/` and `output/`
2. **Create IAM Role for Lambda**

   * Attach `iam-policy.json` from `infra/`.
3. **Deploy Lambda function** (`handler.py`)

   * Runtime: Python 3.12
   * Timeout: 2–3 minutes
   * Attach IAM Role.
4. **Add S3 Trigger** → Event type: `PUT`, prefix: `input/`.
5. **Upload a test PDF** into `input/`.
6. **Check `output/` folder** for JSON result.

---

## 🎯 Use Cases

* Migrating dashboards from **Power BI / Tableau / Qlik** to another tool.
* Extracting **KPIs and charts** from historical dashboards.
* Automating **BI report documentation**.

---

## 📌 Next Steps

* Add **SNS notifications** when a summary is ready.
* Store results in **DynamoDB** for querying.
* Build a **frontend app** to browse dashboard insights.

---




