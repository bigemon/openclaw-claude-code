# OpenClaw Claude Code Skill 🤖

*Turn Anthropic's Claude Code CLI into a programmatic, headless coding engine for AI agents.*

[![NPM Version](https://img.shields.io/npm/v/openclaw-claude-code-skill.svg)](https://www.npmjs.com/package/openclaw-claude-code-skill)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

## 💡 Why does this exist?

Anthropic's `claude-code` is an incredibly powerful coding tool, but it's designed as an interactive CLI for **humans**. It relies on TTY prompts, terminal ANSI escapes, and interactive permission approvals. 

If you are building an AI agent (like [OpenClaw](https://github.com/openclaw/openclaw)) and want it to delegate complex software engineering tasks to Claude Code, you can't easily pipe data into it. And if you try to pass large prompts via standard CLI arguments (`claude -p "long text"`), you quickly hit **OS `ARG_MAX` limitations** and your app crashes.

**This project solves that by wrapping Claude Code in a robust Client-Server architecture.** It exposes Claude Code's capabilities via a clean API and a stateless CLI, allowing other AI agents or automated scripts to spawn, monitor, and steer Claude Code sessions programmatically.

## 🏗️ Architecture & How It Works

This skill is split into two components:

1. **The Backend Server (Express):** 
   - Spawns and manages long-lived `claude-code` processes in the background.
   - Communicates with Claude Code using the native `--input-format stream-json` protocol via `stdin`. **This completely bypasses OS `ARG_MAX` limitations**, allowing you to send massive prompts (entire files, plans, or codebase context) without crashing.
   - Manages Session TTLs, context compaction, and **Graceful Shutdowns** (prevents zombie Claude processes when the server restarts).
   - Exposes a REST & Server-Sent Events (SSE) API.

2. **The Frontend CLI (`claude-code-skill`):**
   - A thin, stateless wrapper that your Agent (or CI/CD pipeline) calls.
   - Streams responses back in real-time or machine-readable `NDJSON`.
   - Supports advanced routing: effortlessly point Claude Code to Gemini, OpenAI, or local models via proxy URLs.

## ✨ Core Features

- 🔌 **API-First Design** — Drive Claude Code programmatically without messy TTY emulators.
- 💾 **Persistent Sessions** — Start a session, send a task, disconnect, and resume later. Context is maintained.
- 🛡️ **No `ARG_MAX` Limits** — Send prompts of any size thanks to the `stdin` stream-json architecture.
- 🧠 **Effort & Thinking Control** — Full support for `--effort` flags (`low`/`medium`/`high`/`max`) and `auto` permission modes.
- 📋 **Plan Mode** — Have Claude write a plan (`--plan`) before executing to avoid costly trial-and-error.
- 📊 **Context & Cost Tracking** — Inspect token usage, cache hit rates, and exact session costs in real-time.
- 🌐 **Multi-Model Routing** — Route Claude Code's agent loop through any OpenAI-compatible endpoint (Gemini 2.0 Flash, GPT-4o, etc.).
- 🔀 **Session Branching** — Fork a session and change its model/effort mid-flight.
- 🪝 **Webhook Callbacks** — Get HTTP callbacks for tool errors, context limits, or session completion.

---

## 🚀 Installation & Setup

You need to run the backend server first, then you can use the CLI.

### 1. Start the Backend API Server

```bash
git clone https://github.com/Enderfga/openclaw-claude-code-skill.git
cd openclaw-claude-code-skill/backend

# Install and build
npm install
npm run build

# Set your API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Run the daemon (starts on port 18795 by default)
./start.sh --daemon
```

*For more backend options (like Amazon Bedrock support), see the [Backend README](backend/README.md).*

### 2. Install the Frontend CLI

```bash
cd .. # Back to the project root

npm install
npm run build
npm link  # Optional: makes `claude-code-skill` globally available
```

---

## 🛠️ Quick Start

```bash
# Start a session in your project directory
# auto permission mode skips interactive prompts safely
claude-code-skill session-start myproject -d ~/my-project \
  --permission-mode auto \
  --allowed-tools "Bash,Read,Edit,Write,Glob,Grep" \
  --effort high

# Send a task with streaming output
claude-code-skill session-send myproject "Refactor the auth module to use JWT" --stream

# Need deep reasoning? Use ultrathink
claude-code-skill session-send myproject "Design a scalable caching layer" --stream --ultrathink

# Check context usage (tokens)
claude-code-skill session-context myproject

# Check exact cost (Input/Output/Cached breakdown)
claude-code-skill session-cost myproject

# Stop the session
claude-code-skill session-stop myproject
```

---

## 📚 Advanced Workflows

### Effort & Plan Mode
Save tokens and time by using the right effort level and asking Claude to plan first.

```bash
# Low effort for quick linting/formatting
claude-code-skill session-send myproject "Fix lint errors" --effort low

# Create a plan before executing (great for complex features)
claude-code-skill session-send myproject "Add Redis rate limiting" --plan
```

### Multi-Model Support 🌐
Want to run the Claude Code tool-loop with a different model? You can override the model and base URL:

```bash
# Use Gemini 2.0 Flash via an OpenAI-compatible proxy
claude-code-skill session-start gemini-task -d ~/project \
  --model gemini-2.0-flash \
  --base-url http://127.0.0.1:8082
```

### Branching & Model Switching
Fork an active session to try an experimental approach, or switch to a cheaper model once the hard architecture work is done.

```bash
# Fork 'main' into 'experiment' and switch to a cheaper model
claude-code-skill session-branch main experiment --model sonnet --effort low
```

### Webhooks (Event Hooks)
Perfect for CI/CD or agent orchestrators. Register URLs to be hit when events happen.

```bash
claude-code-skill session-hooks myproject \
  --on-tool-error http://localhost:8080/webhook \
  --on-stop http://localhost:8080/webhook
```

### NDJSON for Automation
If your agent needs to parse Claude Code's output programmatically, use `--ndjson`.

```bash
claude-code-skill session-send myproject "Run tests" --stream --ndjson

# {"type":"tool_use","tool":"Bash","input":"npm test"}
# {"type":"tool_result"}
# {"type":"done","text":"All tests pass.","stop_reason":"end_turn"}
```

---

## 🛣️ Roadmap

### ✅ Completed (v1.2 & v1.3)
- [x] Dedicated Express backend to bypass OS `ARG_MAX` limits via `stream-json`.
- [x] Graceful Shutdowns (SIGTERM handling to prevent zombie processes).
- [x] Effort control (`low`/`medium`/`high`/`max`/`auto`) & Ultrathink.
- [x] Session cost tracking with token/price breakdown (`session-cost`).
- [x] Session branching (`session-branch`) — fork + model/effort change in one step.
- [x] Hook system (`session-hooks`) — webhook callbacks for automation.
- [x] Model alias resolution & custom model overrides (`--model-overrides`).
- [x] Auto-resume stopped sessions (`--auto-resume`).
- [x] NDJSON streaming output.

### 🔜 Future
- [ ] MCP elicitation support (needs human-in-the-loop, currently not useful for headless agents).
- [ ] Direct Docker container isolation per session.

## 🤝 Contributing
Contributions, issues, and feature requests are welcome! If you're building agentic workflows on top of this, let us know.

## 📄 License
MIT
