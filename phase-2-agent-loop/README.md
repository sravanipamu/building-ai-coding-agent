# Build an AI Coding Agent — Phase 2: The Agent Loop

In Phase 1 you built a chatbot: you ask, it answers, done. **Phase 2 turns it into an agent.** The AI can now run commands on your computer, read the results and keep working until the job is finished.

All of it comes from one small idea: **a `while` loop plus one tool**.

- 🔁 **The agent loop**: the AI asks to run a tool, sees the result, decides what to do next
- 🛠️ **A bash tool**: the AI can run shell commands (`ls`, `cat`, `python script.py`, ...)
- 🛡️ **Safety guards**: dangerous commands are blocked, slow ones are stopped, long output is cut short
- 🔒 **Still fully local** with [Ollama](https://ollama.com): free, no API key, your data stays on your machine

## Quick start

You need **Python 3.10+** and **[Ollama](https://ollama.com/download)** running.

```bash
ollama pull qwen2.5:7b-instruct         # download the model (once)

python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

python -m agent.cli                     # start the agent
```

**Commands:** `/help` · `/model` · `/clear` (forget the chat) · `/inspect` · `/clear-inspect` · `/quit`

**Try asking:** "list the files in this folder" · "how many lines are in README.md?" · "write a Python script that prints 1 to 10 and run it"

> ⚠️ The agent runs real commands in the folder you start it from. It blocks the worst ones, but a normal `rm file.txt` is allowed. Try it in a folder you don't mind changing.

## Data flow

What happens when you type **"How many files are in this folder?"**:

```
 ┌───────────────┐       ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
 │      USER     │       │     AGENT     │       │      LLM      │       │      TOOL     │
 │    (cli.py)   │       │   (loop.py)   │       │    (Ollama)   │       │   (bash.py)   │
 └───────┬───────┘       └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
         │                       │                       │                       │
         │ 1. types query        │                       │                       │
         │    "How many files?"  │                       │                       │
         │──────────────────────▶│                       │                       │
         │                       │                       │                       │
         │                       │ 2. messages + tools   │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │ 3. text: none         │                       │
         │                       │    tool call: bash    │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │                       │ 4. tool call present? │                       │
         │                       │    YES → call tool    │                       │
         │                       │                       │                       │
         │                       │ 5. run: ls | wc -l    │                       │
         │                       │───────────────────────┼──────────────────────▶│
         │                       │                       │                       │
         │                       │ 6. tool response: "5" │                       │
         │                       │◀──────────────────────┼───────────────────────│
         │                       │                       │                       │
         │                       │ 7. tool response      │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │ 8. text: "5 files"    │                       │
         │                       │    tool call: none    │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │                       │ 9. text present?      │                       │
         │                       │    tool call none?    │                       │
         │                       │    YES → done         │                       │
         │                       │                       │                       │
         │ 10. final answer      │                       │                       │
         │◀──────────────────────│                       │                       │
         │                       │                       │                       │
         ▼                       ▼                       ▼                       ▼
```

If the LLM asks for another tool at step 8, the agent goes back to step 4 and repeats. It keeps looping until the LLM replies with text and no tool call.

What the conversation looks like after one trip around the loop:

```python
[
  {"role": "user",      "content": "How many files are in this folder?"},
  {"role": "assistant", "tool_calls": [{"function": {"name": "bash",
                                         "arguments": {"command": "ls | wc -l"}}}]},
  {"role": "tool",      "content": "5"},                                  # the tool's output
  {"role": "assistant", "content": "There are 5 files in this folder."}   # final answer
]
```

**The key trick:** the AI never runs anything itself. It only **asks** for a tool by name. Your Python code runs the tool and gives the output back as a new message. The model reads that message on its next turn, the same way it reads yours.

## The agent loop

This is the heart of every AI coding agent, including the big ones. From `agent/core/loop.py`:

```python
for i in range(MAX_ITERATIONS):                  # at most 50 rounds
    response = provider.chat_stream(messages, tools)
    messages.append(response)

    if not response["tool_calls"]:
        return                                   # no tool requested → done

    for tool_call in response["tool_calls"]:
        result = tools.dispatch(tool_call["name"], tool_call["arguments"])
        messages.append({"role": "tool", "content": result})
    # loop again: the model now sees the tool results
```

Every later phase adds new tools and features **around** this loop. The loop itself never changes.

## Safety guards

| Guard | What it does | Where |
|-------|-------------|-------|
| **Blocked commands** | Refuses `rm -rf /`, `sudo rm`, `shutdown`, `reboot`, `mkfs`, `dd if=`, fork bombs ... | `tools/safety.py` |
| **60-second timeout** | Stops commands that hang | `tools/bash.py` |
| **Output limit** | Output over 50,000 characters keeps only the start and end | `tools/safety.py` |
| **50-round limit** | Stops the agent after 50 AI calls so it can't loop forever | `core/loop.py` |

## Project structure

```
agent/
├── cli.py              Chat program (start here)
├── core/
│   ├── config.py       Settings: model name, Ollama address
│   ├── loop.py         ✨ NEW: the agent loop
│   ├── streaming.py    Prints answers live
│   └── inspector.py    Records every AI call for /inspect
├── providers/
│   ├── base.py         Rules every AI provider must follow
│   ├── registry.py     Picks the provider
│   └── ollama.py       Talks to Ollama, sends the tool list, collects tool calls
└── tools/
    ├── registry.py     ✨ NEW: keeps the list of tools, calls one by name
    ├── bash.py         ✨ NEW: the bash tool
    └── safety.py       ✨ NEW: blocks dangerous commands, trims long output
```

Want a different model? Change `MODEL` in `agent/core/config.py`. Small models like `qwen2.5:1.5b-instruct` are faster but worse at choosing tools.

## What's next

**Phase 3: File tools.** The agent gets dedicated tools to read, write, edit and search files, locked inside your project folder so it can't wander off.
