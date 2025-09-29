

1. User uploads the file → it goes into `uploads/` prefix on S3.
2. The page will then **poll S3** (via presigned URL or API call) to check if the corresponding JSON result exists in `results/`.
3. Once available → it will **download and display the JSON content** on the page.

Here’s the **updated HTML**:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>MirrorBI Upload</title>
  <script src="https://sdk.amazonaws.com/js/aws-sdk-2.1487.0.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; }
    .card { border: 1px solid #ddd; padding: 20px; border-radius: 10px; width: 400px; margin: auto; }
    .hidden { display: none; }
    #resultBox { margin-top: 20px; padding: 15px; border: 1px solid #ddd; border-radius: 5px; background: #f9f9f9; white-space: pre-wrap; }
  </style>
</head>
<body>
  <div class="card">
    <h2>Upload PDF to MirrorBI</h2>
    <input type="file" id="fileInput" />
    <button onclick="uploadFile()">Upload</button>

    <div id="status"></div>
    <div id="resultBox" class="hidden"></div>
  </div>

  <script>
    // Configure AWS SDK (You must set these properly via Cognito or IAM user with limited policy)
    AWS.config.update({
      region: "us-east-1", // change to your region
      credentials: new AWS.Credentials({
        accessKeyId: "YOUR_ACCESS_KEY",
        secretAccessKey: "YOUR_SECRET_KEY"
      })
    });

    const s3 = new AWS.S3({ apiVersion: "2006-03-01" });
    const bucketName = "your-bucket-name";
    const uploadPrefix = "uploads/";
    const resultPrefix = "results/";

    async function uploadFile() {
      const fileInput = document.getElementById("fileInput");
      if (!fileInput.files.length) {
        alert("Please select a file first.");
        return;
      }

      const file = fileInput.files[0];
      const uploadKey = uploadPrefix + file.name;
      const resultKey = resultPrefix + file.name.replace(/\.[^/.]+$/, "") + ".json";

      document.getElementById("status").innerText = "Uploading...";

      try {
        // Upload to S3
        await s3.putObject({
          Bucket: bucketName,
          Key: uploadKey,
          Body: file,
          ContentType: file.type
        }).promise();

        document.getElementById("status").innerText = "File uploaded! Processing...";

        // Poll for result
        pollForResult(resultKey);

      } catch (err) {
        console.error("Upload error", err);
        document.getElementById("status").innerText = "Error uploading file.";
      }
    }

    async function pollForResult(resultKey) {
      let attempts = 0;
      const maxAttempts = 20; // ~100 seconds total
      const interval = 5000; // 5 sec

      const check = async () => {
        attempts++;
        try {
          const data = await s3.getObject({
            Bucket: bucketName,
            Key: resultKey
          }).promise();

          const text = new TextDecoder("utf-8").decode(data.Body);
          document.getElementById("resultBox").classList.remove("hidden");
          document.getElementById("resultBox").innerText = text;
          document.getElementById("status").innerText = "Processing complete ✅";

        } catch (err) {
          if (err.code === "NoSuchKey" && attempts < maxAttempts) {
            setTimeout(check, interval);
          } else {
            document.getElementById("status").innerText = "No result found yet. Try again later.";
          }
        }
      };

      check();
    }
  </script>
</body>
</html>
```

---

⚡ **How it works**:

* Uploads file → `uploads/filename.pdf`
* Lambda runs → saves JSON → `results/filename.json`
* Page polls every 5s → fetches result → displays JSON in a box.

---


