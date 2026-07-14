---
title: "Manually Setting Up a Japanese Light Novel Writer Agent"
description: "Setting Up the Japanese Light Novel Writer Agent using docker and mcp"
category: "Docker & Agents"
pubDate: 2026-07-12
difficulty: "intermediate"
---

This is a step-by-step walkthrough of everything `install.sh` does for you, explained one piece at a time, so you understand what's being installed and why before you run a single command. It targets macOS and Linux.

If you'd rather not type each command yourself, `install.sh` automates Steps 1, 2, 4, and 5 below — each section notes this and shows the script as the alternative.

---

## Contents

- [Project description](#project-description)
- [System requirements](#system-requirements)
- [NVIDIA API keys](#nvidia-api-keys)
- [Understanding the stack](#understanding-the-stack)
- [Features](#features)
- [Modules and scripts](#modules-and-scripts)
- [Project structure](#project-structure)
- [Manual setup, step by step](#manual-setup-step-by-step)
- [Configuring your API key (`.env`)](#configuring-your-api-key-env)
- [Usage guide](#usage-guide)
- [Troubleshooting](#troubleshooting)

---

## Project description

This project is a self-hosted writing assistant for Japanese light novel–style chapters. It runs an OpenAI-compatible HTTP service backed by either a local Ollama model or NVIDIA's cloud models, with retrieval-based memory over your own saved chapters and a full browser chat UI.

**Everything application-level runs inside Docker.** That's a deliberate choice: Docker guarantees the exact same Python version, exact same pinned dependency versions, and exact same container filesystem layout whether you're on macOS or Linux — you're not debugging "works on my machine" differences between an
Intel Mac, an Apple Silicon Mac, and a Linux server.

The only two things that run natively on your host, outside Docker, are Docker itself and Ollama (Ollama needs direct hardware access for performance, so containerizing it would work against you).

---

## System requirements

- **OS:** macOS or Linux (x86_64 or arm64). Windows is untested but should work under WSL2 with Docker Desktop.
- **Docker:** Docker Desktop (macOS) or Docker Engine + the Compose plugin (Linux).
- **Ollama:** installed natively on the host, not containerized.
- **Disk space:** a few GB for Docker images, plus Ollama model weights (`llama3.1` ≈ 4.7 GB, `nomic-embed-text` ≈ 275 MB).
- **Global packages:** none beyond Docker and Ollama — no system Python packages are required; everything Python-related is pinned and installed inside the container image.
- **Optional:** `curl` (used throughout this tutorial to verify each step) and `git` (to clone/update the project).

---

## NVIDIA API keys

The local model needs no API key. To use the two NVIDIA cloud models, sign up for a free account at
**[build.nvidia.com](https://build.nvidia.com/models?modal=signin)**, then generate a key against either of these model pages (one account-level key works for both):

- [z-ai/glm-5.2](https://build.nvidia.com/z-ai/glm-5.2)
- [google/gemma-4-31b-it](https://build.nvidia.com/google/gemma-4-31b-it)

You'll put this key in `.env` in Step 4 below.

---

## Understanding the stack

Each of the following is a real, running piece of this system — not an aspirational architecture diagram. Read through these once so the manual steps later make sense in context.

### 1. Inference Engine (Ollama + NVIDIA NIM)
[Ollama](https://ollama.com/) runs the local model directly on your machine's CPU/GPU — no network round-trip, works offline, no per-token cost. NVIDIA's cloud API ([NIM](https://build.nvidia.com/)) gives you access to larger, hosted models over HTTPS when you want higher quality output and don't mind the latency or (free-tier) usage limits. Both are spoken to through the exact same OpenAI-compatible client code — the agent doesn't need separate logic per backend.

### 2. Models
`llama3.1` runs locally through Ollama. `z-ai/glm-5.2` and `google/gemma-4-31b-it` run in NVIDIA's cloud. A fourth model, `nomic-embed-text`, also runs locally through Ollama — it's not a chat model, it converts text into vectors for the memory system (see item 5).

### 3. Application Layer (FastAPI)
[FastAPI](https://fastapi.tiangolo.com/) serves an HTTP API that mimics OpenAI's own API shape (`/v1/chat/completions`, `/v1/models`). This is deliberate: any tool built to talk to OpenAI's API (like Open WebUI) can talk to this agent with zero code changes on the client side — you just point its base URL here instead.

### 4. Containerization (Docker + Docker Compose)
Docker packages the application layer and its exact dependency versions into a portable image that runs identically everywhere. Docker Compose then coordinates *multiple* containers together — in this project, the agent and the UI — handling their networking, startup order, and health checks as one unit instead of you managing each container by hand.

### 5. Memory & Retrieval — RAG (Chroma)
[Chroma](https://www.trychroma.com/) is a vector database. When you save a chapter, its text is split into chunks and each chunk is converted into a vector (via `nomic-embed-text`) and stored. When you chat, your message is converted into a vector the same way, and Chroma finds the most similar stored chunks — those get inserted into the prompt as context. This is how the agent "remembers" your story without you having to paste previous chapters into every message. It runs embedded, in-process, inside the agent's own container — there's no separate database server to install, configure, or keep running.

### 6. Persistence (bind mounts)
Every folder that holds data you'd care about losing — chapters, chat transcripts, the vector store, application logs, the UI's own saved state — is bind-mounted from a folder on your host into the container. This means that data physically lives on your filesystem, in this project folder, and survives container rebuilds, restarts, and even `docker compose down` — right up until you delete the files yourself.

### 7. Logging
Every log line is written to two places at once: your terminal (via `docker compose logs`) and a rotating file on disk (`logs/agent.log`, capped at 5MB × 3 backups) — so you always have a persistent, greppable record even after a container restarts and its console output is gone.

### 8. Interface — UI (Open WebUI)
[Open WebUI](https://github.com/open-webui/open-webui) is a full browser chat interface — the same kind of experience as a commercial chat product, but pointed at your own agent. It reads the agent's `/v1/models` endpoint
to populate its model dropdown, so you can pick the local model or either cloud model per conversation, and switch mid-conversation without restarting anything.

---

## Features

- OpenAI-compatible chat API (`/v1/chat/completions`), streaming and non-streaming.
- Runtime model switching (local ↔ cloud) with no redeploy.
- Retrieval-augmented memory over your own saved chapters.
- Human-readable Markdown chat transcripts.
- Full browser chat UI via Open WebUI.
- Fully offline-capable with the local model; cloud models are opt-in.
- All persistent data lives on your host filesystem, not trapped in a container.
- Container health checks and startup ordering.
- Scripted install/uninstall with a full audit trail.

---

## Modules and scripts

| File                    | Function |
|-------------------------|----------|
| `app/server.py`         | FastAPI app. Defines `/health`, `/v1/models`, `/chapters` (GET+POST), and `/v1/chat/completions` (streaming + non-streaming). Wires memory and chat history together. |
| `app/models.py`         | Model registry. Maps a model name to an OpenAI-compatible client + backend model id; resolves API keys from the environment at call time, never stores them. |
| `app/memory.py`         | Embedded Chroma RAG: chunks and embeds chapter files, indexes them, and retrieves the top-k most relevant chunks for a given query. |
| `app/history_manager.py`| Appends every chat turn to a timestamped Markdown transcript under `chat_history/`. |
| `app/logging_config.py` | Shared logger: writes to stdout and to a rotating file under `logs/`. |
| `app/Dockerfile`        | Builds the agent service's container image (python:3.11-slim + pinned dependencies). |
| `docker-compose.yml`    | Orchestrates the agent and open-webui services: networking, health checks, and bind mounts. |
| `install.sh`            | Automated installer: checks/installs Docker and Ollama, pulls required models, sets up `.env`, and optionally starts the stack. |
| `uninstall.sh`          | Reverses `install.sh` using its own install manifest. Leaves all your project data untouched. |

---

## Project structure

```
jp-ln-writer/
├── app/
│   ├── server.py
│   ├── models.py
│   ├── memory.py
│   ├── history_manager.py
│   ├── logging_config.py
│   ├── Dockerfile
│   └── requirements.txt
├── chapters/                   # your story files — bind-mounted
├── chat_history/               # chat transcripts — bind-mounted
├── chroma_db/                  # vector store — bind-mounted, self-rebuilding
├── logs/                       # agent.log — bind-mounted
├── open-webui-data/            # Open WebUI's own state — bind-mounted
├── docker-compose.yml
├── .env   			# you create this in Step 4 by change env.example file                     
├── install.sh
├── uninstall.sh
├── system-design-lightweight.md
└── README.md
```

---

## Manual setup, Step by Step

Run these in order. Each step names the exact commands for macOS and Linux where they differ.

### Step 1 — Install Docker

**macOS**, via Homebrew:
```bash
brew install --cask docker
open /Applications/Docker.app   # first launch is manual; wait for it to say "running"
```
**Linux** (Debian/Ubuntu/Fedora and derivatives), via Docker's official
convenience script:
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # log out and back in for this to take effect
```
**Verify (both OSes):**
```bash
docker compose version
```

*Scripted alternative:* `install.sh` detects your OS, checks whether Docker is already installed and running, and — only with your explicit go-ahead — installs it using the same commands shown above.

### Step 2 — Install Ollama and pull the required models

**macOS**, via Homebrew:
```bash
brew install ollama
ollama serve &                 # or just open the Ollama app
```
**Linux**, via Ollama's official install script:
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
**Pull the two models this project needs (both OSes):**
```bash
ollama pull llama3.1
ollama pull nomic-embed-text
```
`llama3.1` is the chat model; `nomic-embed-text` is required separately for the memory/retrieval system (Step 5 of "Understanding the stack" above) — without it, chat still works, but chapter retrieval silently returns no context.

**Verify:**
```bash
ollama list
```

*Scripted alternative:* `install.sh` installs Ollama and pulls both models for you, after asking permission and showing you exactly what it's about to pull.

### Step 3 — Get the project files

```bash
git clone <your-fork-or-remote-url> jp-ln-writer
cd jp-ln-writer
```
(Or copy the project folder directly if you're not using git.) `install.sh` and `uninstall.sh` both expect to be run from inside this directory.

### Step 4 — Create your `.env` file

```bash
cp env.example .env
```
Open `.env` in any text editor and set your NVIDIA key (skip this if you're only using the local model):
```
NVIDIA_API_KEY=nvapi-your-real-key-here
```
See [Configuring your API key](#configuring-your-api-key-env) below for more detail on this file specifically.

*Scripted alternative:* `install.sh` creates `.env` from the template for you and prompts you for the key value interactively (or lets you skip it and edit the file yourself later).

### Step 5 — Build and start the stack

```bash
docker compose up --build -d
```
This builds the `agent` image from `app/Dockerfile`, pulls the official `open-webui` image, and starts both. Docker Compose's `depends_on: condition: service_healthy` means `open-webui` won't start until `agent` passes its health check — so you never hit the UI before the backend it needs is actually ready.

*Scripted alternative:* `install.sh` can run this final command for you at the end of the install, after confirming with you.

### Step 6 — Verify everything is working

```bash
docker compose ps                          # both services should show "healthy"/"running"
curl http://localhost:8000/health           # {"status":"ok"}
curl http://localhost:8000/v1/models        # lists all three registered models
```
Then open **http://localhost:8080** in a browser. On first launch, Open WebUI will ask you to create a local admin account (its own onboarding — nothing this project configures) — do that, then confirm the model dropdown
lists `llama3.1-local`, `glm-5.2`, and `gemma-4-31b-it`, and that sending a message streams a reply back.

You're done — the same result as running `install.sh` end to end.

---

## Configuring your API key (`.env`)

`.env` holds one thing: `NVIDIA_API_KEY`. It's read by `docker-compose.yml`'s `env_file: .env` directive, which passes it into the `agent` container's environment at startup — `app/models.py` reads it from there at request time, never hardcoding or logging it.

1. `cp env.example .env` (only needed once — `env.example` is the safe-to-commit template; `.env` is your real, gitignored copy).
2. Edit `.env`:
   ```
   NVIDIA_API_KEY=nvapi-your-real-key-here
   ```
3. Environment variables are only read when a container **starts** — if the agent is already running, apply a changed key with:
   ```bash
   docker compose restart agent
   ```
4. Never commit `.env`. It's already listed in `.gitignore`; double-check with `git status` that it doesn't show up as a tracked/staged file before any commit.

---

## Usage guide

- **Start:** `docker compose up --build -d`
- **Stop:** `docker compose down` (data in bind-mounted folders is untouched)
- **Restart just the agent (e.g. after editing `.env`):** `docker compose restart agent`
- **View logs:** `docker compose logs -f agent` or `tail -f logs/agent.log`
- **Update:** `git pull && docker compose up --build -d`
- **Check health:** `docker compose ps` / `curl localhost:8000/health`
- **List models:** `curl localhost:8000/v1/models`
- **Chat via the API directly:**

```bash
curl -N -X POST localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"llama3.1-local","stream":true,"messages":[{"role":"user","content":"Write the opening line of a light novel."}]}'
```

**Save a chapter:**
```bash
curl -X POST localhost:8000/chapters \
  -H 'Content-Type: application/json' \
  -d '{"filename":"chapter_01.md","content":"Your chapter text here..."}'
```

**Uninstall:** `./uninstall.sh` — reverses only what `install.sh` installed; never touches your chapters, history, memory, logs, UI state, or Ollama models.

---

## Troubleshooting

**`MissingAPIKeyError` for a cloud model** 

`.env` doesn't have a real value, or the container started before you set it. Fix the value, then `docker compose restart agent`.

**`docker compose up` fails pulling `ghcr.io/open-webui/open-webui:main` with a DNS-style "no such host" error**

DNS resolution problem on your machine, not a project bug. Try, in order: `nslookup ghcr.io` in a plain terminal (fails too → system DNS issue); restart Docker Desktop fully; check Docker Desktop → Settings → Resources → Network for a stuck DNS override; try a different network if you're behind a restrictive VPN (`ghcr.io` is sometimes blocked even when `docker.io` isn't).

**Agent can't reach Ollama (`connection refused` via `host.docker.internal`)**

Confirm Ollama is actually running natively on your host — it's deliberately not containerized. On Linux, confirm `extra_hosts: host-gateway` is still present in `docker-compose.yml`.

**Chat works but never seems to use earlier chapters**

Chapters must be saved through `POST /chapters` (or dropped as `.md`/`.txt` directly into `chapters/`) before they're searchable. Also confirm `nomic-embed-text` is pulled (`ollama list`) — without it, retrieval degrades silently to no context.

**`logs/agent.log` exists but is empty**

Normal right after a fresh start — the file is created at boot, before anything has logged a line. Send one chat request and check again.

**Port 8000 or 8080 already in use**

Something else on your machine is bound to that port. Stop it, or remap the left-hand side in `docker-compose.yml`'s `ports:` (e.g. `"8001:8000"`).

**`docker compose config` prints my real key to the terminal**

Expected — it shows the fully resolved config Docker will actually use, which includes anything from `.env`. Not a leak; just don't paste, log, or screenshot that specific command's output while a real key is loaded.

**Starting completely fresh**

This is a manual, opt-in reset — a command *you* choose to run. Neither `install.sh` nor `uninstall.sh` ever deletes any of these files on their own:

```bash
docker compose down
rm -f chapters/*.md chapters/*.txt chat_history/*.md logs/*.log*
rm -rf chroma_db/*
docker compose up --build -d
```
This also never touches Docker, Ollama, or anything `install.sh` set up at the OS level — it only clears the four generated-data folders you just told it to, and only because you ran the `rm` commands yourself.

---

## Project Files

You can download the complete files used in this tutorial from the repository below:

<a class="btn-github" href="https://github.com/ecellsworth/jp-ln-writer">View repository on GitHub</a>
