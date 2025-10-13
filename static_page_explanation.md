explain **everything** in `mirrorBI_index.html` file — including structure, CSS, and JavaScript logic.

---

## 🧱 **HTML Structure**

```html
<!doctype html>
<html lang="en">
```

* Declares the document type as **HTML5**.
* The `lang="en"` attribute says the document language is English.

---

### `<head>` Section

```html
<head>
  <meta charset="utf-8" />
```

* Sets the character encoding to **UTF-8**, which supports all languages and symbols.

```html
  <meta name="viewport" content="width=device-width,initial-scale=1" />
```

* Makes the page **responsive** for mobile devices.
* `width=device-width` makes it fit the device screen size.
* `initial-scale=1` means zoom level is 100%.

```html
  <title>MirrorBI — Analyse</title>
```

* Sets the page title that appears in the browser tab.

---

### `<style>` Section

The `<style>` block defines **CSS styling** for layout and design.

```css
:root{
  --bg:#f6f7fb;
  --card:#ffffff;
  --accent:#0b69ff;
  --muted:#64748b;
}
```

* Defines **CSS variables** for colors (background, accent, muted text, etc.)
* These can be reused across styles using `var(--accent)`.

---

#### General styles

```css
*{box-sizing:border-box}
```

* Ensures padding and borders are included inside element width/height.

```css
body{
  font-family:Inter, ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
  background:var(--bg);
  color:#0f172a;
  margin:0;
}
```

* Sets font family.
* Gives a light gray background.
* Text color is dark gray.
* Removes default browser margins.

---

#### Header & Logo

```css
.header{
  background:var(--card);
  padding:18px 24px;
  box-shadow:0 1px 4px rgba(16,24,40,0.06);
  display:flex;
  align-items:center;
}
.logo{
  font-weight:700;
  color:var(--accent);
  letter-spacing:0.2px;
}
```

* `.header` styles the top bar (white background, shadow, padding).
* `.logo` defines the “MirrorBI” title in bold blue.

---

#### Main container

```css
.wrap{
  max-width:960px;
  margin:32px auto;
  padding:28px;
}
```

* Centers the content and limits width to 960px.

```css
.card{
  background:var(--card);
  border-radius:12px;
  padding:22px;
  box-shadow:0 6px 18px rgba(15,23,42,0.06);
}
```

* Creates a **card box** with rounded corners and subtle shadow.

---

#### Headings and Paragraphs

```css
h1{font-size:20px; margin:0 0 8px}
p.lead{color:#475569; margin:0 0 18px}
```

* Styles the title and the small description text below it.

---

#### Buttons and File Input

```css
.controls{display:flex; gap:12px; align-items:center; flex-wrap:wrap}
.file-select{display:flex; align-items:center; gap:10px}
```

* `.controls` arranges the file button and analyze button in one line.
* `.file-select` aligns the “Select file” button and file name text.

```css
.btn{
  background:var(--accent);
  color:white;
  border:0;
  padding:10px 14px;
  border-radius:8px;
  cursor:pointer;
  font-weight:600;
}
.btn.secondary{
  background:#e6eefc;
  color:var(--accent);
}
```

* `.btn` styles all buttons (blue color, rounded, bold).
* `.btn.secondary` is for the “Select file” button (light blue).

```css
input[type=file]{display:none}
```

* Hides the default file input (we trigger it manually with JS).

```css
.filename{
  font-size:13px;
  color:var(--muted);
  max-width:420px;
  overflow:hidden;
  text-overflow:ellipsis;
  white-space:nowrap;
}
```

* Shows selected file name neatly (truncates long names with “...”)

---

#### Spinner & Animation

```css
.spinner{
  border:4px solid #f3f4f6;
  border-top:4px solid var(--accent);
  border-radius:50%;
  width:28px;
  height:28px;
  animation:spin 1s linear infinite;
}
@keyframes spin{to{transform:rotate(360deg)}}
```

* Creates a rotating circle (spinner) while analyzing the file.

---

#### Results area and code block

```css
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
```

* `.results` gives space above results.
* `.code` styles raw JSON output (dark background like a console).

---

#### Footer

```css
footer{
  margin-top:14px;
  text-align:center;
  color:#94a3b8;
  font-size:13px;
}
```

* Adds footer text centered in light gray.

---

### `<body>` Content

```html
<header class="header"><div class="logo">MirrorBI</div></header>
```

* The page header with the “MirrorBI” logo.

---

#### Main Card Content

```html
<main class="wrap">
  <div class="card">
    <h1>Analyse your file</h1>
    <p class="lead">Upload a file ... and fetch the JSON result back.</p>
```

* Explains what the user should do.

---

#### File selection controls

```html
<div class="controls">
  <div class="file-select">
    <button id="selectBtn" class="btn secondary">Select file (admin style)</button>
    <input id="fileInput" type="file" accept=".pdf,.csv,.xlsx,.json" />
    <div id="fileName" class="filename">No file selected</div>
  </div>
  <button id="analyseBtn" class="btn" disabled>Analyse</button>
</div>
```

* “Select file” button (click triggers hidden file input).
* Displays selected filename.
* “Analyse” button starts processing — initially **disabled**.

---

```html
<div id="resultsArea" class="results"></div>
```

* Placeholder for showing analysis results or errors.

---

```html
<footer>...deploy this static page...</footer>
```

* Note about S3 deployment and backend setup.

---

## ⚙️ **JavaScript Logic**

This makes the page interactive.

---

### Element references

```js
const fileInput = document.getElementById('fileInput');
const selectBtn = document.getElementById('selectBtn');
const fileName = document.getElementById('fileName');
const analyseBtn = document.getElementById('analyseBtn');
const resultsArea = document.getElementById('resultsArea');
```

* Stores references to important HTML elements for later use.

---

### Open file dialog

```js
selectBtn.addEventListener('click', () => fileInput.click());
```

* When you click the “Select file” button, it triggers the hidden file input.

---

### Handle file selection

```js
fileInput.addEventListener('change', () => {
  const f = fileInput.files[0];
  if (f) {
    fileName.textContent = f.name;
    analyseBtn.disabled = false;
  } else {
    fileName.textContent = 'No file selected';
    analyseBtn.disabled = true;
  }
});
```

* When a file is chosen:

  * Shows its name next to the button.
  * Enables the “Analyse” button.

---

### Start analysis process

```js
analyseBtn.addEventListener('click', async () => {
  const f = fileInput.files[0];
  if (!f) return;

  resultsArea.innerHTML = `<div style="display:flex;gap:12px;align-items:center;">
    <div class="spinner"></div>
    <div class="small">Analysing ${escapeHtml(f.name)} — please wait...</div>
  </div>`;
  analyseBtn.disabled = true; selectBtn.disabled = true;
```

* Shows a spinner and message “Analysing filename...”.
* Disables both buttons during upload.

---

### Try to analyse the file

```js
try {
  const json = await analyseFile(f);
  resultsArea.innerHTML = renderResults(json);
} catch(err) {
  resultsArea.innerHTML = `<div class="small">Error: ${escapeHtml(err.message||err)}</div>`;
} finally {
  analyseBtn.disabled = false; selectBtn.disabled = false;
}
```

* Calls `analyseFile(f)` to upload and get results.
* If successful → display the formatted result.
* If error → show error message.
* Re-enable buttons at the end.

---

### Backend communication

```js
async function analyseFile(file) {
  // Step 1: ask backend for pre-signed URLs
  const presignRes = await fetch('/get-presigned-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ filename: file.name })
  });
  const { uploadUrl, resultUrl } = await presignRes.json();
```

* Sends the filename to backend endpoint `/get-presigned-url`.
* Backend should return two URLs:

  * `uploadUrl`: where to upload the file.
  * `resultUrl`: where the JSON result will appear later.

---

```js
  // Step 2: upload file to S3
  await fetch(uploadUrl, {
    method: 'PUT',
    body: file
  });
```

* Uploads the file directly to S3 using the presigned URL.

---

```js
  // Step 3: poll for result JSON
  let json = null;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 3000));
    const res = await fetch(resultUrl);
    if (res.ok) {
      json = await res.json();
      break;
    }
  }
  if (!json) throw new Error("Result not ready in time.");
  return json;
}
```

* Waits up to **60 seconds (20 × 3s)** checking S3 for the result JSON.
* If it appears, returns it; otherwise throws an error.

---

### Render analysis result

```js
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
```

* Displays the analysis summary.
* Adds a collapsible `<details>` section to view raw JSON.

---

### Escape HTML (for security)

```js
function escapeHtml(s){
  return String(s).replace(/[&<>"']/g, c => ({
    '&':'&amp;',
    '<':'&lt;',
    '>':'&gt;',
    '"':'&quot;',
    '\'':'&#39;'
  })[c]);
}
```

* Prevents **HTML injection** by converting special characters to safe codes.

---

✅ **In short:**
This page lets you:

1. Select a file (PDF, CSV, XLSX, JSON).
2. Upload it to S3 via backend-signed URL.
3. Wait for a Bedrock/Lambda process to analyze it.
4. Display the analysis results in a user-friendly card.

---


