# Build the Speech Evaluator Prototype on Windows

This guide walks a student through building the same prototype in this folder from scratch on a Windows machine.

The app does this:

1. Lets a user upload a speech video.
2. Sends that video to Gemini through a Python backend.
3. Returns judging-style feedback on argument, delivery, elocution, gestures, filler words, and an overall score.

This guide is written so a student can do it with or without Codex. If needed, they can simply copy-paste the code blocks into the right files.

## Before you begin

Before starting this guide, make sure you already know how to:

1. Open PowerShell.
2. Run `git init`.
3. Run `uv init --bare`.
4. Set `GEMINI_API_KEY` on Windows.

If you need those steps first, use:

[00-WINDOWS-PROJECT-SETUP.md](./00-WINDOWS-PROJECT-SETUP.md)

## Step 1: Create the project folder

In PowerShell:

```powershell
cd $HOME\Desktop
mkdir speech-eval-prototype
cd speech-eval-prototype
```

## Step 2: Initialize Git

```powershell
git init
```

## Step 3: Initialize the Python project with `uv`

```powershell
uv init --bare
```

If needed, install or pin Python:

```powershell
uv python install 3.12
uv python pin 3.12
```

## Step 4: Add the dependencies

**Dependencies** are packages (other people's code) that your project needs in order to work. Here is what each one does:

- **fastapi** — a Python framework for building web APIs. It handles incoming web requests (like when the browser sends a video to be evaluated) and sends back responses.
- **uvicorn** — a lightweight web server that actually runs your FastAPI app and listens for requests on your computer.
- **python-multipart** — lets FastAPI accept file uploads (like video files) from a web browser.
- **google-genai** — Google's official Python library for talking to the Gemini API. This is how your backend sends the video to Gemini and gets evaluation feedback back.

```powershell
uv add fastapi uvicorn python-multipart google-genai
```

## Step 5: Create the folder structure

This app has two main parts. The **backend** (`app.py`) is the Python code that runs on your computer and does the real work: receiving the video, sending it to Gemini, and returning the results. The **frontend** (`static/index.html`) is the web page that runs in the browser and gives the user something to look at and click on. The frontend talks to the backend behind the scenes. This frontend/backend split is how most modern web apps work.

Create a `static` folder:

```powershell
mkdir static
```

At this point, your project should contain:

1. `pyproject.toml`
2. `uv.lock`
3. `app.py`
4. `static\index.html`
5. `.gitignore`

You will create `app.py`, `static\index.html`, and `.gitignore` next.

## Step 6: Create `.gitignore`

Create a file named `.gitignore` and paste this:

```gitignore
.venv/
__pycache__/
.env
```

## Step 7: Create `app.py`

This is the backend — the Python code that does all the heavy lifting. You do not need to understand every line right now. The important thing to know is that this file receives a video upload from the browser, sends it to Gemini through the API, and returns structured feedback as JSON (a standard format for passing data between programs).

Create a file named `app.py` in the project root and paste this:

```python
"""
Speech Evaluation Prototype
Upload a video of a speech -> get AI feedback on argument, delivery, elocution, and gestures.
"""

import asyncio
import json
import os
import tempfile
from pathlib import Path

from fastapi import FastAPI, File, Header, HTTPException, UploadFile, status
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from google import genai
from google.genai import types
from pydantic import BaseModel, Field, ValidationError

APP_DIR = Path(__file__).resolve().parent
STATIC_DIR = APP_DIR / "static"
INDEX_FILE = STATIC_DIR / "index.html"

GEMINI_MODEL = os.environ.get("GEMINI_MODEL", "gemini-2.5-flash")
GEMINI_TEMPERATURE = float(os.environ.get("GEMINI_TEMPERATURE", "0.3"))
GEMINI_THINKING_BUDGET = int(os.environ.get("GEMINI_THINKING_BUDGET", "0"))
MAX_UPLOAD_MB = int(os.environ.get("MAX_UPLOAD_MB", "100"))
MAX_UPLOAD_BYTES = MAX_UPLOAD_MB * 1024 * 1024
UPLOAD_CHUNK_SIZE = int(os.environ.get("UPLOAD_CHUNK_SIZE_BYTES", str(1024 * 1024)))
APP_PASSWORD = os.environ.get("APP_PASSWORD", "").strip()

app = FastAPI(title="Speech Evaluator")
app.mount("/static", StaticFiles(directory=str(STATIC_DIR)), name="static")

EVALUATION_PROMPT = """You are an expert speech and debate coach evaluating a video recording of a speaker.

Analyze BOTH the audio and the visual content of this video. Pay close attention to what you can SEE (body language, gestures, posture, eye contact, facial expressions, movement) as well as what you can HEAR (voice, pacing, filler words, clarity).

Return your evaluation as a JSON object with exactly this structure (no markdown, no code fences, just raw JSON):

{
  "summary": "A 2-3 sentence overall assessment of the speech.",
  "argument": {
    "score": <1-10>,
    "strengths": ["strength 1", "strength 2"],
    "improvements": ["improvement 1", "improvement 2"],
    "details": "A paragraph analyzing argument structure, evidence usage, persuasiveness, logical flow, and transitions."
  },
  "delivery": {
    "score": <1-10>,
    "strengths": ["strength 1", "strength 2"],
    "improvements": ["improvement 1", "improvement 2"],
    "details": "A paragraph analyzing pacing, filler words (list any detected with approximate timestamps), vocal variety, volume, confidence, and energy."
  },
  "elocution": {
    "score": <1-10>,
    "strengths": ["strength 1", "strength 2"],
    "improvements": ["improvement 1", "improvement 2"],
    "details": "A paragraph analyzing clarity of speech, pronunciation, articulation, enunciation, and word choice."
  },
  "gestures": {
    "score": <1-10>,
    "strengths": ["strength 1", "strength 2"],
    "improvements": ["improvement 1", "improvement 2"],
    "details": "A paragraph analyzing hand gestures, body language, posture, facial expressions, eye contact, and physical presence. Reference specific moments in the video where gestures helped or hindered the message."
  },
  "top_three": [
    "The single most impactful thing the speaker should work on, with a specific example from the video.",
    "The second most important improvement, with a specific example.",
    "The third most important improvement, with a specific example."
  ],
  "filler_words": {
    "count": <number>,
    "instances": ["~0:15 'um'", "~0:42 'like'"]
  },
  "overall_score": <1-10>
}

Be constructive but honest. Reference specific timestamps where possible. If you cannot see the speaker clearly in the video, note that in the gestures section and evaluate based on what is visible."""


class EvaluationCategory(BaseModel):
    score: int = Field(ge=1, le=10)
    strengths: list[str]
    improvements: list[str]
    details: str


class FillerWords(BaseModel):
    count: int = Field(ge=0)
    instances: list[str]


class EvaluationResult(BaseModel):
    summary: str
    argument: EvaluationCategory
    delivery: EvaluationCategory
    elocution: EvaluationCategory
    gestures: EvaluationCategory
    top_three: list[str]
    filler_words: FillerWords
    overall_score: int = Field(ge=1, le=10)


def get_client() -> genai.Client:
    api_key = os.environ.get("GEMINI_API_KEY")
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="GEMINI_API_KEY environment variable not set",
        )
    return genai.Client(api_key=api_key)


def require_password(x_app_password: str | None) -> None:
    if APP_PASSWORD and x_app_password != APP_PASSWORD:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid tester password",
        )


def normalize_file_state(state: object) -> str:
    if isinstance(state, str):
        return state
    name = getattr(state, "name", None)
    if isinstance(name, str):
        return name
    return str(state).split(".")[-1]


def parse_response_json(raw_text: str) -> dict:
    text = raw_text.strip()
    if text.startswith("```"):
        lines = text.splitlines()
        if lines:
            lines = lines[1:]
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]
        text = "\n".join(lines).strip()

    if not text:
        raise ValueError("Gemini returned an empty response")

    return json.loads(text)


async def save_upload_to_tempfile(video: UploadFile) -> str:
    suffix = os.path.splitext(video.filename or "video.mp4")[1] or ".mp4"
    bytes_written = 0

    with tempfile.NamedTemporaryFile(delete=False, suffix=suffix) as tmp:
        tmp_path = tmp.name
        try:
            while True:
                chunk = await video.read(UPLOAD_CHUNK_SIZE)
                if not chunk:
                    break

                bytes_written += len(chunk)
                if bytes_written > MAX_UPLOAD_BYTES:
                    raise HTTPException(
                        status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
                        detail=f"Video exceeds the {MAX_UPLOAD_MB} MB upload limit",
                    )

                tmp.write(chunk)
        except Exception:
            if os.path.exists(tmp_path):
                os.unlink(tmp_path)
            raise
        finally:
            await video.close()

    return tmp_path


@app.get("/", response_class=HTMLResponse)
async def index() -> str:
    return INDEX_FILE.read_text(encoding="utf-8")


@app.get("/healthz")
async def healthz() -> dict[str, str]:
    return {"status": "ok"}


@app.get("/config")
async def config() -> dict[str, object]:
    return {
        "auth_required": bool(APP_PASSWORD),
        "max_upload_mb": MAX_UPLOAD_MB,
        "model": GEMINI_MODEL,
    }


@app.post("/evaluate", response_model=EvaluationResult)
async def evaluate_speech(
    video: UploadFile = File(...),
    x_app_password: str | None = Header(default=None),
):
    require_password(x_app_password)

    if not video.content_type or not video.content_type.startswith("video/"):
        raise HTTPException(status_code=400, detail="File must be a video")

    client = get_client()
    tmp_path = await save_upload_to_tempfile(video)
    uploaded_file = None

    try:
        uploaded_file = client.files.upload(
            file=tmp_path,
            config=types.UploadFileConfig(mime_type=video.content_type),
        )

        while normalize_file_state(uploaded_file.state) == "PROCESSING":
            await asyncio.sleep(1)
            uploaded_file = client.files.get(name=uploaded_file.name)

        upload_state = normalize_file_state(uploaded_file.state)
        if upload_state != "ACTIVE":
            raise HTTPException(
                status_code=status.HTTP_502_BAD_GATEWAY,
                detail=f"Gemini could not process the video file (state: {upload_state})",
            )

        config_kwargs = {
            "temperature": GEMINI_TEMPERATURE,
            "response_mime_type": "application/json",
        }
        if GEMINI_THINKING_BUDGET >= 0:
            config_kwargs["thinking_config"] = types.ThinkingConfig(
                thinkingBudget=GEMINI_THINKING_BUDGET
            )

        response = client.models.generate_content(
            model=GEMINI_MODEL,
            contents=[
                types.Content(
                    role="user",
                    parts=[
                        types.Part.from_uri(
                            file_uri=uploaded_file.uri,
                            mime_type=video.content_type,
                        ),
                        types.Part.from_text(text=EVALUATION_PROMPT),
                    ],
                ),
            ],
            config=types.GenerateContentConfig(**config_kwargs),
        )

        raw_text = response.text or ""
        parsed = parse_response_json(raw_text)
        result = EvaluationResult.model_validate(parsed)
        return result.model_dump()

    except HTTPException:
        raise
    except (json.JSONDecodeError, ValidationError, ValueError) as exc:
        raise HTTPException(
            status_code=status.HTTP_502_BAD_GATEWAY,
            detail="Gemini returned an invalid evaluation format. Please try again.",
        ) from exc
    except Exception as exc:
        raise HTTPException(
            status_code=status.HTTP_502_BAD_GATEWAY,
            detail="Gemini request failed. Please try again.",
        ) from exc
    finally:
        if os.path.exists(tmp_path):
            os.unlink(tmp_path)
        if uploaded_file is not None:
            try:
                client.files.delete(name=uploaded_file.name)
            except Exception:
                pass


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="127.0.0.1", port=8000)
```

## Step 8: Create `static\index.html`

This is the frontend — the web page that users actually see and interact with. It includes HTML (the structure of the page), CSS (the visual styling), and JavaScript (the code that handles user interactions like uploading a file and displaying results). Everything is in a single file for simplicity.

Create a file named `index.html` inside the `static` folder and paste this:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Speech Evaluator</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: #0f1117;
      color: #e0e0e0;
      min-height: 100vh;
    }

    .container {
      max-width: 900px;
      margin: 0 auto;
      padding: 2rem 1.5rem;
    }

    h1 {
      font-size: 1.8rem;
      font-weight: 600;
      margin-bottom: 0.3rem;
      color: #fff;
    }

    .subtitle {
      color: #888;
      margin-bottom: 1rem;
      font-size: 0.95rem;
    }

    .settings-card {
      background: #181a20;
      border: 1px solid #242833;
      border-radius: 12px;
      padding: 1rem;
      margin-bottom: 1.5rem;
    }

    .field {
      margin-bottom: 0.75rem;
    }

    .field:last-child {
      margin-bottom: 0;
    }

    .field label {
      display: block;
      font-size: 0.85rem;
      font-weight: 600;
      color: #d7dbe7;
      margin-bottom: 0.4rem;
    }

    .field input {
      width: 100%;
      padding: 0.8rem 0.9rem;
      border: 1px solid #303543;
      border-radius: 8px;
      background: #10131a;
      color: #f3f5fa;
      font-size: 0.95rem;
    }

    .field input:focus {
      outline: 2px solid rgba(79, 140, 255, 0.35);
      border-color: #4f8cff;
    }

    .hint {
      color: #8f96a8;
      font-size: 0.85rem;
      line-height: 1.5;
    }

    .upload-area {
      border: 2px dashed #333;
      border-radius: 12px;
      padding: 3rem 2rem;
      text-align: center;
      cursor: pointer;
      transition: border-color 0.2s, background 0.2s;
      margin-bottom: 1.5rem;
    }

    .upload-area:hover, .upload-area.dragover {
      border-color: #4f8cff;
      background: rgba(79, 140, 255, 0.05);
    }

    .upload-icon {
      font-size: 2.5rem;
      margin-bottom: 0.8rem;
      display: block;
    }

    .upload-text {
      color: #aaa;
      font-size: 0.95rem;
    }

    .upload-text strong {
      color: #4f8cff;
    }

    input[type="file"] { display: none; }

    .preview-container {
      display: none;
      margin-bottom: 1rem;
    }

    .preview-container video {
      width: 100%;
      max-height: 400px;
      border-radius: 8px;
      background: #000;
    }

    .file-info {
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: 1rem;
      padding: 0.5rem 0;
      font-size: 0.85rem;
      color: #888;
    }

    .remove-btn {
      background: none;
      border: none;
      color: #e74c3c;
      cursor: pointer;
      font-size: 0.85rem;
    }

    .evaluate-btn {
      display: none;
      width: 100%;
      padding: 1rem;
      font-size: 1.1rem;
      font-weight: 600;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      background: #4f8cff;
      color: #fff;
      transition: background 0.2s;
      margin-bottom: 2rem;
    }

    .evaluate-btn:hover { background: #3a7ae8; }

    .evaluate-btn:disabled {
      background: #333;
      color: #666;
      cursor: not-allowed;
    }

    .loading {
      display: none;
      text-align: center;
      padding: 3rem 1rem;
      margin-bottom: 2rem;
    }

    .spinner {
      width: 40px;
      height: 40px;
      border: 3px solid #333;
      border-top-color: #4f8cff;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
      margin: 0 auto 1rem;
    }

    @keyframes spin { to { transform: rotate(360deg); } }

    .loading-text {
      color: #888;
      font-size: 0.95rem;
    }

    .error {
      display: none;
      background: rgba(231, 76, 60, 0.1);
      border: 1px solid #e74c3c;
      border-radius: 8px;
      padding: 1rem 1.2rem;
      margin-bottom: 1.5rem;
      color: #e74c3c;
      white-space: pre-wrap;
    }

    .results {
      display: none;
    }

    .overall {
      text-align: center;
      padding: 2rem;
      background: #181a20;
      border-radius: 12px;
      margin-bottom: 1.5rem;
    }

    .overall-score {
      font-size: 3.5rem;
      font-weight: 700;
      color: #4f8cff;
    }

    .overall-score span {
      font-size: 1.5rem;
      color: #555;
      font-weight: 400;
    }

    .overall-summary {
      color: #bbb;
      margin-top: 0.8rem;
      line-height: 1.6;
      font-size: 0.95rem;
    }

    .category {
      background: #181a20;
      border-radius: 12px;
      padding: 1.5rem;
      margin-bottom: 1rem;
    }

    .category-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 100%;
      background: none;
      border: none;
      color: inherit;
      margin-bottom: 1rem;
      cursor: pointer;
      padding: 0;
      text-align: left;
    }

    .category-title {
      font-size: 1.15rem;
      font-weight: 600;
      color: #fff;
    }

    .category-score {
      font-size: 1.3rem;
      font-weight: 700;
      min-width: 3rem;
      text-align: right;
    }

    .category-body {
      display: none;
    }

    .category.open .category-body {
      display: block;
    }

    .category-details {
      color: #bbb;
      line-height: 1.7;
      margin-bottom: 1rem;
      font-size: 0.92rem;
    }

    .list-section {
      margin-bottom: 0.8rem;
    }

    .list-section h4 {
      font-size: 0.8rem;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      margin-bottom: 0.4rem;
    }

    .list-section.strengths h4 { color: #2ecc71; }
    .list-section.improvements h4 { color: #f39c12; }

    .list-section ul {
      list-style: none;
      padding: 0;
    }

    .list-section li {
      padding: 0.3rem 0 0.3rem 1.2rem;
      position: relative;
      font-size: 0.9rem;
      color: #ccc;
    }

    .list-section.strengths li::before {
      content: "+";
      position: absolute;
      left: 0;
      color: #2ecc71;
      font-weight: 700;
    }

    .list-section.improvements li::before {
      content: "!";
      position: absolute;
      left: 0.15rem;
      color: #f39c12;
      font-weight: 700;
    }

    .top-three,
    .filler-words {
      background: #181a20;
      border-radius: 12px;
      padding: 1.5rem;
      margin-bottom: 1rem;
    }

    .top-three h3,
    .filler-words h3 {
      font-size: 1.15rem;
      font-weight: 600;
      color: #fff;
      margin-bottom: 0.8rem;
    }

    .top-three ol {
      padding-left: 1.5rem;
    }

    .top-three li {
      padding: 0.5rem 0;
      line-height: 1.6;
      font-size: 0.92rem;
      color: #ccc;
    }

    .filler-count {
      font-size: 2rem;
      font-weight: 700;
      color: #f39c12;
      margin-bottom: 0.5rem;
    }

    .filler-list {
      color: #999;
      font-size: 0.9rem;
      line-height: 1.6;
    }

    .score-high { color: #2ecc71; }
    .score-mid { color: #f39c12; }
    .score-low { color: #e74c3c; }

    .reset-btn {
      display: block;
      width: 100%;
      padding: 0.8rem;
      margin-top: 1.5rem;
      font-size: 1rem;
      background: #222;
      color: #aaa;
      border: 1px solid #333;
      border-radius: 8px;
      cursor: pointer;
    }

    .reset-btn:hover {
      background: #2a2a2a;
      color: #fff;
    }

    @media (max-width: 640px) {
      .container {
        padding: 1.2rem 1rem 2rem;
      }

      .file-info {
        flex-direction: column;
        align-items: flex-start;
      }

      .overall-score {
        font-size: 3rem;
      }
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Speech Evaluator</h1>
    <p class="subtitle">Upload a short speech video to get AI feedback on argument, delivery, elocution, and gestures.</p>

    <div class="settings-card">
      <div class="field">
        <label for="passwordInput">Tester password</label>
        <input type="password" id="passwordInput" autocomplete="current-password" placeholder="Only needed if the deployed app is password-protected">
      </div>
      <p class="hint" id="uploadHint">Keep test uploads short so feedback comes back quickly.</p>
    </div>

    <div class="upload-area" id="uploadArea">
      <span class="upload-icon">&#127909;</span>
      <p class="upload-text"><strong>Click to upload</strong> or drag and drop a video file</p>
      <p class="upload-text" id="maxSizeText" style="font-size:0.8rem; margin-top:0.4rem;">Video files only</p>
    </div>

    <input type="file" id="fileInput" accept="video/*">

    <div class="preview-container" id="previewContainer">
      <video id="videoPreview" controls></video>
      <div class="file-info">
        <span id="fileName"></span>
        <button class="remove-btn" id="removeBtn" type="button">Remove</button>
      </div>
    </div>

    <button class="evaluate-btn" id="evaluateBtn" type="button">Evaluate Speech</button>

    <div class="loading" id="loading">
      <div class="spinner"></div>
      <p class="loading-text">Uploading video and analyzing speech...<br>This may take 1-2 minutes depending on video length.</p>
    </div>

    <div class="error" id="error"></div>

    <div class="results" id="results">
      <div class="overall">
        <div class="overall-score" id="overallScore"></div>
        <p class="overall-summary" id="overallSummary"></p>
      </div>

      <div id="categoriesContainer"></div>

      <div class="filler-words" id="fillerWords">
        <h3>Filler Words</h3>
        <div class="filler-count" id="fillerCount"></div>
        <div class="filler-list" id="fillerList"></div>
      </div>

      <div class="top-three" id="topThree">
        <h3>Top 3 Things to Work On</h3>
        <ol id="topThreeList"></ol>
      </div>

      <button class="reset-btn" id="resetBtn" type="button">Evaluate Another Video</button>
    </div>
  </div>

  <script>
    const uploadArea = document.getElementById('uploadArea');
    const fileInput = document.getElementById('fileInput');
    const previewContainer = document.getElementById('previewContainer');
    const videoPreview = document.getElementById('videoPreview');
    const fileName = document.getElementById('fileName');
    const removeBtn = document.getElementById('removeBtn');
    const evaluateBtn = document.getElementById('evaluateBtn');
    const loading = document.getElementById('loading');
    const error = document.getElementById('error');
    const results = document.getElementById('results');
    const resetBtn = document.getElementById('resetBtn');
    const passwordInput = document.getElementById('passwordInput');
    const uploadHint = document.getElementById('uploadHint');
    const maxSizeText = document.getElementById('maxSizeText');

    let selectedFile = null;
    let appConfig = {
      auth_required: false,
      max_upload_mb: 100,
      model: 'gemini-2.5-flash',
    };

    function createNode(tag, className, text) {
      const node = document.createElement(tag);
      if (className) node.className = className;
      if (typeof text === 'string') node.textContent = text;
      return node;
    }

    function setUploadMessaging() {
      maxSizeText.textContent = `Video files only - up to ${appConfig.max_upload_mb} MB per upload`;
      uploadHint.textContent =
        `This app uses ${appConfig.model}. Keep videos short for cheaper, faster test runs.`;
      passwordInput.placeholder = appConfig.auth_required
        ? 'Enter the shared tester password'
        : 'Optional locally; deployed app may require it';
    }

    async function loadConfig() {
      try {
        const res = await fetch('/config');
        if (!res.ok) return;
        const data = await res.json();
        appConfig = { ...appConfig, ...data };
      } catch (err) {
        console.warn('Could not load app config', err);
      } finally {
        setUploadMessaging();
      }
    }

    function showError(message) {
      error.textContent = message;
      error.style.display = 'block';
    }

    function clearError() {
      error.textContent = '';
      error.style.display = 'none';
    }

    function scoreClass(score) {
      if (score >= 7) return 'score-high';
      if (score >= 5) return 'score-mid';
      return 'score-low';
    }

    function resetSelection() {
      selectedFile = null;
      videoPreview.src = '';
      fileInput.value = '';
      uploadArea.style.display = 'block';
      previewContainer.style.display = 'none';
      evaluateBtn.style.display = 'none';
    }

    function handleFile(file) {
      if (!file.type.startsWith('video/')) {
        showError('Please select a video file.');
        return;
      }

      if (file.size > appConfig.max_upload_mb * 1048576) {
        showError(`This video is too large. The current upload limit is ${appConfig.max_upload_mb} MB.`);
        return;
      }

      clearError();
      selectedFile = file;
      videoPreview.src = URL.createObjectURL(file);
      fileName.textContent = `${file.name} (${(file.size / 1048576).toFixed(1)} MB)`;
      uploadArea.style.display = 'none';
      previewContainer.style.display = 'block';
      evaluateBtn.style.display = 'block';
    }

    function renderListSection(title, items, type) {
      const section = createNode('div', `list-section ${type}`);
      section.appendChild(createNode('h4', '', title));

      const list = createNode('ul');
      for (const item of items || []) {
        list.appendChild(createNode('li', '', item));
      }
      section.appendChild(list);
      return section;
    }

    function renderCategory(label, data) {
      const card = createNode('div', 'category open');
      const header = createNode('button', 'category-header');
      header.type = 'button';

      header.appendChild(createNode('span', 'category-title', label));
      header.appendChild(createNode('span', `category-score ${scoreClass(data.score)}`, `${data.score}/10`));

      const body = createNode('div', 'category-body');
      body.appendChild(createNode('p', 'category-details', data.details));
      body.appendChild(renderListSection('Strengths', data.strengths, 'strengths'));
      body.appendChild(renderListSection('Areas for Improvement', data.improvements, 'improvements'));

      header.addEventListener('click', () => {
        card.classList.toggle('open');
      });

      card.appendChild(header);
      card.appendChild(body);
      return card;
    }

    function renderResults(data) {
      const overallScore = document.getElementById('overallScore');
      overallScore.replaceChildren();

      const scoreValue = createNode('span', scoreClass(data.overall_score), String(data.overall_score));
      const scoreSuffix = createNode('span', '', '/10');
      overallScore.appendChild(scoreValue);
      overallScore.appendChild(scoreSuffix);

      document.getElementById('overallSummary').textContent = data.summary;

      const categories = [
        { key: 'argument', label: 'Argument' },
        { key: 'delivery', label: 'Delivery' },
        { key: 'elocution', label: 'Elocution' },
        { key: 'gestures', label: 'Gestures' },
      ];

      const container = document.getElementById('categoriesContainer');
      container.replaceChildren();
      for (const category of categories) {
        if (data[category.key]) {
          container.appendChild(renderCategory(category.label, data[category.key]));
        }
      }

      const fillerWords = data.filler_words || { count: 0, instances: [] };
      document.getElementById('fillerCount').textContent = `${fillerWords.count} detected`;
      document.getElementById('fillerList').textContent =
        fillerWords.instances.length ? fillerWords.instances.join(' · ') : 'None detected';

      const topThreeList = document.getElementById('topThreeList');
      topThreeList.replaceChildren();
      for (const item of data.top_three || []) {
        topThreeList.appendChild(createNode('li', '', item));
      }

      results.style.display = 'block';
      results.scrollIntoView({ behavior: 'smooth' });
    }

    uploadArea.addEventListener('click', () => fileInput.click());

    uploadArea.addEventListener('dragover', (event) => {
      event.preventDefault();
      uploadArea.classList.add('dragover');
    });

    uploadArea.addEventListener('dragleave', () => {
      uploadArea.classList.remove('dragover');
    });

    uploadArea.addEventListener('drop', (event) => {
      event.preventDefault();
      uploadArea.classList.remove('dragover');
      if (event.dataTransfer.files.length) {
        handleFile(event.dataTransfer.files[0]);
      }
    });

    fileInput.addEventListener('change', () => {
      if (fileInput.files.length) {
        handleFile(fileInput.files[0]);
      }
    });

    removeBtn.addEventListener('click', () => {
      resetSelection();
      clearError();
    });

    evaluateBtn.addEventListener('click', async () => {
      if (!selectedFile) return;

      if (appConfig.auth_required && !passwordInput.value.trim()) {
        showError('Enter the tester password before running an evaluation.');
        return;
      }

      evaluateBtn.disabled = true;
      evaluateBtn.textContent = 'Evaluating...';
      loading.style.display = 'block';
      clearError();
      results.style.display = 'none';

      const formData = new FormData();
      formData.append('video', selectedFile);

      const headers = {};
      if (passwordInput.value.trim()) {
        headers['X-App-Password'] = passwordInput.value.trim();
      }

      try {
        const res = await fetch('/evaluate', {
          method: 'POST',
          body: formData,
          headers,
        });

        const payload = await res.json();
        if (!res.ok) {
          throw new Error(payload.detail || 'Evaluation failed');
        }

        renderResults(payload);
      } catch (err) {
        showError(err.message || 'Evaluation failed');
      } finally {
        loading.style.display = 'none';
        evaluateBtn.disabled = false;
        evaluateBtn.textContent = 'Evaluate Speech';
      }
    });

    resetBtn.addEventListener('click', () => {
      results.style.display = 'none';
      resetSelection();
      clearError();
      window.scrollTo({ top: 0, behavior: 'smooth' });
    });

    loadConfig();
  </script>
</body>
</html>
```

## Step 9: Set your API key

If you have not already done it in this PowerShell window:

```powershell
$env:GEMINI_API_KEY="paste-your-key-here"
```

Do not hard-code the key into `app.py` or `index.html`.

## Step 10: Run the app

Start the local server with:

```powershell
uv run uvicorn app:app --reload
```

Breaking this command down: `uv run` means "run this command using the project's Python environment." `uvicorn` is the web server. `app:app` tells uvicorn to look in the file `app.py` for the FastAPI application object named `app`. `--reload` tells uvicorn to automatically restart whenever you save a change to your code, which is useful during development.

You should see output that includes a local address, usually:

```text
http://127.0.0.1:8000
```

Open that address in a browser.

## Step 11: Try it

1. Open the app in a browser.
2. Upload a short video file.
3. Click **Evaluate Speech**.
4. Wait for the upload and Gemini response.
5. Review the feedback categories and overall score.

## What each piece does

- **`app.py`** is the backend. It is a Python program that runs a web server on your computer. When the browser sends a video, this is the code that receives it.
- **`static/index.html`** is the frontend. It is the web page you see in the browser. It handles the upload button, the loading spinner, and displaying the evaluation results.
- **`GEMINI_API_KEY`** is the environment variable that stores your API key. The backend reads it when it needs to authenticate with Google's servers. It is never sent to the browser or stored in your code files.
- **`/evaluate`** is the API route (the specific URL path) that the frontend calls when you click "Evaluate Speech." The backend receives the video at this route, sends it to Gemini, and returns the structured feedback.

## If something goes wrong

### `uv` is not recognized

Install `uv` using the setup guide:

[00-WINDOWS-PROJECT-SETUP.md](./00-WINDOWS-PROJECT-SETUP.md)

### `git` is not recognized

Install Git for Windows:

[https://git-scm.com/install/windows](https://git-scm.com/install/windows)

### `GEMINI_API_KEY environment variable not set`

Set the variable in PowerShell again:

```powershell
$env:GEMINI_API_KEY="paste-your-key-here"
```

### The app starts, but evaluation fails

Check these things:

1. The uploaded file is actually a video.
2. The video is not too large.
3. Your API key is valid.
4. Your internet connection is working.
5. Gemini is returning JSON in the expected format.

### The page opens, but looks blank or broken

Check that:

1. The file path is exactly `static\index.html`.
2. The file name is exactly `app.py`.
3. You started the app from the project root.

## Nice next improvements

Once the student has this working, they could try:

1. Adding a rubric for different speech event types.
2. Saving past evaluations in a database.
3. Exporting results to PDF.
4. Adding coach comments side-by-side with AI comments.
5. Letting a user compare two recordings of the same speaker.

## Official references

- Gemini API quickstart: [https://ai.google.dev/gemini-api/docs/quickstart](https://ai.google.dev/gemini-api/docs/quickstart)
- Google AI Studio: [https://ai.google.dev/aistudio](https://ai.google.dev/aistudio)
- Gemini API keys: [https://ai.google.dev/gemini-api/docs/api-key](https://ai.google.dev/gemini-api/docs/api-key)
- `uv` installation: [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/)
- `uv` project creation: [https://docs.astral.sh/uv/concepts/projects/init/](https://docs.astral.sh/uv/concepts/projects/init/)
