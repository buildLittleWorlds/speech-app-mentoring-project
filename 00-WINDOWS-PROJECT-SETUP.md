# Windows Project Setup for a New Python App

This guide is for a student using a Windows computer. It walks through the basics of getting a new folder ready for a Python project that uses Git, `uv`, and a Gemini API key from Google AI Studio.

Verified against official docs on March 19, 2026.

## What you are setting up

By the end of this guide, you will know how to:

1. Install Git on Windows if needed.
2. Install `uv` on Windows if needed.
3. Create a new project folder.
4. Initialize Git in that folder.
5. Create a bare-bones Python project with `uv`.
6. Add Python packages to the project.
7. Create and store a Google AI Studio API key in your Windows environment.

## Recommended tool: PowerShell

Use **PowerShell** for the commands in this guide.

You can open it by:

1. Press `Windows`.
2. Type `PowerShell`.
3. Open **Windows PowerShell** or **PowerShell**.

## Step 1: Check whether Git is already installed

In PowerShell, run:

```powershell
git --version
```

If you see a version number, Git is installed and you can move on.

If PowerShell says the command is not recognized, install Git for Windows from the official page:

[Git for Windows](https://git-scm.com/install/windows)

After installing Git, close PowerShell and open it again, then run:

```powershell
git --version
```

## Step 2: Check whether `uv` is already installed

In PowerShell, run:

```powershell
uv --version
```

If you see a version number, `uv` is installed and you can move on.

If PowerShell says the command is not recognized, install `uv`.

### Option A: Official PowerShell installer

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close PowerShell and open it again. Then confirm:

```powershell
uv --version
```

### Option B: Install with `winget`

```powershell
winget install --id=astral-sh.uv -e
```

Then confirm:

```powershell
uv --version
```

## Step 3: Create a folder for a new project

Pick a place where you want to keep your project. For example:

```powershell
cd $HOME\Desktop
mkdir speech-eval-student
cd speech-eval-student
```

You can check where you are with:

```powershell
pwd
```

## Step 4: Initialize Git in the folder

Run:

```powershell
git init
```

That turns the folder into a Git repository.

Optional but helpful for the first time you use Git on a machine:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## Step 5: Create a bare-bones Python project with `uv`

Inside your project folder, run:

```powershell
uv init --bare
```

This creates a minimal `pyproject.toml` without extra starter files.
It also does **not** initialize Git for you, which is why `git init` is a separate step.

If your machine does not already have a usable Python version, `uv` can manage Python for you. You can explicitly install Python 3.12 with:

```powershell
uv python install 3.12
```

If you want to pin the folder to Python 3.12, run:

```powershell
uv python pin 3.12
```

## Step 6: Add the packages you need

For a web app like this one, you can add packages with:

```powershell
uv add fastapi uvicorn python-multipart google-genai
```

This updates `pyproject.toml`, creates a lockfile, and sets up the local environment.

## Step 7: Make a simple `.gitignore`

Create a file named `.gitignore` in the project folder and paste this:

```gitignore
.venv/
__pycache__/
.env
```

This keeps local environment files and cache folders out of Git.

## Step 8: Create a Google AI Studio API key

Go to:

[Google AI Studio](https://ai.google.dev/aistudio)

Then:

1. Sign in with your Google account.
2. Open the area for API keys.
3. Create a Gemini API key.
4. Copy the key somewhere safe temporarily so you can add it to your environment.

Important:

- Treat the key like a password.
- Do not paste it into code that gets committed to Git.
- Do not put it into frontend JavaScript.
- For this project, the key belongs in the backend environment only.

## Step 9: Add the API key to your Windows environment

### Official Windows GUI method

Google's documentation gives this Windows flow:

1. Search for `Environment Variables` in the Windows search bar.
2. Open the option to modify system settings.
3. Click **Environment Variables**.
4. Under **User variables** or **System variables**, click **New...**
5. Set the variable name to `GEMINI_API_KEY`.
6. Paste your API key as the value.
7. Click **OK**.
8. Open a new PowerShell window so the new variable is available.

### Temporary method for the current PowerShell window

This is good for testing right now:

```powershell
$env:GEMINI_API_KEY="paste-your-key-here"
```

This only lasts for the current PowerShell session.

### More persistent method with `setx`

```powershell
setx GEMINI_API_KEY "paste-your-key-here"
```

Important:

- After running `setx`, close PowerShell and open a new PowerShell window.
- The new value usually does not appear in the current window.

Then test it in a new PowerShell window:

```powershell
echo $env:GEMINI_API_KEY
```

If you see your key, the variable is set.

## Step 10: Start running Python commands through `uv`

Once your project is set up, run commands through `uv` so they use the project environment.

Examples:

```powershell
uv run python --version
uv run python app.py
uv run uvicorn app:app --reload
```

## Very short version

If you already have Git and `uv`, the fastest starter flow is:

```powershell
cd $HOME\Desktop
mkdir my-new-app
cd my-new-app
git init
uv init --bare
uv add fastapi uvicorn python-multipart google-genai
```

Then create your files, set `GEMINI_API_KEY`, and run the app with `uv run ...`.

## Good starter habit

When beginning any new coding project, ask:

1. What folder am I working in?
2. Did I run `git init`?
3. Did I create the project with `uv init --bare`?
4. Did I add my packages with `uv add ...`?
5. Did I keep secrets like API keys out of the code?

## Official references

- `uv` installation: [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/)
- `uv` project creation: [https://docs.astral.sh/uv/concepts/projects/init/](https://docs.astral.sh/uv/concepts/projects/init/)
- `uv` Python management: [https://docs.astral.sh/uv/guides/install-python/](https://docs.astral.sh/uv/guides/install-python/)
- Git for Windows: [https://git-scm.com/install/windows](https://git-scm.com/install/windows)
- Gemini API keys: [https://ai.google.dev/gemini-api/docs/api-key](https://ai.google.dev/gemini-api/docs/api-key)
- Gemini API quickstart: [https://ai.google.dev/gemini-api/docs/quickstart](https://ai.google.dev/gemini-api/docs/quickstart)
