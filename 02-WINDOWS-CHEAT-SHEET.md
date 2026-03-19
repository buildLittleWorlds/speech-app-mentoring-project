# Windows Cheat Sheet: Build the Speech Evaluator Prototype

This is the fast version for a student on Windows using PowerShell.

## 1. Open PowerShell

Press `Windows`, type `PowerShell`, and open it.

## 2. Check tools

```powershell
git --version
uv --version
```

If `git` is missing:

[https://git-scm.com/install/windows](https://git-scm.com/install/windows)

If `uv` is missing:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Then close PowerShell and open it again.

## 3. Make the project folder

```powershell
cd $HOME\Desktop
mkdir speech-eval-prototype
cd speech-eval-prototype
```

## 4. Initialize Git and Python project

```powershell
git init
uv init --bare
uv python install 3.12
uv python pin 3.12
uv add fastapi uvicorn python-multipart google-genai
mkdir static
```

## 5. Create these 3 files

Create:

1. `.gitignore`
2. `app.py`
3. `static\index.html`

## 6. Paste this into `.gitignore`

```gitignore
.venv/
__pycache__/
.env
```

## 7. Paste the code

Copy the code from these walkthrough files:

- [01-BUILD-SPEECH-EVALUATOR-PROTOTYPE.md](./01-BUILD-SPEECH-EVALUATOR-PROTOTYPE.md)

Paste the big Python block into `app.py`.

Paste the big HTML block into `static\index.html`.

## 8. Set the Gemini API key

Temporary for this PowerShell window:

```powershell
$env:GEMINI_API_KEY="paste-your-key-here"
```

More permanent:

```powershell
setx GEMINI_API_KEY "paste-your-key-here"
```

If you use `setx`, close PowerShell and open a new one before running the app.

## 9. Start the app

```powershell
uv run uvicorn app:app --reload
```

Then open:

```text
http://127.0.0.1:8000
```

## 10. Test it

1. Upload a short video.
2. Click **Evaluate Speech**.
3. Wait for the results.

## If something breaks

If `uv` is not recognized:

- reinstall `uv`
- close and reopen PowerShell

If `git` is not recognized:

- install Git for Windows
- close and reopen PowerShell

If the app says `GEMINI_API_KEY environment variable not set`:

```powershell
$env:GEMINI_API_KEY="paste-your-key-here"
```

If the page looks broken:

- make sure the file is named `static\index.html`
- make sure the backend file is named `app.py`
- make sure you started `uvicorn` from the project root folder

## Full guides

- [00-WINDOWS-PROJECT-SETUP.md](./00-WINDOWS-PROJECT-SETUP.md)
- [01-BUILD-SPEECH-EVALUATOR-PROTOTYPE.md](./01-BUILD-SPEECH-EVALUATOR-PROTOTYPE.md)
