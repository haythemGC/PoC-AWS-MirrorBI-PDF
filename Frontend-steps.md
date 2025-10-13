Perfect, Haythem 👌 — since you already have your **S3 bucket (`dashboard-analysis-bucket`)** with **`input/`** and **`output/`** folders and your region is **Frankfurt (`eu-central-1`)**, we’ll go through the **exact AWS Console steps** to set up everything:

You’ll end up with a **working Lambda + API Gateway** that your HTML page can call to get pre-signed URLs for upload and download.

---

## 🪄 Overview

Here’s what we’ll do:

1. ✅ Create the Lambda function
2. 🔑 Create or attach an IAM role with S3 permissions
3. 🌐 Create the API Gateway endpoint (`/get-presigned-url`)
4. 🧩 Connect everything and test
5. 💻 Update your HTML file

---

## 🧱 Step 1 — Create the Lambda Function

1. Go to **AWS Console → Lambda → Create function**

2. Choose:

   * **Author from scratch**
   * Function name: `mirrorbi_presign_lambda`
   * Runtime: **Python 3.12**
   * Region: **eu-central-1**
   * Execution role: **Create a new role with basic Lambda permissions**

3. Click **Create function**

---

## 🔑 Step 2 — Add S3 Permissions to the Lambda Role

1. After creation, go to the **Configuration** tab of your Lambda → **Permissions** section.
2. Click the **Role name** (e.g., `mirrorbi_presign_lambda-role-xyz`).
3. In the **IAM Role** page → **Add permissions → Attach policies**.
4. Choose **AmazonS3FullAccess** (simplest for now).
   👉 Later, you can replace it with a minimal custom policy like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": [
        "arn:aws:s3:::dashboard-analysis-bucket/input/*",
        "arn:aws:s3:::dashboard-analysis-bucket/output/*"
      ]
    }
  ]
}
```

---

## 🧠 Step 3 — Paste the Lambda Code

Go back to your **Lambda function → Code tab**, and replace everything with this code 👇

```python
import json
import boto3

s3 = boto3.client('s3')
BUCKET_NAME = 'dashboard-analysis-bucket'  # your existing bucket name

def lambda_handler(event, context):
    body = json.loads(event['body'])
    filename = body['filename']

    # Create paths inside your bucket
    upload_key = f"input/{filename}"
    result_key = f"output/{filename.replace('.', '_result.json')}"

    # Generate pre-signed upload URL (PUT)
    upload_url = s3.generate_presigned_url(
        'put_object',
        Params={'Bucket': BUCKET_NAME, 'Key': upload_key},
        ExpiresIn=3600
    )

    # Generate pre-signed result download URL (GET)
    result_url = s3.generate_presigned_url(
        'get_object',
        Params={'Bucket': BUCKET_NAME, 'Key': result_key},
        ExpiresIn=3600
    )

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*',
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'uploadUrl': upload_url,
            'resultUrl': result_url
        })
    }
```

---

## ⚙️ Step 4 — Add CORS (to allow browser access)

Since your HTML page runs from another domain (S3 static site), you need to allow **CORS**.

The Lambda already adds this header:

```python
'Access-Control-Allow-Origin': '*'
```

✅ That’s enough for this use case.

If you later secure it, you can replace `*` with your website domain.

---

## 🌐 Step 5 — Create API Gateway Endpoint

1. Go to **API Gateway → Create API → HTTP API**
2. Choose:

   * **Build** → HTTP API
   * Click **Add integration → Lambda**
   * Select your Lambda function (`mirrorbi_presign_lambda`)
3. Under **Routes**, create:

   * Method: `POST`
   * Path: `/get-presigned-url`
4. Click **Next** and **Deploy**.
5. Note the **Invoke URL**, something like:

   ```
   https://abc123xyz.execute-api.eu-central-1.amazonaws.com/get-presigned-url
   ```

---

## 🧪 Step 6 — Test Your Endpoint

You can test it with `curl` or Postman:

```bash
curl -X POST https://abc123xyz.execute-api.eu-central-1.amazonaws.com/get-presigned-url \
  -H "Content-Type: application/json" \
  -d '{"filename": "test.pdf"}'
```

✅ You should get a JSON response like:

```json
{
  "uploadUrl": "https://dashboard-analysis-bucket.s3.eu-central-1.amazonaws.com/input/test.pdf?...",
  "resultUrl": "https://dashboard-analysis-bucket.s3.eu-central-1.amazonaws.com/output/test_result.json?..."
}
```

---

## 🧩 Step 7 — Update Your HTML File

In your `mirrorBI_index.html`, find this line:

```js
const presignRes = await fetch('/get-presigned-url', {
```

Change it to:

```js
const presignRes = await fetch('https://abc123xyz.execute-api.eu-central-1.amazonaws.com/get-presigned-url', {
```

That’s your API Gateway invoke URL.

---

## 🌍 Step 8 — Deploy your static website (if not yet)

If your page is hosted in S3 as a static site:

1. Upload `mirrorBI_index.html` to your S3 bucket (or another “website” bucket).
2. Enable static website hosting under **Properties**.
3. Set index document = `mirrorBI_index.html`.
4. Visit your S3 site URL:

   ```
   http://your-bucket-name.s3-website.eu-central-1.amazonaws.com
   ```
5. Try uploading a file — it should:

   * Upload to `input/` folder
   * Wait for the result in `output/`
   * Display the JSON when ready

---

## ✅ Quick Recap

| Step | Description              | Key Output                                |
| ---- | ------------------------ | ----------------------------------------- |
| 1    | Create Lambda            | `mirrorbi_presign_lambda`                 |
| 2    | Add permissions          | Access to S3 bucket paths                 |
| 3    | Paste Python code        | Generates pre-signed URLs                 |
| 4    | CORS config              | `Access-Control-Allow-Origin: *`          |
| 5    | Create API Gateway route | `/get-presigned-url`                      |
| 6    | Test endpoint            | Returns upload + result URLs              |
| 7    | Update HTML              | Replace `/get-presigned-url` with API URL |
| 8    | Deploy static website    | S3 static hosting URL works               |

---

Would you like me to show you **how to automatically trigger your analysis Lambda (Bedrock/Textract)** when a new file is uploaded to the `input/` folder?
That’s the next step to complete the full workflow 🔁
