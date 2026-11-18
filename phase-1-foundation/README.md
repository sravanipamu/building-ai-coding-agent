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

## How it works

```
You type a question
   ▼
cli.py        adds it to the conversation and sends it on
   ▼
ollama.py     adds the system prompt and sends everything to the local model
   ▼
Ollama        writes the answer piece by piece
   ▼
streaming.py  prints each piece on screen as it arrives
   ▼
cli.py        saves the answer, so the next question has context
```

**The key trick:** an LLM remembers nothing. The agent sends the **whole conversation** every time. Its "memory" is just a Python list.

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
