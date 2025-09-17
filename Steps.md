

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

s3 = boto3.client("s3")
bedrock = boto3.client("bedrock-runtime")

# Predefined question
QUESTION = "Summarize the most important insights from this dashboard for use in recreating it in another BI tool."

def lambda_handler(event, context):
    # 1. Get S3 event details
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # 2. Download the PDF
    local_file = '/tmp/input.pdf'
    s3.download_file(bucket, key, local_file)

    # 3. Read PDF as base64 text
    with open(local_file, "rb") as f:
        pdf_bytes = f.read()

    # Convert to base64 string
    import base64
    encoded_pdf = base64.b64encode(pdf_bytes).decode("utf-8")

    # 4. Send to Bedrock Claude
    prompt = f"""
    You are given a dashboard in PDF format.
    Task: {QUESTION}
    Provide a structured JSON response with the key insights.
    """

    response = bedrock.invoke_model(
        modelId="anthropic.claude-3-sonnet-20240229-v1:0",
        contentType="application/json",
        accept="application/json",
        body=json.dumps({
            "messages": [
                {"role": "user", "content": [
                    {"type": "text", "text": prompt},
                    {"type": "document", "name": "dashboard.pdf", "media_type": "application/pdf", "data": encoded_pdf}
                ]}
            ],
            "max_tokens": 500
        })
    )

    result = json.loads(response['body'].read())
    output_text = result['output']['content'][0]['text']

    # 5. Save JSON result back to S3
    output_key = key.replace("input/", "output/").replace(".pdf", ".json")
    s3.put_object(
        Bucket=bucket,
        Key=output_key,
        Body=json.dumps({"summary": output_text}, indent=2),
        ContentType="application/json"
    )

    return {"status": "success", "output_file": output_key}
```

---

## **7. Testing**

1. Upload a dashboard PDF into:
   `s3://dashboard-analysis-bucket/input/my_dashboard.pdf`

2. Lambda runs automatically.

3. JSON summary is saved into:
   `s3://dashboard-analysis-bucket/output/my_dashboard.json`

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

👉 Do you want me to also **give you Terraform/CloudFormation** steps to automate all this setup, or you prefer to deploy it **manually from the AWS console**?
