

1. Save a **Power BI (or any) dashboard as PDF** into **S3**.
2. That **triggers a Lambda function**.
3. Lambda uploads the **PDF to Claude (via Bedrock API)** and **asks a single predefined question**.
4. The **answer is parsed into JSON**.
5. Lambda saves the JSON back into the **same S3 bucket**.

Here’s a **step-by-step deployment guide**:

---

## **1. Prerequisites**

* An **AWS account** with permissions for:

  * S3
  * Lambda
  * Bedrock
  * IAM
* An existing **Claude model** in Bedrock (e.g., `anthropic.claude-3-sonnet-20240229-v1:0`).

---

## **2. Create an S3 Bucket**

1. Go to **S3 console** → Create bucket.

   * Example: `dashboard-analysis-bucket`
2. Enable **default encryption** (AES-256).
3. Create a folder structure if you like:

   * `input/` → PDFs go here
   * `output/` → JSON results

---

## **3. Create an IAM Role for Lambda**

1. Go to **IAM console** → Roles → Create Role.
2. Trusted entity: **AWS Lambda**.
3. Attach policies:

   * `AmazonS3FullAccess` (or scoped to your bucket only).
   * `AmazonBedrockFullAccess` (or scoped).
   * `CloudWatchLogsFullAccess` (for debugging).
4. Save role as: `LambdaBedrockS3Role`.

---

## **4. Create the Lambda Function**

1. Go to **Lambda console** → Create function.

   * Name: `DashboardAnalyzer`.
   * Runtime: **Python 3.12**.
   * Role: `LambdaBedrockS3Role`.

2. In **Configuration → General Configuration**, increase timeout to **2–3 minutes** (since Bedrock inference may take time).

---

## **5. Add S3 Trigger**

1. In Lambda → **Triggers** → Add trigger.
2. Select **S3** → Bucket: `dashboard-analysis-bucket`.
3. Event type: **PUT**.
4. Prefix filter: `input/` (so it only runs for new PDFs uploaded to `input/`).

---

## **6. Lambda Code**

Here’s a **Python sample**:

```python
import json
import boto3
import os
import base64

s3 = boto3.client('s3')
bedrock = boto3.client('bedrock-runtime')

QUESTION = (
    "Extract the most important information from this dashboard "
    "that will help when recreating it in another BI tool. "
    "Summarize KPIs, charts, filters, and key insights in structured JSON."
)
SCHEMA_INSTRUCTION = """
Return the answer strictly in the following JSON format:

{
  "KPIs": [
    {
      "name": "string",
      "value": "string",
      "unit": "string (optional)"
    }
  ],
  "Charts": [
    {
      "title": "string",
      "type": "string (bar, line, pie, table, etc.)",
      "x_axis": "string",
      "y_axis": "string",
      "metrics": ["list of metrics shown"],
      "dimensions": ["list of dimensions used"]
    }
  ],
  "Filters": [
    {
      "name": "string",
      "type": "string (dropdown, date range, checkbox, etc.)",
      "values": ["list of available values"]
    }
  ],
  "Insights": [
    "string (key observation or conclusion from the dashboard)"
  ]
}
"""


def lambda_handler(event, context):
    
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    local_file = "/tmp/input.pdf"
    s3.download_file(bucket, key, local_file)

    with open(local_file, "rb") as pdf_file:
        encoded_pdf = base64.b64encode(pdf_file.read()).decode("utf-8")
    
    request_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "base64",
                        "media_type": "application/pdf",
                        "data": encoded_pdf
                    },
                    "title": os.path.basename(key),
                    "citations": {"enabled": True},
                    "cache_control": {"type": "ephemeral"}
                },
                {
                    "type": "text",
                    "text": f" {QUESTION}\n\n{SCHEMA_INSTRUCTION}"
                }
            ]
        }
    ],
    "max_tokens": 1500
}
    response = bedrock.invoke_model(
        modelId="eu.anthropic.claude-sonnet-4-20250514-v1:0",
        contentType="application/json",
        accept="application/json",
        body=json.dumps(request_body)
    )
    response_body = json.loads(response["body"].read())
    output_key = key.replace("input/", "output/").replace(".pdf", ".json")
    result_json = {
    "source_file": key,
    "summary": response_body
    }
    s3.put_object(
        Bucket=bucket,
        Key=output_key,
        Body=json.dumps(result_json),
        ContentType="application/json"
    )
 
    return {
        "status": "success",
        "bucket": bucket,
        "output_file": output_key
    }

```

---

## **7. Testing**

1. Upload a dashboard PDF into:
   `s3://dashboard-analysis-bucket/input/my_dashboard.pdf`

2. Lambda runs automatically.

3. JSON summary is saved into:
   `s3://dashboard-analysis-bucket/output/my_dashboard.json`
4. Create test event:
   * add event name **TestPDFUpload**.
   * the event:
     ` {
  "Records": [
    {
      "s3": {
        "bucket": { "name": "dashboard-analysis-bucket" },
        "object": { "key": "input/Competitive Marketing Analysis.pdf" }
      }
    }
  ]
}
 `
---

## **8. Monitoring & Logs**

* Go to **CloudWatch Logs** for debugging Lambda.
* Add CloudWatch alarms if needed for failures.

---

## **9. Enhancements**

* Add **error handling** (invalid PDFs, Bedrock timeout).
* Store metadata (filename, timestamp, model used).
* Add an **SNS notification** when JSON is ready.

---


