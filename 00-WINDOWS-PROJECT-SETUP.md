# Windows Project Setup for a New Python App

This guide is for a student using a Windows computer. It walks through the basics of getting a new folder ready for a Python project that uses Git, `uv`, and a Gemini API key from Google AI Studio.

Verified against official docs on March 19, 2026.

## What are these tools?

This project uses several tools that professional developers use every day. Here is a quick overview of each one before you start installing anything.

### GitHub and Git

[GitHub](https://github.com) is a website where developers store and share code. It keeps a complete history of every change ever made to a project, so you can always go back to an earlier version if something breaks. Git is the program that runs on your own computer and tracks those changes locally. When you run `git init`, you are telling Git to start watching a folder. Later, you can push your local history up to GitHub so other people (or your future self on a different computer) can see it.

For this project, your mentor has a GitHub repository where the guides you are reading right now are hosted: [speech-app-mentoring-project](https://github.com/buildLittleWorlds/speech-app-mentoring-project).

### uv

[uv](https://docs.astral.sh/uv/) is a fast, modern tool for managing Python projects. It handles installing Python itself, creating isolated project environments, and adding packages (libraries of code that other people have written so you do not have to write everything from scratch). It replaces older tools like `pip` and `virtualenv` with a single command.

You can find the official install page here: [Install uv](https://docs.astral.sh/uv/getting-started/installation/).

### Google AI Studio, API keys, and why we need API access

[Google AI Studio](https://ai.google.dev/aistudio) is a website where you can try out Google's Gemini AI models by typing prompts and uploading files directly in a browser. It is great for experimenting. But if you want to build an app that sends data to Gemini automatically from your own code — without a person manually typing into a website — you need **API access**.

An API (Application Programming Interface) is a way for one program to talk to another program. When your Python backend sends a video to Gemini and gets evaluation feedback back, that conversation happens through the Gemini API. The API key is like a password that proves your app has permission to use Gemini. Google AI Studio is where you create that key.

The short version: AI Studio is for humans experimenting in a browser. The API is for code talking to Gemini automatically. We need the API because our app's Python backend is the one sending the video and receiving the evaluation, not a person clicking buttons on a website.

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

That turns the folder into a **Git repository** (often shortened to "repo"). A repository is just a folder that Git is watching. From this point on, Git can track every change you make to the files inside it. Think of it like turning on "track changes" in a Word document, except it covers every file in the folder at once.

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

**Packages** are libraries of code that other developers have written and shared publicly. Instead of writing a web server from scratch, for example, you install a package that already knows how to handle web requests. This is one of the most important ideas in modern software development: you build on top of other people's work.

For a web app like this one, you can add packages with:

```powershell
uv add fastapi uvicorn python-multipart google-genai
```

This updates `pyproject.toml`, creates a lockfile, and sets up the local environment.

## Step 7: Make a simple `.gitignore`

A `.gitignore` file tells Git which files and folders to ignore — that is, which ones should never be tracked or uploaded to GitHub. This is important for two reasons: some files are private (like your API key), and some files are generated automatically and would just clutter up the history (like the `.venv/` folder that `uv` creates).

Create a file named `.gitignore` in the project folder and paste this:

```gitignore
.venv/
__pycache__/
.env
```

This keeps local environment files and cache folders out of Git.

## Step 8: Create a Google AI Studio API key

An **API key** is a long string of characters that acts like a password between your app and Google's servers. Every time your Python code sends a video to Gemini, it includes this key so Google knows the request is authorized. Without it, Google would reject the request.

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
