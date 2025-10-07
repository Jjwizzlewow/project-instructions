# Windows Development Guidelines

This document outlines development practices for running Python-based backend projects on Windows. It addresses terminal output, virtual environments, process cleanup, and file path reliability.

These rules are enforced to ensure consistency, avoid platform-specific bugs, and prevent environment issues when using Python, FastAPI, and local tooling.

---

## Terminal Output

### Unicode / Emoji Encoding

**Problem**: Windows terminal (CMD or PowerShell) often fails when printing Unicode characters, leading to `UnicodeEncodeError`.

```python
# Incorrect - may crash on Windows
print("🚀 Starting server...")

# Correct - plain ASCII is always safe
print("Starting server...")
```

**Root cause**: Windows terminals often use `cp1252` encoding by default, which does not support many Unicode symbols.

**Rule**: Use plain ASCII characters in all console/log output. Avoid emojis and Unicode-only symbols.

---

## Package Management

### Virtual Environment Best Practices

**Problem**: Accidentally installing packages globally instead of into the local virtual environment.

```bash
# Correct - use the explicit venv path
".venv\Scripts\python.exe" -m pip install package_name

# Incorrect - may use global Python
pip install package_name
```

**Rule**: Always activate or reference the `.venv` path directly when installing packages. Do not assume `pip` targets the virtual environment.

---

## Process Management

### Background Process Cleanup

**Problem**: Development or test scripts leave background servers running, blocking ports or causing stale behavior.

**Solution**: After aborting a run or test, ensure all background processes are stopped:

```bash
# Kill stuck processes manually if needed
# Use Task Manager or taskkill
```

**Rule**: Tests must not spawn uvicorn or any background servers. The assistant must never generate code that leaves persistent processes running in the background.

---

## File Path Reliability

### Use settings.DATA_DIR

**Problem**: Relative paths fail when code is run from different directories, leading to file not found errors.

```python
# Correct - resolve all file paths from settings.DATA_DIR
from app.config.settings import settings
import os

file_path = os.path.join(settings.DATA_DIR, "myfile.json")

# Incorrect - fragile relative paths or ad-hoc roots
file_path = os.path.join("..", "data", "myfile.json")
```

**Rule**: Always resolve file paths using `settings.DATA_DIR`. `settings.DATA_DIR` should be an absolute path and default to `backend/data`.

---

## Assistant Instructions

- Do not use emojis or special characters in console output.
- Do not assume `pip` uses the correct Python environment — always use `.venv` explicitly.
- Do not start or leave background server processes running during development or testing.
- Always resolve file paths using `settings.DATA_DIR`.

---

These Windows-specific rules ensure consistent behavior, avoid encoding issues, and prevent common pitfalls in file access and process management. They apply to all local development and testing workflows on Windows.
