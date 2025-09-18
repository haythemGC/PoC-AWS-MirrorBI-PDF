

---

# **Step-by-Step Code Explanation**



---

## **1️⃣ Imports**

```python
import json
import boto3
import os
import base64
```

* `json`: to handle JSON data (parse and serialize).
* `boto3`: AWS SDK for Python → used to interact with **S3** and **Bedrock**.
* `os`: used to get the filename from a path (`os.path.basename`).
* `base64`: used to encode the PDF so it can be sent to Claude in the required format.

---

## **2️⃣ AWS Clients**

```python
s3 = boto3.client('s3')
bedrock = boto3.client('bedrock-runtime')
```

* Creates **clients** to communicate with AWS services.
* `s3` → download PDFs and upload JSON results.
* `bedrock` → call the Claude AI model via Bedrock Runtime API.

---

## **3️⃣ Question & Schema Instruction**

```python
QUESTION = (
    "Extract the most important information from this dashboard "
    "that will help when recreating it in another BI tool. "
    "Summarize KPIs, charts, filters, and key insights in structured JSON."
)

SCHEMA_INSTRUCTION = """
Return the answer strictly in the following JSON format:
{
  "KPIs": [...],
  "Charts": [...],
  "Filters": [...],
  "Insights": [...]
}
"""
```

* `QUESTION` → what you want Claude to analyze from the PDF.
* `SCHEMA_INSTRUCTION` → forces Claude to return **structured JSON**, ensuring the response can be parsed automatically.

> This is a key improvement: it avoids having free text and gives predictable JSON output.

---

## **4️⃣ Lambda Handler**

```python
def lambda_handler(event, context):
```

* Entry point for AWS Lambda.
* `event` → contains information about the S3 upload that triggered the Lambda.
* `context` → runtime info (not used here).

---

## **5️⃣ Get S3 Event Details**

```python
bucket = event['Records'][0]['s3']['bucket']['name']
key = event['Records'][0]['s3']['object']['key']
```

* Extracts the **bucket name** and **file key** from the S3 event.
* Example: if you uploaded `input/dashboard.pdf`, bucket = `my-bucket`, key = `input/dashboard.pdf`.

---

## **6️⃣ Download PDF from S3**

```python
local_file = "/tmp/input.pdf"
s3.download_file(bucket, key, local_file)
```

* Downloads the PDF to **Lambda’s local `/tmp/` directory** (the only writable space).

---

## **7️⃣ Encode PDF**

```python
with open(local_file, "rb") as pdf_file:
    encoded_pdf = base64.b64encode(pdf_file.read()).decode("utf-8")
```

* Reads the PDF in **binary mode**.
* Encodes it into **Base64** because Bedrock expects the PDF content as a Base64 string.

---

## **8️⃣ Build Claude Request**

```python
request_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {"type": "base64", "media_type": "application/pdf", "data": encoded_pdf},
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
```

* `request_body` is what you send to Claude.
* Contains:

  1. **PDF document** (`type: document`)
  2. **Question + JSON schema instruction** (`type: text`)
* `max_tokens = 1500` → limits the size of Claude’s response.

---

## **9️⃣ Call Bedrock / Claude**

```python
response = bedrock.invoke_model(
    modelId="eu.anthropic.claude-sonnet-4-20250514-v1:0",
    contentType="application/json",
    accept="application/json",
    body=json.dumps(request_body)
)
```

* Sends the request to Claude Sonnet 4 through **Bedrock Runtime API**.
* Returns a response with **Claude’s JSON answer**.

---

## **🔟 Parse Response & Save to S3**

```python
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
```

* Reads the JSON from Claude.
* Creates **output key** in `output/` folder with `.json` extension.
* Uploads a JSON file containing:

  * Original PDF filename
  * Claude’s JSON response

> This allows **automated downstream processing**, e.g., dashboards or ETL pipelines.

---

## **1️⃣1️⃣ Return Result**

```python
return {
    "status": "success",
    "bucket": bucket,
    "output_file": output_key
}
```

* Returns **Lambda execution info**.
* Confirms which file was processed and where the JSON is saved.

---

## ✅ **Summary / Workflow**

1. Upload PDF to `input/` in S3.
2. S3 triggers Lambda.
3. Lambda downloads PDF, encodes it, sends it to Claude with a **question + JSON schema**.
4. Claude returns **structured JSON**.
5. Lambda saves the JSON to `output/` in the same bucket.
6. You can now use this JSON for **dashboard recreation or analysis**.

---


