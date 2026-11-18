# Build an AI Coding Agent — Phase 3: File Tools

In Phase 2 the agent could only run shell commands. **Phase 3 gives it proper tools for files.** The AI can now read, write, edit, find and search files directly, and every file action stays locked inside your project folder.

- 📖 **Read and write**: open a file with line numbers, or create one from scratch
- ✏️ **Edit**: change one exact piece of text in a file without rewriting the whole thing
- 🔍 **Find and search**: list files by pattern (`**/*.py`) and search inside them (like `grep`)
- 🔒 **Sandboxed**: a file tool can't touch anything outside the folder you started the agent in
- 🏠 **Still fully local** with [Ollama](https://ollama.com): free, no API key, your data stays on your machine

## Quick start

You need **Python 3.10+** and **[Ollama](https://ollama.com/download)** running.

```bash
ollama pull qwen2.5:7b-instruct         # download the model (once)

python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

python -m agent.cli                     # start the agent
```

**Commands:** `/help` · `/model` · `/clear` (forget the chat) · `/inspect` · `/clear-inspect` · `/quit`

**Try asking:** "show me agent/core/config.py" · "find every Python file in this project" · "where is MAX_ITERATIONS used?" · "create notes.txt with a todo list, then change the first item"

> ⚠️ The agent changes real files in the folder you start it from. Try it in a folder you don't mind changing.

## The new tools

The agent now has **6 tools**: `bash` from Phase 2, plus 5 file tools.

| Tool | What it does | Example |
|------|-------------|---------|
| `read_file` | Shows a file with line numbers. Can start at a line and read only some lines | `read_file("agent/cli.py", offset=30, limit=20)` |
| `write_file` | Creates or overwrites a file. Makes missing folders too | `write_file("notes/todo.txt", "buy milk")` |
| `edit_file` | Replaces one exact piece of text. Fails if it's missing or appears more than once | `edit_file("app.py", "DEBUG = True", "DEBUG = False")` |
| `list_files` | Finds files by name pattern (glob). Shows up to 200 | `list_files("**/*.py")` |
| `search_files` | Finds lines that match a regex inside files. Shows up to 100 | `search_files("def handle_", file_pattern="*.py")` |

**Why not just use `bash`?** The agent *could* use `cat`, `sed` and `grep`, but small models often get the quoting and flags wrong. Dedicated tools have simple inputs, give clear error messages the model can fix itself, and can be sandboxed.

## Data flow

What happens when you type **"Change DEBUG to False in app.py"**:

```
 ┌───────────────┐       ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
 │      USER     │       │     AGENT     │       │      LLM      │       │      TOOL     │
 │    (cli.py)   │       │   (loop.py)   │       │    (Ollama)   │       │   (files.py)  │
 └───────┬───────┘       └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
         │                       │                       │                       │
         │ 1. "Change DEBUG to   │                       │                       │
         │    False in app.py"   │                       │                       │
         │──────────────────────▶│                       │                       │
         │                       │ 2. messages + tools   │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │ 3. tool call:         │                       │
         │                       │    read_file(app.py)  │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │                       │ 4. safe_path → read   │                       │
         │                       │───────────────────────┼──────────────────────▶│
         │                       │ 5. "1  DEBUG = True"  │                       │
         │                       │◀──────────────────────┼───────────────────────│
         │                       │                       │                       │
         │                       │ 6. tool response      │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │ 7. tool call:         │                       │
         │                       │    edit_file(app.py,  │                       │
         │                       │    old, new)          │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │                       │ 8. safe_path → edit   │                       │
         │                       │───────────────────────┼──────────────────────▶│
         │                       │ 9. "Edited app.py"    │                       │
         │                       │◀──────────────────────┼───────────────────────│
         │                       │                       │                       │
         │                       │ 10. tool response     │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │ 11. text: "Done"      │                       │
         │                       │     tool call: none   │                       │
         │                       │◀──────────────────────│                       │
         │ 12. final answer      │                       │                       │
         │◀──────────────────────│                       │                       │
         ▼                       ▼                       ▼                       ▼
```

The loop is exactly the same as Phase 2. It just goes around twice: once to **read**, once to **edit**. The system prompt tells the model to always read a file before editing it, so it copies the exact text to replace.

## Sandboxing

Every file tool sends its path through `safe_path()` first. From `agent/tools/safety.py`:

```python
WORKSPACE = Path(os.getcwd()).resolve()          # the folder you started the agent in

def safe_path(file_path: str) -> Path:
    resolved = (WORKSPACE / file_path).resolve()  # turns "../" and links into a real path
    if not resolved.is_relative_to(WORKSPACE):
        raise ValueError("Path resolves outside the workspace")
    return resolved
```

| The model asks for | Result |
|--------------------|--------|
| `src/main.py` | ✅ allowed |
| `../../etc/passwd` | ❌ blocked: goes outside the folder |
| `/Users/me/.ssh/id_rsa` | ❌ blocked: absolute path outside the folder |

A blocked path doesn't crash the agent. The tool returns an `Error: ...` message, and the model reads it and tries something else.

> ℹ️ The sandbox covers the **file tools** only. The `bash` tool still has Phase 2's guards (blocked commands, timeout, output limit), but it can run commands anywhere.

## Safety guards

| Guard | What it does | Where |
|-------|-------------|-------|
| **Path sandbox** ✨ NEW | File tools can't leave the workspace folder | `tools/safety.py` |
| **Unique edits** ✨ NEW | `edit_file` refuses if the text isn't found or matches more than once | `tools/files.py` |
| **Result limits** ✨ NEW | `list_files` shows at most 200 files, `search_files` at most 100 matches | `tools/files.py` |
| **Skip junk folders** ✨ NEW | `search_files` skips hidden folders, `node_modules`, `__pycache__`, `venv` | `tools/files.py` |
| **Blocked commands** | Refuses `rm -rf /`, `sudo rm`, `shutdown`, `reboot`, `mkfs`, `dd if=`, fork bombs ... | `tools/safety.py` |
| **60-second timeout** | Stops commands that hang | `tools/bash.py` |
| **Output limit** | Output over 50,000 characters keeps only the start and end | `tools/safety.py` |
| **50-round limit** | Stops the agent after 50 AI calls so it can't loop forever | `core/loop.py` |

## Project structure

```
agent/
├── cli.py              Chat program (start here). ✨ Now registers 6 tools
├── core/
│   ├── config.py       Settings: model name, Ollama address
│   ├── loop.py         The agent loop (unchanged from Phase 2)
│   ├── streaming.py    Prints answers live
│   └── inspector.py    Records every AI call for /inspect
├── providers/
│   ├── base.py         Rules every AI provider must follow
│   ├── registry.py     Picks the provider
│   └── ollama.py       Talks to Ollama, sends the tool list, collects tool calls
└── tools/
    ├── registry.py     Keeps the list of tools, calls one by name
    ├── bash.py         The bash tool
    ├── files.py        ✨ NEW: read_file, write_file, edit_file, list_files, search_files
    └── safety.py       ✨ Now also has safe_path() for the sandbox
```

Want a different model? Change `MODEL` in `agent/core/config.py`. Small models like `qwen2.5:1.5b-instruct` are faster but worse at choosing tools.

## What's next

**Phase 4: Smart context.** Long sessions fill up the model's memory. The agent learns to compress old messages and to read project rules from a `CLAUDE.md` file.
