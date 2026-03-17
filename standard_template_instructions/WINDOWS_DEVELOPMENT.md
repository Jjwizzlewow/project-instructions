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

**Solution**: Always check for and kill processes occupying specific ports before starting servers.

#### Step 1: Check if a port is in use

```bash
# Check what process is using port 8010
netstat -ano | findstr :8010

# Output example:
# TCP    127.0.0.1:8010    0.0.0.0:0    LISTENING    13000
#                                                     ^^^^^ This is the PID
```

#### Step 2: Kill the specific process

```bash
# INCORRECT - kills ALL python processes (dangerous)
taskkill /F /IM python.exe

# INCORRECT - bash interprets /F as path F:/
taskkill /F /PID 13000

# CORRECT - use double slashes in Git Bash
taskkill //F //PID 13000

# ALTERNATIVE - use cmd /c wrapper
cmd /c "taskkill /F /PID 13000"
```

**Root cause**: Git Bash interprets single forward slashes as paths. Use double slashes (`//`) for Windows command flags in Git Bash, or wrap the command with `cmd /c`.

#### Step 3: Verify port is free

```bash
# Check again - should return nothing or "Error" if port is free
netstat -ano | findstr :8010
```

#### Complete workflow for restarting backend server

```bash
# 1. Find process on port 8010
netstat -ano | findstr :8010

# 2. If output shows a PID, kill that specific process
taskkill //F //PID <PID_NUMBER>

# 3. Verify port is free
netstat -ano | findstr :8010

# 4. Only then start the server
cd backend && "../.venv/Scripts/python.exe" run.py
```

**Rule**:
- Tests must not spawn uvicorn or any background servers
- Never kill processes by image name (e.g., `python.exe`) - always target specific PIDs
- Always check if a port is occupied before attempting to start a server on that port
- When starting a background process for testing, always kill it when done
- Use double slashes (`//`) for Windows command flags in Git Bash or wrap with `cmd /c`

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

## Creating Directories

### mkdir Command Syntax

**Problem**: Using `mkdir` with backslash-separated paths in a single command creates incorrectly named folders.

```bash
# INCORRECT - creates folders named "backendappconfig", "backendappdb", etc.
mkdir backend\app\config backend\app\db backend\app\routes

# CORRECT - chain separate mkdir commands with &&
mkdir backend\app && mkdir backend\app\config && mkdir backend\app\db && mkdir backend\app\routes

# ALTERNATIVE - use PowerShell
powershell -Command "New-Item -ItemType Directory -Path 'backend\app\config', 'backend\app\db', 'backend\app\routes' -Force"
```

**Root cause**: When multiple paths with backslashes are passed to mkdir in Git Bash or similar shells, the backslashes are not interpreted as path separators.

**Rule**: Always chain mkdir commands with `&&` on Windows, or use PowerShell's `New-Item` command for creating multiple nested directories.

---

## Searching and Listing Files

### Directory Listing Commands

**Problem**: Mixing bash-style commands (`ls`, `dir`) with Windows-style paths causes errors.

```bash
# INCORRECT - mixing bash command with Windows path separator
ls components\ui
dir components\ui

# CORRECT - use cmd /c for Windows commands
cmd /c dir components\ui

# CORRECT - convert to forward slashes for bash commands
ls components/ui

# BEST - use Glob tool instead of bash commands
# Use Glob tool with pattern: "components/ui/*"
```

**Root cause**: Git Bash interprets backslashes differently than Windows CMD. When you write `dir components\ui`, bash treats `\u` as an escape sequence, resulting in paths like `componentsui`.

**Rule**:
- For Windows `dir` command: Always use `cmd /c dir path\to\directory`
- For bash `ls` command: Always use forward slashes `ls path/to/directory`
- **PREFERRED**: Use the Glob tool instead of bash commands for file searching

### Searching for Files

```bash
# INCORRECT - using bash find/grep with Windows paths
find backend\app -name "*.py"

# CORRECT - use Glob tool
# Glob with pattern: "backend/app/**/*.py"

# ALTERNATIVE - use PowerShell
powershell -Command "Get-ChildItem -Path backend\app -Filter *.py -Recurse"
```

**Rule**: Always prefer using the Glob tool for file pattern matching. If you must use bash commands, convert all backslashes to forward slashes.

### Checking if Files/Directories Exist

```bash
# INCORRECT - using bash test with Windows paths
test -d backend\app

# CORRECT - use cmd /c to check directories
cmd /c "if exist backend\app echo exists"

# BEST - use Glob or Read tools
# Glob will return "No files found" if path doesn't exist
# Read will return error if file doesn't exist
```

**Rule**: Use Glob or Read tools to verify paths. If using bash commands, always convert backslashes to forward slashes or wrap with `cmd /c`.

---

## Assistant Instructions

- Do not use emojis or special characters in console output.
- Do not assume `pip` uses the correct Python environment — always use `.venv` explicitly.
- **Before starting a server, always check if the port is occupied using `netstat -ano | findstr :PORT`**
- **Never kill processes by image name (`taskkill /IM`) - always target specific PIDs**
- **Use double slashes (`//`) for Windows command flags in Git Bash (e.g., `taskkill //F //PID 123`)**
- When starting a background process for testing, always kill it when done
- Always resolve file paths using `settings.DATA_DIR`.
- Always chain mkdir commands with `&&` when creating nested directories on Windows.
- **Never mix bash commands (ls, dir, find) with Windows-style paths (backslashes).**
- **Always use `cmd /c` when running Windows commands, or convert paths to forward slashes for bash.**
- **Prefer Glob and Read tools over bash file search commands.**

---

These Windows-specific rules ensure consistent behavior, avoid encoding issues, and prevent common pitfalls in file access and process management. They apply to all local development and testing workflows on Windows.
