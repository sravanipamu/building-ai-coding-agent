# Build an AI Coding Agent — Phase 1: Foundation

Ever wondered what happens inside tools like Claude Code or Cursor? **Build one yourself, from scratch, in plain Python.** No frameworks, nothing hidden: every line is yours to read.

Phase 1 builds the base: a **terminal chat agent** that talks to an AI model running **on your own computer**.

- 💬 **Chat in the terminal** with memory of the conversation
- ⚡ **Live answers** that stream word by word
- 🔒 **Fully local** with [Ollama](https://ollama.com): free, no API key, your data stays on your machine
- 🔍 **`/inspect`** shows the exact data sent to the AI and what came back. See how LLMs really work.

## Quick start

You need **Python 3.10+** and **[Ollama](https://ollama.com/download)** running.

```bash
ollama pull qwen2.5:7b-instruct         # download the model (once)

python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

python -m agent.cli                     # start chatting
```

**Commands:** `/help` · `/model` · `/clear` (forget the chat) · `/inspect` · `/clear-inspect` · `/quit`

## Data flow

What happens when you type **"What is Python?"**:

```
 ┌───────────────┐       ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
 │      USER     │       │      CLI      │       │    PROVIDER   │       │      LLM      │
 │   (terminal)  │       │    (cli.py)   │       │  (ollama.py)  │       │    (Ollama)   │
 └───────┬───────┘       └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
         │                       │                       │                       │
         │ 1. types query        │                       │                       │
         │    "What is Python?"  │                       │                       │
         │──────────────────────▶│                       │                       │
         │                       │                       │                       │
         │                       │ 2. /command? NO       │                       │
         │                       │    add to messages    │                       │
         │                       │                       │                       │
         │                       │ 3. messages           │                       │
         │                       │──────────────────────▶│                       │
         │                       │                       │                       │
         │                       │                       │ 4. add system prompt  │
         │                       │                       │    at the front       │
         │                       │                       │                       │
         │                       │                       │ 5. system + messages  │
         │                       │                       │──────────────────────▶│
         │                       │                       │                       │
         │                       │                       │ 6. answer in pieces   │
         │                       │                       │    "Python" " is" ... │
         │                       │                       │◀──────────────────────│
         │                       │                       │                       │
         │                       │ 7. each piece         │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │ 8. prints it live     │                       │                       │
         │    (streaming.py)     │                       │                       │
         │◀──────────────────────│                       │                       │
         │                       │                       │                       │
         │                       │                       │ 9. done               │
         │                       │                       │◀──────────────────────│
         │                       │                       │                       │
         │                       │ 10. full answer       │                       │
         │                       │◀──────────────────────│                       │
         │                       │                       │                       │
         │                       │                       │ 11. inspector records │
         │                       │                       │     input + output    │
         │                       │                       │                       │
         │                       │ 12. save answer       │                       │
         │                       │     to messages       │                       │
         │                       │     ready for next    │                       │
         │                       │                       │                       │
         ▼                       ▼                       ▼                       ▼
```

Steps 6 to 8 repeat for every piece, so you see the answer appear word by word instead of waiting for the whole thing.

What actually gets sent to the model at step 5:

```python
[
  {"role": "system", "content": "You are a helpful AI coding agent..."},  # hidden instructions
  {"role": "user",   "content": "What is Python?"}                       # your question
]
```

**The key trick:** an LLM remembers nothing. The agent sends the **whole conversation** every time. Its "memory" is just this Python list, and it grows by one user message and one AI answer each turn.

## Project structure

```
agent/
├── cli.py              Chat program (start here)
├── core/
│   ├── config.py       Settings: model name, Ollama address
│   ├── streaming.py    Prints answers live
│   └── inspector.py    Records every AI call for /inspect
└── providers/
    ├── base.py         Rules every AI provider must follow
    ├── registry.py     Picks the provider
    └── ollama.py       Talks to Ollama
```

Want a different model? Change `MODEL` in `agent/core/config.py`.

## What's next

**Phase 2: The agent loop.** The AI can ask to run a tool, see the result and keep going. That's the step where a chatbot becomes an **agent**.
