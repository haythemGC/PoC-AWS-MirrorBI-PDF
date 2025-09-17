

---

# **Step-by-Step Code Explanation**

---

### **Imports**

```python
import json
import boto3
import base64
import os
```

* `json`: for working with JSON objects.
* `boto3`: AWS SDK for Python → lets us talk to **S3** and **Bedrock**.
* `base64`: used to encode the PDF so Claude can read it.
* `os`: used only to extract the filename (optional).

---

### **AWS Clients**

```python
s3 = boto3.client("s3")
bedrock = boto3.client("bedrock-runtime")
```

* Creates clients for **S3** and **Bedrock Runtime API**.
* These clients are used later to **download/upload files** and **call Claude**.

---

### **Question for Claude**

```python
QUESTION = (
    "Extract the most important information from this dashboard "
    "that will help when recreating it in another BI tool. "
    "Summarize KPIs, charts, filters, and key insights in structured JSON."
)
```

* This is the **fixed question** you want to ask Claude for every dashboard PDF.
* You can change this string if you want to ask something else.

---

### **Lambda Handler**

```python
def lambda_handler(event, context):
```

* Every Lambda starts here.
* `event` contains details of the S3 upload (bucket name, file key, etc.).
* `context` has runtime info (not really used here).

---

### **1. Get S3 Event Details**

```python
bucket = event['Records'][0]['s3']['bucket']['name']
key = event['Records'][0]['s3']['object']['key']
```

* Reads the **S3 bucket name** and the **file path (key)** of the uploaded PDF.
* Example:

  * Bucket = `dashboard-analysis-bucket`
  * Key = `input/my_dashboard.pdf`

---

### **2. Download the PDF**

```python
local_file = "/tmp/input.pdf"
s3.download_file(bucket, key, local_file)
```

* Lambda can’t work directly on S3 files.
* So it downloads the PDF to `/tmp/` (the only writeable folder inside Lambda).

---

### **3. Encode PDF**

```python
with open(local_file, "rb") as pdf_file:
    encoded_pdf = base64.b64encode(pdf_file.read()).decode("utf-8")
```

* Reads the PDF file.
* Encodes it into **base64** (required format for Bedrock Claude when sending documents).

---

### **4. Build Request for Claude**

```python
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
                    "text": QUESTION
                }
            ]
        }
    ],
    "max_tokens": 500
}
```

* This is the **payload for Claude**.
* It includes:

  * The PDF (encoded).
  * A title (the filename).
  * The **question** you want Claude to answer.
* `max_tokens=500` → controls max length of the answer.

---

### **5. Call Bedrock**

```python
response = bedrock.invoke_model(
    modelId="us.anthropic.claude-sonnet-4-20250514-v1:0",
    contentType="application/json",
    accept="application/json",
    body=json.dumps(request_body)
)
```

* Sends the request to Claude **through Bedrock API**.
* `modelId` → the specific Claude version you’re using.

---

### **6. Extract Claude’s Answer**

```python
response_body = json.loads(response["body"].read())

extracted_texts = [
    block.get("text", "")
    for block in response_body.get("content", [])
    if block.get("type") == "text"
]
summary_text = " ".join(extracted_texts).strip()
```

* Bedrock returns a **JSON response**.
* This extracts only the **text content** that Claude generated.
* Joins multiple pieces into one string.

---

### **7. Save JSON Back to S3**

```python
output_key = key.replace("input/", "output/").replace(".pdf", ".json")
result_json = {
    "source_file": key,
    "summary": summary_text
}

s3.put_object(
    Bucket=bucket,
    Key=output_key,
    Body=json.dumps(result_json, indent=2),
    ContentType="application/json"
)
```

* Creates an output path:

  * If input was `input/my_dashboard.pdf`
  * Output will be `output/my_dashboard.json`
* Uploads a JSON file back to the same bucket with Claude’s summary.

---

### **8. Return Result**

```python
return {
    "status": "success",
    "bucket": bucket,
    "output_file": output_key
}
```

* Lambda returns a success message so you can track the execution.

---

# ✅ Workflow in Plain Words

1. You upload `my_dashboard.pdf` into `s3://bucket/input/`.
2. S3 event triggers Lambda.
3. Lambda downloads the PDF.
4. Lambda sends it to Claude with a predefined question.
5. Claude replies with the **summary/insights**.
6. Lambda saves the summary into `s3://bucket/output/my_dashboard.json`.

---


