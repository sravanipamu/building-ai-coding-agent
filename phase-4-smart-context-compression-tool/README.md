# Build an AI Coding Agent — Phase 4: Smart Context

**The problem:** the AI model can only remember a limited amount of text. In Phase 3, every message and every file the agent read was kept forever, so in a long session the memory filled up and the agent started forgetting or failing.

**The fix in Phase 4:** the agent cleans up its own memory before every AI call, and it reads your project's rules from a `CLAUDE.md` file.

- 🗜️ **Shrinks old tool results**: keeps only the last 3 in full
- 📝 **Summarizes when memory is full**: replaces the whole chat with a short summary
- 📋 **Follows project rules**: from a `CLAUDE.md` file in your project
- 🏠 **Still fully local** with [Ollama](https://ollama.com): free, no API key

## Quick start

You need **Python 3.10+** and **[Ollama](https://ollama.com/download)** running.

```bash
ollama pull qwen2.5:7b-instruct         # download the model (once)

python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

python -m agent.cli                     # start the agent
```

**Try it:** ask *"read every file in agent/tools"*, then type `/debug` to see the memory, `/tokens` to see its size, and `/compact` to shrink it.

> ⚠️ The agent changes real files in the folder you start it from. Try it in a folder you don't mind changing.

## The idea in 3 parts

### 1. Shrink old tool results (every time)

Tool results are the biggest thing in memory: a whole file, a long search. The agent rarely needs old ones again, so before every AI call it keeps the **last 3** and replaces the rest with a short note.

```
Before                                     After
──────                                     ─────
tool: (all of cli.py, 180 lines)           tool: [Previous tool result — see context above]
tool: (all of loop.py, 120 lines)          tool: [Previous tool result — see context above]
tool: (all of bash.py)                     tool: (all of bash.py)       ← kept
tool: (all of files.py)                    tool: (all of files.py)      ← kept
tool: (all of safety.py)                   tool: (all of safety.py)     ← kept
```

No AI call needed, so it's free.

### 2. Summarize when memory is full (automatic)

If memory is still too big (**over ~100,000 tokens**), the agent asks the AI to write a short summary of the whole chat, then throws away all the old messages and keeps only the summary.

```
Before: 200 messages          After: 2 messages
                              user:      [Context Summary] The user asked to... we edited...
                              assistant: Understood. I have the context...
```

You'll see `[Compacting context...]` when this happens.

### 3. Summarize when asked (manual)

Same as part 2, but on request:

- **You** type `/compact`
- **The AI** calls the new `compact` tool. It decides this by itself, so it doesn't happen often.

## Data flow

What happens after you type a message:

```
              You type a message
                      │
                      ▼
      ┌───────────────────────────────┐
      │ 1. Shrink old tool results    │   always
      └───────────────┬───────────────┘
                      │
                      ▼
            Memory over 100K tokens?
               │               │
              yes              no
               │               │
               ▼               │
      ┌─────────────────┐      │
      │ 2. Summarize    │      │
      │    the chat     │      │
      └────────┬────────┘      │
               └───────┬───────┘
                       ▼
      ┌───────────────────────────────┐
      │ 3. Send to the AI:            │
      │    rules + CLAUDE.md + memory │
      └───────────────┬───────────────┘
                      │
                      ▼
             AI wants to use a tool?
               │               │
              yes              no
               │               │
               ▼               ▼
      ┌─────────────────┐   ┌─────────────────┐
      │ 4. Run the tool,│   │ Show the answer │
      │  save the result│   │      Done       │
      └────────┬────────┘   └─────────────────┘
               │
               └──────► back to step 1
```

**Example:** you ask *"explain the files in agent/tools"*.

1. Memory is small, so nothing changes. The AI is called.
2. The AI asks to read `bash.py`. The agent reads it and saves the result. Back to step 1.
3. The AI reads `files.py`, then `registry.py`, then `safety.py`. Each time the loop goes back to step 1.
4. Now there are 4 tool results. Step 1 turns the oldest (`bash.py`) into a short note, so the AI only sees the newest 3 in full.
5. The AI has what it needs and writes the explanation. No tool needed, so you get the answer.

Only steps 1 and 2 are new. The rest of the loop is the same as Phase 2 and 3.

## CLAUDE.md: your project's rules

Create a file named `CLAUDE.md` in your project and write rules in plain words:

```markdown
# Project Rules
- Use Python 3.12+ features
- Always run tests before committing
- Follow Google docstring style
```

When the agent starts, it adds these rules to its instructions. You'll see `Loaded CLAUDE.md project instructions.`

**Where it looks** (it uses every file it finds):

1. The folder you start the agent in
2. Every parent folder, all the way up
3. `~/.claude/CLAUDE.md`, for your personal rules in every project

Good to know:
- The rules are never removed by compression. They aren't part of the chat memory.
- Edited `CLAUDE.md`? Restart the agent to load it again.

## Commands

| Command | What it does |
|---------|-------------|
| `/compact` ✨ NEW | Summarize the chat now |
| `/tokens` ✨ NEW | Show roughly how big the memory is |
| `/debug` ✨ NEW | List every message in memory (first 80 characters) |
| `/clear` | Forget the whole chat |
| `/model` | Show the model |
| `/inspect` | Show every AI call's input and output |
| `/clear-inspect` | Clear the `/inspect` records |
| `/help` | Show the commands |
| `/quit` | Exit |

## Tools

The agent now has **7 tools**: the 6 from Phase 3 (`bash`, `read_file`, `write_file`, `edit_file`, `list_files`, `search_files`) plus the new **`compact`**.

## What changed in the code

```
agent/
├── cli.py               ✨ loads CLAUDE.md, adds the compact tool and new commands
├── core/
│   └── loop.py          ✨ runs steps 1 and 2 before every AI call
└── memory/              ✨ NEW folder
    ├── claude_md.py     finds and reads CLAUDE.md files
    └── compression.py   shrinks, measures and summarizes the memory
```

Everything else is the same as Phase 3.

**Want to experiment?** In `agent/memory/compression.py`:
- `KEEP_RECENT_TOOLS = 3`: how many tool results stay in full
- `AUTO_COMPACT_THRESHOLD = 100_000`: when step 2 kicks in. Set it to `2_000` to see a summary happen quickly.

> ℹ️ Tokens are estimated, not counted exactly: about 4 characters = 1 token. Ollama's real memory limit is often much smaller than 100K, so in long sessions you may want to run `/compact` yourself.

## What's next

**Phase 5: Skills.** Instead of putting all its know-how in the instructions up front, the agent loads extra knowledge only when it needs it.
