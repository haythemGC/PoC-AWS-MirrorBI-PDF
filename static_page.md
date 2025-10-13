

1. User uploads the file → it goes into `uploads/` prefix on S3.
2. The page will then **poll S3** (via presigned URL or API call) to check if the corresponding JSON result exists in `results/`.
3. Once available → it will **download and display the JSON content** on the page.

Here’s the **updated HTML**:

```html
<!doctype html>
<!-- Declares the document type as HTML5 -->
<html lang="en"> <!-- Sets the language of the page to English -->
<head>
  <meta charset="utf-8" /> <!-- Ensures characters are displayed correctly -->
  <meta name="viewport" content="width=device-width,initial-scale=1" /> <!-- Makes the layout responsive -->
  <title>MirrorBI — Analyse</title> <!-- Browser tab title -->

  <!-- CSS Styles -->
  <style>
    /* Define reusable color variables */
    :root{--bg:#f6f7fb;--card:#ffffff;--accent:#0b69ff;--muted:#64748b}

    /* Make padding/border count inside the width */
    *{box-sizing:border-box}

    /* Global body styling */
    body{
      font-family:Inter, ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
      background:var(--bg);
      color:#0f172a;
      margin:0;
    }

    /* Header (top bar) */
    .header{
      background:var(--card);
      padding:18px 24px;
      box-shadow:0 1px 4px rgba(16,24,40,0.06);
      display:flex;
      align-items:center;
    }

    /* Logo text styling */
    .logo{
      font-weight:700;
      color:var(--accent);
      letter-spacing:0.2px;
    }

    /* Main container wrapper */
    .wrap{max-width:960px; margin:32px auto; padding:28px;}

    /* Card-style box for content */
    .card{
      background:var(--card);
      border-radius:12px;
      padding:22px;
      box-shadow:0 6px 18px rgba(15,23,42,0.06);
    }

    /* Title and paragraph styles */
    h1{font-size:20px; margin:0 0 8px}
    p.lead{color:#475569; margin:0 0 18px}

    /* Layout for controls */
    .controls{display:flex; gap:12px; align-items:center; flex-wrap:wrap}
    .file-select{display:flex; align-items:center; gap:10px}

    /* Button styles */
    .btn{
      background:var(--accent);
      color:white;
      border:0;
      padding:10px 14px;
      border-radius:8px;
      cursor:pointer;
      font-weight:600;
    }
    .btn.secondary{background:#e6eefc; color:var(--accent)}

    /* Hide file input (we trigger it with JS) */
    input[type=file]{display:none}

    /* Selected filename display */
    .filename{
      font-size:13px;
      color:var(--muted);
      max-width:420px;
      overflow:hidden;
      text-overflow:ellipsis;
      white-space:nowrap;
    }

    /* Spinner animation during analysis */
    .spinner{
      border:4px solid #f3f4f6;
      border-top:4px solid var(--accent);
      border-radius:50%;
      width:28px;
      height:28px;
      animation:spin 1s linear infinite;
    }
    @keyframes spin{to{transform:rotate(360deg)}}

    /* Results area and code styling */
    .results{margin-top:18px}
    .small{font-size:13px}
    .code{
      background:#0b1220;
      color:#e6eefc;
      padding:12px;
      border-radius:8px;
      overflow:auto;
      font-family:monospace;
      font-size:13px;
      max-height:320px;
    }

    /* Footer styling */
    footer{margin-top:14px; text-align:center; color:#94a3b8; font-size:13px}
  </style>
</head>

<body>
  <!-- Top header -->
  <header class="header"><div class="logo">MirrorBI</div></header>

  <!-- Main section -->
  <main class="wrap">
    <div class="card">
      <h1>Analyse your file</h1>
      <p class="lead">Upload a file, then click <strong>Analyse</strong>. This will send the file to S3, trigger Lambda + Bedrock, and fetch the JSON result back.</p>

      <!-- File upload controls -->
      <div class="controls">
        <div class="file-select">
          <button id="selectBtn" class="btn secondary">Select file (admin style)</button>
          <input id="fileInput" type="file" accept=".pdf,.csv,.xlsx,.json" /> <!-- Hidden input -->
          <div id="fileName" class="filename">No file selected</div>
        </div>
        <button id="analyseBtn" class="btn" disabled>Analyse</button>
      </div>

      <!-- Area where results or errors will appear -->
      <div id="resultsArea" class="results"></div>

      <footer>You can deploy this static page to an S3 bucket. Ensure your backend provides pre-signed URLs for upload and result retrieval.</footer>
    </div>
  </main>

  <!-- JavaScript for interactivity -->
  <script>
    // Get HTML elements by ID
    const fileInput = document.getElementById('fileInput');
    const selectBtn = document.getElementById('selectBtn');
    const fileName = document.getElementById('fileName');
    const analyseBtn = document.getElementById('analyseBtn');
    const resultsArea = document.getElementById('resultsArea');

    // When user clicks "Select file" button, open file chooser
    selectBtn.addEventListener('click', () => fileInput.click());

    // Handle file selection
    fileInput.addEventListener('change', () => {
      const f = fileInput.files[0];
      if (f) {
        fileName.textContent = f.name; // Show chosen file name
        analyseBtn.disabled = false;   // Enable analyse button
      } else {
        fileName.textContent = 'No file selected';
        analyseBtn.disabled = true;    // Disable if no file
      }
    });

    // When user clicks "Analyse"
    analyseBtn.addEventListener('click', async () => {
      const f = fileInput.files[0];
      if (!f) return;

      // Show spinner while processing
      resultsArea.innerHTML = `<div style="display:flex;gap:12px;align-items:center;"><div class="spinner"></div><div class="small">Analysing ${escapeHtml(f.name)} — please wait...</div></div>`;
      analyseBtn.disabled = true; selectBtn.disabled = true;

      try {
        // Upload and get result
        const json = await analyseFile(f);
        resultsArea.innerHTML = renderResults(json); // Display formatted results
      } catch(err) {
        // Handle any errors
        resultsArea.innerHTML = `<div class="small">Error: ${escapeHtml(err.message||err)}</div>`;
      } finally {
        // Re-enable buttons
        analyseBtn.disabled = false; selectBtn.disabled = false;
      }
    });

    // Function that handles the entire upload + analysis process
    async function analyseFile(file) {
      // Step 1: Get presigned URLs from backend
      const presignRes = await fetch('/get-presigned-url', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ filename: file.name })
      });
      const { uploadUrl, resultUrl } = await presignRes.json();

      // Step 2: Upload file directly to S3
      await fetch(uploadUrl, {
        method: 'PUT',
        body: file
      });

      // Step 3: Poll S3 for the result file every 3s (up to 20 times)
      let json = null;
      for (let i = 0; i < 20; i++) {
        await new Promise(r => setTimeout(r, 3000)); // wait 3 seconds
        const res = await fetch(resultUrl);
        if (res.ok) {
          json = await res.json();
          break; // stop polling once ready
        }
      }

      // Throw error if no result found in time
      if (!json) throw new Error("Result not ready in time.");
      return json;
    }

    // Render analysis result in a styled box
    function renderResults(json){
      return `
        <div style="background:var(--card); padding:14px; border-radius:10px">
          <h3 class="small">Analysis — ${escapeHtml(json.fileName)}</h3>
          <p class="small">${escapeHtml(json.summary)}</p>
          <details style="margin-top:12px">
            <summary class="small">Show raw JSON</summary>
            <pre class="code">${escapeHtml(JSON.stringify(json,null,2))}</pre>
          </details>
        </div>`;
    }

    // Helper to safely display text (avoid HTML injection)
    function escapeHtml(s){
      return String(s).replace(/[&<>"']/g, c => ({
        '&':'&amp;',
        '<':'&lt;',
        '>':'&gt;',
        '"':'&quot;',
        '\'':'&#39;'
      })[c]);
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


