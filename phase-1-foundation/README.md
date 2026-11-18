# Build an AI Coding Agent — Phase 1: Foundation

## What is this project?

This project builds an **AI coding agent** from scratch, step by step, in Python.

An AI coding agent is a program you talk to in the terminal. It uses an AI model (an LLM) to answer
questions and, in later phases, to read files, write code and run commands for you.
Tools like Claude Code and Cursor work this way.

The goal is to **learn how AI agents really work inside**. We use no big frameworks, so you can read every line.

---

## What do we build in Phase 1?

In Phase 1 we build the **base** that every later phase sits on:

1. **A chat window in the terminal.** You type a question, and the AI answers.
2. **A connection to a local AI model.** We use [Ollama](https://ollama.com), which runs the model
   on your own computer. It's free, needs no API key, and your data stays on your machine.
3. **Live (streaming) answers.** The answer appears word by word as the model writes it,
   so you don't wait for the full answer.
4. **Memory of the conversation.** The agent remembers what you said earlier in the same chat.
5. **An inspector.** The `/inspect` command shows exactly what was sent to the AI model and
   what came back. This is the best way to see how a chat with an LLM really works.

The agent **cannot use tools yet**: it can only talk. Tools come in Phase 2.
But we already prepare the code for tools, so Phase 2 is easy to add.

---

## How to run it

**You need:**
- Python 3.10 or newer
- [Ollama](https://ollama.com/download) installed and running

**Steps:**

```bash
# 1. Download the AI model (only once, about 4.7 GB)
ollama pull qwen2.5:7b-instruct

# 2. Create a virtual environment and install the Python package
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Start the agent
python -m agent.cli
```

**Commands you can type in the chat:**

| Command | What it does |
|---------|--------------|
| `/help` | Shows all commands |
| `/model` | Shows which AI model is being used |
| `/clear` | Forgets the conversation and starts fresh |
| `/inspect` | Shows every message sent to the AI and every answer that came back |
| `/clear-inspect` | Clears the inspector's records |
| `/quit` | Exits the program (`Ctrl+C` also works) |

---

## Folder structure

```
phase-1-foundation/
├── requirements.txt        Python packages we need (only "ollama")
├── .gitignore              Files git should not save
└── agent/
    ├── cli.py              The chat program — the starting point
    ├── core/
    │   ├── config.py       Settings: which model, where Ollama runs
    │   ├── streaming.py    Prints the answer word by word
    │   └── inspector.py    Records every AI call for /inspect
    └── providers/
        ├── base.py         The "rules" every AI provider must follow
        ├── registry.py     Picks which provider to use
        └── ollama.py       Talks to Ollama
```

---

## Architecture (how the parts fit together)

The project has **three layers**. Each layer has one job:

```
┌─────────────────────────────────────────────────┐
│  1. CLI layer            agent/cli.py           │   Talks to YOU
│     reads your input, shows answers, commands   │
└────────────────────────┬────────────────────────┘
                         │
┌────────────────────────▼────────────────────────┐
│  2. Core layer           agent/core/            │   Helpers
│     config · streaming · inspector              │
└────────────────────────┬────────────────────────┘
                         │
┌────────────────────────▼────────────────────────┐
│  3. Provider layer       agent/providers/       │   Talks to the AI MODEL
│     base · registry · ollama                    │
└────────────────────────┬────────────────────────┘
                         │
                 ┌───────▼───────┐
                 │ Ollama server │   runs the model on your computer
                 │ (qwen2.5 7B)  │
                 └───────────────┘
```

**Why split it like this?**

- The **CLI** doesn't know which AI company or model is used. It only calls `provider.chat_stream()`.
- The **provider** hides all the Ollama-specific details.
- So, to switch to another model service later (for example Claude or OpenAI), you only write
  **one new provider file**. Nothing else changes.

---

## Data flow: what happens, step by step

Say you type: **`What is Python?`**

```
 You type "What is Python?"
        │
        ▼
 ① cli.py reads your input
        │
        ▼
 ② Is it a command (/help, /quit ...)? ── yes ──▶ run the command, wait for next input
        │ no
        ▼
 ③ inspector marks "a new turn started"
        │
        ▼
 ④ cli.py adds your message to the conversation list
        │
        ▼
 ⑤ cli.py calls provider.chat_stream(messages, system prompt)
        │
        ▼
 ⑥ ollama.py puts the system prompt at the front of the messages
        │
        ▼
 ⑦ ollama.py sends everything to the Ollama server
        │
        ▼
 ⑧ Ollama sends the answer back in small pieces (chunks)
        │
        ▼
 ⑨ ollama.py passes each piece on as a "text" event
        │
        ▼
 ⑩ streaming.py prints each piece on screen right away
        │
        ▼
 ⑪ When the model finishes, ollama.py sends a "done" event with the full answer
        │
        ▼
 ⑫ inspector saves what was sent and what came back
        │
        ▼
 ⑬ cli.py adds the AI's answer to the conversation list
        │
        ▼
 ⑭ Wait for your next message ── and repeat from ①
```

### The steps in more detail

**① Read your input.** `cli.py` waits for you to type something and press Enter. Empty lines are skipped.

**② Check for commands.** If the text starts with `/` and matches a known command, `cli.py` handles it
directly. **The AI is not called** for commands.

**③ Start a new turn.** `cli.py` tells the inspector "the user asked something new". This is only used to
group records in `/inspect`.

**④ Save your message.** Your message is added to a Python list called `messages`:

```python
[{"role": "user", "content": "What is Python?"}]
```

This list is the agent's **memory**. Every time, we send the **whole list** to the model,
because the model itself remembers nothing between calls.

**⑤ Ask the provider.** `cli.py` calls `provider.chat_stream(...)` with the message list and a
**system prompt**. The system prompt is a hidden instruction that tells the model how to behave
("You are a helpful AI coding agent...").

**⑥ Add the system prompt.** Ollama expects the system prompt as the first message, so `ollama.py` builds:

```python
[
  {"role": "system", "content": "You are a helpful AI coding agent..."},
  {"role": "user",   "content": "What is Python?"}
]
```

**⑦ Send to Ollama.** `ollama.py` sends this list to the Ollama server at `http://localhost:11434`
with `stream=True`, which means "send me the answer in pieces".

**⑧–⑨ Receive pieces.** The model writes one piece at a time ("Python", " is", " a", ...).
For each piece, `ollama.py` sends out a small event:

```python
{"type": "text", "content": " is"}
```

**⑩ Print live.** `streaming.py` receives each event and prints it immediately. This is why you see the
answer being "typed".

**⑪ Done.** When the model finishes, `ollama.py` sends one last event with the full answer:

```python
{"type": "done", "response": {"role": "assistant", "content": "Python is a ..."}}
```

**⑫ Record for the inspector.** `ollama.py` gives the inspector the exact input and output of this call.

**⑬ Save the answer.** `cli.py` adds the answer to `messages`. Now the list has 2 messages.
Next time you ask something, the model sees both, so it can understand follow-up questions like
"give me an example".

**If something goes wrong** (for example, Ollama isn't running), `cli.py` prints the error and removes
your last message from the list, so the conversation stays clean.

---

## What each file does

### `agent/cli.py` — the chat program

This is the **starting point**. When you run `python -m agent.cli`, the `main()` function in this file starts.

- Loads the settings from `config.py`.
- Gets the AI provider from `registry.py`.
- Prints a welcome banner.
- Runs a loop forever: read your input → handle commands → or send it to the AI.
- Keeps the conversation in the `messages` list.
- Holds the **system prompt** (`SYSTEM_PROMPT`), the hidden instruction for the model.
- The `chat()` function sends the messages to the provider, prints the streamed answer, and saves it.

### `agent/core/config.py` — settings

A very small file with the project's settings:

- `MODEL = "qwen2.5:7b-instruct"`: which AI model to use.
- `OLLAMA_HOST = "http://localhost:11434"`: where the Ollama server runs.

`get_config()` returns these as a dictionary. **To try a different model, change `MODEL` here**
(and run `ollama pull <model-name>` first).

### `agent/core/streaming.py` — live printing

Has one function, `print_stream()`. It takes the events from the provider one by one:

- **`text` event** → print it on the screen straight away.
- **`tool_call` event** → ignored for now (tools come in Phase 2).
- **`done` event** → keep the full answer and return it to `cli.py`.

### `agent/core/inspector.py` — the "X-ray" tool

Records everything that happens so you can study it with `/inspect`. It saves three kinds of records:

| Record | When it's saved | What it holds |
|--------|-----------------|---------------|
| `turn` | Each time you ask something | Your input |
| `llm` | Each time the AI model is called | The exact messages sent, the tools offered, and the answer |
| `tool` | Each time a tool runs (from Phase 2) | Tool name, inputs and result |

`/inspect` prints all records as JSON, so you see the **real data format** that goes to the model.
Turns are shown in alternating colors (orange and pink) so they're easy to tell apart.
`/clear-inspect` deletes the records.

### `agent/providers/base.py` — the rules for providers

A **provider** is the code that talks to one AI service. `base.py` defines what every provider
**must** have:

- `chat()`: send messages, get the full answer back at once.
- `chat_stream()`: send messages, get the answer back piece by piece.
- `name`: a readable name, like `Ollama (qwen2.5:7b-instruct)`.

It also fixes **one common message format** (`{"role": ..., "content": ...}`) used across the whole project.
Each provider converts this format to whatever its AI service needs.

### `agent/providers/registry.py` — picks the provider

Has one function, `get_provider(config)`. It reads the settings and creates the right provider.
Right now it always creates the Ollama provider. Later, you can add more choices here,
for example: "if the setting says `claude`, create the Claude provider".

### `agent/providers/ollama.py` — talks to Ollama

The only real provider in Phase 1. Its parts:

- **`__init__`**: connects to the Ollama server.
- **`chat_stream()`**: sends the messages with `stream=True`, sends out a `text` event for every piece,
  then a `done` event with the full answer, and records the call in the inspector.
- **`chat()`**: the same, but waits for the full answer (no streaming).
- **`_inject_system()`**: puts the system prompt at the front of the message list.
- **`_convert_tools()`**: turns our tool descriptions into the format Ollama expects (ready for Phase 2).
- **`_parse_response()`**: turns Ollama's answer into our common format.

### `requirements.txt`

Lists the Python packages to install. Phase 1 needs only one: `ollama` (the official Ollama Python library).

### `.gitignore`

Tells git which files **not** to save, such as `__pycache__/`, `venv/` and other generated files.

---

## Key ideas you learn in Phase 1

1. **An LLM has no memory.** We send the full conversation every time. "Memory" is just a Python list.
2. **The system prompt** is a hidden first message that shapes how the model behaves.
3. **Streaming** makes the agent feel fast: show the answer while it's being written.
4. **A provider layer** keeps the code independent of any one AI company.
5. **Inspecting the real input and output** is the best way to understand and debug an AI agent.

---

## What's next: Phase 2

In Phase 2 we add the **agent loop**: the model can ask to run a tool (like a terminal command),
the agent runs it, sends the result back, and the model continues. This turns a chatbot into an agent.
