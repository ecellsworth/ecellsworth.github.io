---
title: "Manually Setting Up a Japanese Light Novel Writer Agent"
description: "Setting Up the Japanese Light Novel Writer Agent using docker and mcp"
category: "Docker & Agents"
pubDate: 2026-07-12
difficulty: "intermediate"
---

This is a complete, from-nothing walkthrough: every directory, every source file (with its full real contents), and every command needed to build and run this project, with no assumption that a repository already exists anywhere. It targets macOS and Linux.

Steps 3–6 (creating the project's directories and source files) have no scripted equivalent — `install.sh` assumes those files already exist and only automates the setup *around* them. Once the files exist, `install.sh` can automate Steps 1, 2, 7, and 8 below — each of those sections notes this and shows the script as the alternative.

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

You'll put this key in `.env` in Step 7 below.

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
├── .env                         # you create this in Step 7, from env.example
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

### Step 3 — Create the project directory structure

```bash
mkdir -p jp-ln-writer/app
mkdir -p jp-ln-writer/chapters
mkdir -p jp-ln-writer/chat_history
mkdir -p jp-ln-writer/chroma_db
mkdir -p jp-ln-writer/logs
mkdir -p jp-ln-writer/open-webui-data
cd jp-ln-writer
```
The five data directories (`chapters/`, `chat_history/`, `chroma_db/`, `logs/`, `open-webui-data/`) are bind-mount targets — Docker Compose would create them automatically on first run anyway, but creating them now makes the structure explicit. Everything from here on assumes your working directory is this `jp-ln-writer/` folder.

### Step 4 — Create the application code (`app/`)

Seven files, all inside `app/`. Order doesn't matter for creating them (nothing executes until Step 8), but this order matches their dependency chain, simplest first.

**`app/__init__.py`** — empty; it just marks `app/` as a Python package.
```bash
touch app/__init__.py
```

**`app/logging_config.py`** — shared logger: writes to stdout and to a rotating file under `logs/`.

<details>

```python
"""
Shared logging setup.

Every module-level logger created via get_logger() writes to both stdout
(container logs, `docker compose logs agent`) AND a rotating file under the
bind-mounted logs/ directory (LOG_DIR, default /app/logs). Before this,
`logs/` was bind-mounted (docker-compose.yml, Stage 2) but nothing actually
wrote files into it -- app log output only ever reached Docker's own log
driver, which isn't the same guarantee as a real file on the host. Added per
the requirement that chat logs, documents, db files, and anything else
persistent live on the local machine, not trapped in the container.
"""

import logging
import os
from logging.handlers import RotatingFileHandler
from pathlib import Path

LOG_DIR = os.getenv("LOG_DIR", "/app/logs")
LOG_FILE_NAME = "agent.log"
MAX_BYTES = 5 * 1024 * 1024  # 5 MB per file
BACKUP_COUNT = 3  # agent.log, agent.log.1, agent.log.2, agent.log.3

_configured = set()


def get_logger(name: str) -> logging.Logger:
    """Return a logger configured to write to stdout and LOG_DIR/agent.log.

    Idempotent -- calling this more than once for the same name (e.g. across
    module reloads in a test) does not add duplicate handlers.
    """
    logger = logging.getLogger(name)
    if name in _configured:
        return logger

    formatter = logging.Formatter("%(asctime)s [%(name)s] %(levelname)s %(message)s")

    stream_handler = logging.StreamHandler()
    stream_handler.setFormatter(formatter)
    logger.addHandler(stream_handler)

    try:
        Path(LOG_DIR).mkdir(parents=True, exist_ok=True)
        file_handler = RotatingFileHandler(
            Path(LOG_DIR) / LOG_FILE_NAME,
            maxBytes=MAX_BYTES,
            backupCount=BACKUP_COUNT,
            encoding="utf-8",
        )
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)
    except OSError as exc:
        # A logging-setup failure (e.g. an unwritable LOG_DIR) should never
        # take the app down -- stdout logging above still works regardless.
        logger.warning(
            "could not open log file under %s (%s) -- logging to stdout only",
            LOG_DIR,
            exc,
        )

    logger.setLevel(logging.INFO)
    logger.propagate = False
    _configured.add(name)
    return logger
```

</details>

**`app/history_manager.py`** — appends every chat turn to a timestamped Markdown transcript under `chat_history/`.

<details>

```python
import os
from datetime import datetime

class ChatHistoryManager:
    def __init__(self, base_dir="."):
        self.history_dir = os.path.join(base_dir, "chat_history")
        os.makedirs(self.history_dir, exist_ok=True)
        self.session_file = os.path.join(
            self.history_dir,
            f"session_{datetime.now().strftime('%Y-%m-%d_%H%M%S')}.md"
        )

    def save(self, role: str, content: str):
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        with open(self.session_file, "a", encoding="utf-8") as f:
            f.write(f"\n### {role.upper()} [{timestamp}]\n\n")
            f.write(f"{content.strip()}\n\n")
            f.flush()

    def get_recent(self, max_lines=150):
        """Return the most recent lines from the current session for context."""
        try:
            with open(self.session_file, "r", encoding="utf-8") as f:
                lines = f.readlines()
            return "".join(lines[-max_lines:])
        except FileNotFoundError:
            return ""
        except Exception:
            return ""
```

</details>

**`app/models.py`** — model registry: maps a model name to an OpenAI-compatible client + backend model id.

<details>

```python
"""
Model registry for the lightweight Japanese Light Novel Writer Agent.

Adding a model is a dict entry, nothing else — both `local` and `cloud` backends are called through the same OpenAI-compatible client, differing only in base_url and API key. See system-design-lightweight.md, Stage 1.

Secrets (hard constraint): cloud entries name the *environment variable* that holds their API key (`api_key_env`); the key value itself is never written here, never logged, and is read from that variable only at call time in get_client_and_model().
"""

import os

from openai import OpenAI

# OLLAMA_BASE_URL is overridden by Stage 2's docker-compose.yml to
# http://host.docker.internal:11434 (the only way a container reaches the
# native host Ollama). Left at its default here so Stage 1's own
# verification step — running `uvicorn app.server:app` directly on the host,
# before any container exists — works unmodified against a local Ollama.
#
# Public (no leading underscore) as of Stage 3: app/memory.py reuses this
# exact value for its Ollama embeddings calls (nomic-embed-text), so both the
# chat model and the embedding model resolve the same host Ollama through a
# single source of truth instead of two independently-configured env reads.
OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://127.0.0.1:11434")

MODELS = {
    "llama3.1-local": {
        "type": "local",
        "model": "llama3.1",
        "base_url": f"{OLLAMA_BASE_URL}/v1",  # Ollama's OpenAI-compatible endpoint
    },
    "glm-5.2": {
        "type": "cloud",
        "model": "z-ai/glm-5.2",
        "base_url": "https://integrate.api.nvidia.com/v1",
        "api_key_env": "NVIDIA_API_KEY",
    },
    "gemma-4-31b-it": {
        "type": "cloud",
        "model": "google/gemma-4-31b-it",
        "base_url": "https://integrate.api.nvidia.com/v1",
        "api_key_env": "NVIDIA_API_KEY",
    },
}

DEFAULT_MODEL = "llama3.1-local"


class MissingAPIKeyError(Exception):
    """Raised when a cloud model is requested but its API key env var is unset."""


def get_client_and_model(model_name: str) -> tuple[OpenAI, str]:
    """Resolve a registry entry to (OpenAI-compatible client, backend model id).

    Raises KeyError for an unknown model name, MissingAPIKeyError for a cloud
    model whose api_key_env is not set in the environment.
    """
    entry = MODELS[model_name]  # KeyError on unknown model — handled by caller

    if entry["type"] == "cloud":
        api_key = os.environ.get(entry["api_key_env"])
        if not api_key:
            raise MissingAPIKeyError(
                f"Model '{model_name}' requires environment variable "
                f"'{entry['api_key_env']}' to be set (see .env)."
            )
    else:
        api_key = "ollama"  # dummy value; Ollama's /v1 endpoint ignores it

    client = OpenAI(base_url=entry["base_url"], api_key=api_key)
    return client, entry["model"]


def list_models() -> list[dict]:
    """Return all registered models in OpenAI /v1/models list-item shape."""
    return [
        {"id": name, "object": "model", "owned_by": entry["type"]}
        for name, entry in MODELS.items()
    ]
```

</details>

**`app/memory.py`** — embedded Chroma RAG: chunks and embeds chapter files, indexes them, and retrieves the top-k most relevant chunks for a given query.

<details>

```python
"""
Embedded Chroma vector memory (RAG) for the lightweight Japanese Light Novel
Writer Agent. See system-design-lightweight.md, Stage 3.

Chroma runs in-process inside the agent container (chromadb.PersistentClient,
never a separate Chroma server) writing to the bind-mounted chroma_db/.
Embeddings come from Ollama's own /api/embeddings endpoint (model
nomic-embed-text) via a direct HTTP call -- not Chroma's built-in embedding
machinery -- so the exact same embedding function is used for both indexing
and querying. That closes the design's own open question about embedding
dimensionality mismatch: there is only ever one embedder anywhere in this
codebase, so there is nothing for it to mismatch against.

This module REPLACES server.py's Stage 1 get_recent_context() ("last N lines
of the flat-file log") as the agent's memory mechanism for prompt context.
It does not run alongside it -- see system-design-lightweight.md Stage 3:
"This replaces the crude 'last N lines of the log' context builder... it
does not layer on top of it."
"""

import json
import os
import re
from pathlib import Path

import chromadb
import httpx
from chromadb.config import Settings

from .logging_config import get_logger
from .models import OLLAMA_BASE_URL

# stdout + a real rotating file under the bind-mounted logs/ directory --
# retrieval debug lines (the design's own verify requirement) are never
# trapped only in Docker's own ephemeral log driver.
logger = get_logger("memory")

CHROMA_DB_PATH = os.getenv("CHROMA_DB_PATH", "/app/chroma_db")
CHAPTERS_DIR = os.getenv("CHAPTERS_DIR", "/app/chapters")
EMBED_MODEL = "nomic-embed-text"
COLLECTION_NAME = "chapters"

# Design point left open by system-design-lightweight.md Stage 3 ("decide the
# chunking strategy (size/overlap) and k... a moderate chunk with small
# overlap and a small k (e.g. 3-5) is a reasonable first cut"). Decided here:
# character-based chunking (no extra tokenizer dependency), moderate size,
# small overlap, k within the suggested range.
CHUNK_SIZE = 800
CHUNK_OVERLAP = 100
DEFAULT_K = 4

MANIFEST_PATH = Path(CHROMA_DB_PATH) / ".indexed_manifest.json"

_client = None
_collection = None


def _get_client():
    global _client
    if _client is None:
        Path(CHROMA_DB_PATH).mkdir(parents=True, exist_ok=True)
        # anonymized_telemetry=False: Chroma's default settings phone home
        # (posthog) on every client construction. Not what the "everything
        # stays local" requirement was about (that's storage, this is
        # network egress), but directly adjacent and cheap to close while
        # auditing this -- flagged as a related, distinct finding.
        _client = chromadb.PersistentClient(
            path=CHROMA_DB_PATH,
            settings=Settings(anonymized_telemetry=False),
        )
    return _client


def _get_collection():
    global _collection
    if _collection is None:
        _collection = _get_client().get_or_create_collection(COLLECTION_NAME)
    return _collection


def embed_text(text: str) -> list[float]:
    """Embed one string via Ollama's /api/embeddings endpoint (nomic-embed-text).

    Used for both indexing chapter chunks (index_chapter) and embedding the
    live query (retrieve_context), so the vector space is always
    self-consistent.
    """
    response = httpx.post(
        f"{OLLAMA_BASE_URL}/api/embeddings",
        json={"model": EMBED_MODEL, "prompt": text},
        timeout=60.0,
    )
    response.raise_for_status()
    return response.json()["embedding"]


def chunk_text(text: str, chunk_size: int = CHUNK_SIZE, overlap: int = CHUNK_OVERLAP) -> list[str]:
    """Sliding-window character chunker.

    Simple and dependency-free -- chapter-length prose doesn't need a
    tokenizer-aware splitter for this design's scope.
    """
    text = text.strip()
    if not text:
        return []
    if len(text) <= chunk_size:
        return [text]

    chunks = []
    start = 0
    step = chunk_size - overlap
    while start < len(text):
        chunk = text[start : start + chunk_size].strip()
        if chunk:
            chunks.append(chunk)
        start += step
    return chunks


def _extract_chapter_number(filename: str) -> str | None:
    match = re.search(r"(\d+)", filename)
    return match.group(1) if match else None


def _load_manifest() -> dict:
    if MANIFEST_PATH.exists():
        try:
            return json.loads(MANIFEST_PATH.read_text(encoding="utf-8"))
        except (json.JSONDecodeError, OSError):
            return {}
    return {}


def _save_manifest(manifest: dict) -> None:
    MANIFEST_PATH.parent.mkdir(parents=True, exist_ok=True)
    MANIFEST_PATH.write_text(json.dumps(manifest, indent=2), encoding="utf-8")


def index_chapter(path: Path) -> int:
    """Chunk and embed one chapter file into the Chroma collection.

    Any existing chunks for this filename are deleted first, so re-indexing
    an edited chapter doesn't leave stale chunks behind. Returns the number
    of chunks indexed.
    """
    collection = _get_collection()
    filename = path.name

    collection.delete(where={"source": filename})

    text = path.read_text(encoding="utf-8")
    chunks = chunk_text(text)
    if not chunks:
        logger.info("%s has no indexable content, skipped", filename)
        return 0

    chapter_number = _extract_chapter_number(filename)
    ids = [f"{filename}::{i}" for i in range(len(chunks))]
    metadatas = [
        {"source": filename, "chunk_index": i, "chapter_number": chapter_number or ""}
        for i in range(len(chunks))
    ]
    embeddings = [embed_text(chunk) for chunk in chunks]

    collection.add(ids=ids, embeddings=embeddings, documents=chunks, metadatas=metadatas)
    logger.info("indexed %s (%d chunks)", filename, len(chunks))
    return len(chunks)


def save_and_index_chapter(
    filename: str, content: str, chapters_dir: str = CHAPTERS_DIR
) -> int | None:
    """Write `content` to chapters_dir/filename and index it immediately.

    This is the write path Stage 3 originally lacked entirely (see
    STAGE-3-NOTES.md, Issue 0): before this function existed, nothing in the
    codebase ever wrote files into chapters/ -- indexing only activated on
    files a user dropped in manually. chapters_dir is already bind-mounted
    (docker-compose.yml, Stage 2), so this write lands on the host
    filesystem, not inside the container's image.

    The file write always happens -- that's the primary guarantee, content
    is never lost. Indexing is best-effort: a failure there (e.g. Ollama's
    embeddings endpoint unreachable) is logged and returns None rather than
    raising, since the same file will simply get picked up by the next
    sync_chapters() call (next chat turn, or a restart). On success, the
    sync manifest is updated here too, so that follow-up sync_chapters()
    call doesn't redundantly re-index the same file a second time.
    """
    chapters_path = Path(chapters_dir)
    chapters_path.mkdir(parents=True, exist_ok=True)
    file_path = chapters_path / filename
    file_path.write_text(content, encoding="utf-8")

    try:
        chunks_indexed = index_chapter(file_path)
        manifest = _load_manifest()
        manifest[filename] = file_path.stat().st_mtime
        _save_manifest(manifest)
        return chunks_indexed
    except Exception as exc:
        logger.warning(
            "%s saved but immediate indexing failed (will retry on next sync): %s",
            filename,
            exc,
        )
        return None


def sync_chapters(chapters_dir: str = CHAPTERS_DIR) -> int:
    """Scan chapters_dir for new or modified chapter files and index them.

    This is the trigger for "when a chapter is written to chapters/, chunk
    the text and embed the chunks" (Stage 3's own wording) -- implemented as
    a manifest-compared directory scan rather than a live filesystem watcher.
    That's a deliberate implementation choice, not a substitution of the
    design's architecture: the design specifies the desired behavior (index
    on write) but not the trigger mechanism, and a live watcher (e.g. the
    `watchdog` package) would add a new dependency and a background thread
    for a single-user local tool where a several-second delay between a
    chapter appearing and it being indexed is a non-issue. See STAGE-3-NOTES
    for the full reasoning and the related open issue (nothing in this
    codebase currently writes files into chapters/ in the first place).

    Called once at server startup and once per chat request (cheap: just
    mtime comparisons against a small manifest unless something actually
    changed). Returns the number of files (re)indexed.
    """
    chapters_path = Path(chapters_dir)
    if not chapters_path.is_dir():
        return 0

    manifest = _load_manifest()
    indexed_count = 0

    current_files = {
        f.name: f.stat().st_mtime
        for f in chapters_path.iterdir()
        if f.is_file() and f.suffix.lower() in (".md", ".txt")
    }

    for filename, mtime in current_files.items():
        if manifest.get(filename) == mtime:
            continue  # unchanged since last sync
        index_chapter(chapters_path / filename)
        manifest[filename] = mtime
        indexed_count += 1

    # Drop manifest entries (and their chunks) for files that were removed
    # from chapters/ since the last sync.
    removed = set(manifest) - set(current_files)
    if removed:
        collection = _get_collection()
        for filename in removed:
            collection.delete(where={"source": filename})
            del manifest[filename]

    if indexed_count or removed:
        _save_manifest(manifest)

    return indexed_count


def retrieve_context(query: str, k: int = DEFAULT_K) -> str:
    """Embed `query` and return the top-k most relevant chapter chunks,
    formatted as a context block for prompt injection.

    This is the agent's memory mechanism (Stage 3), replacing Stage 1's
    get_recent_context().
    """
    collection = _get_collection()
    if collection.count() == 0:
        return ""

    query_embedding = embed_text(query)
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=min(k, collection.count()),
    )

    documents = results.get("documents") or [[]]
    metadatas = results.get("metadatas") or [[]]
    chunks = documents[0] if documents else []
    metas = metadatas[0] if metadatas else []

    if not chunks:
        return ""

    logger.info(
        "retrieved %d chunk(s) for query %r from sources: %s",
        len(chunks),
        query[:60],
        [m.get("source") for m in metas],
    )

    formatted = "\n\n".join(
        f"[From {m.get('source', 'unknown')}]\n{chunk}" for chunk, m in zip(chunks, metas)
    )
    return f"\n--- Relevant story context ---\n{formatted}\n--- End of context ---\n"
```

</details>

**`app/server.py`** — the FastAPI app itself. Defines `/health`, `/v1/models`, `/chapters` (GET+POST), and `/v1/chat/completions` (streaming + non-streaming); wires memory and chat history together.

<details>

```python
#!/usr/bin/env python3
"""
Japanese Light Novel Writer Agent — lightweight single-agent HTTP service.

Replaces the old bare input()/print() REPL (audit finding P0-1: a
backgrounded REPL has no terminal to type into) with a real, OpenAI-compatible
FastAPI/uvicorn service. See system-design-lightweight.md, Stages 1-3.

No orchestration framework: this handler calls the selected LLM backend
directly. Conversation continuity comes from two complementary mechanisms:
the flat-file chat history (append-only human record, preserved from the
original agent.py) and, as of Stage 3, embedded Chroma retrieval (the
agent's actual memory mechanism for prompt context — see app/memory.py).
"""

import json
import re
from contextlib import asynccontextmanager
from pathlib import Path
from typing import AsyncGenerator

from dotenv import load_dotenv
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse, StreamingResponse

from .history_manager import ChatHistoryManager
from .logging_config import get_logger
from .memory import CHAPTERS_DIR, retrieve_context, save_and_index_chapter, sync_chapters
from .models import (
    DEFAULT_MODEL,
    MODELS,
    MissingAPIKeyError,
    get_client_and_model,
    list_models,
)

load_dotenv()

# stdout + a real rotating file under the bind-mounted logs/ directory (see
# app/logging_config.py) -- previously `logger = logging.getLogger("server")`
# had no explicit handler, relying on Python's WARNING-level "last resort"
# fallback, and logs/ was bind-mounted but nothing ever wrote a file into it.
logger = get_logger("server")

SYSTEM_PROMPT = (
    "You are an experienced Japanese light novel author. "
    "Write in an engaging, vivid, emotional style typical of Japanese light novels. "
    "Use natural prose, internal monologue, and light novel pacing. "
    "Respond helpfully to the user's requests about writing scenes, characters, plots, or full chapters."
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Catch anything already sitting in chapters/ at boot, so a restart
    # doesn't require a chat turn before old chapters become retrievable.
    try:
        sync_chapters()
    except Exception as exc:
        logger.warning("startup chapter sync failed (continuing without it): %s", exc)
    yield


app = FastAPI(title="Japanese Light Novel Writer Agent (lightweight)", lifespan=lifespan)
history = ChatHistoryManager()

# NVIDIA's API catalog models expect an explicit max_tokens (confirmed against
# the user's own working run_prompt.py reference call); Ollama's OpenAI-compat
# endpoint accepts and respects it too, so one constant covers both backends.
MAX_TOKENS = 4000


def get_context(user_input: str) -> str:
    """Chroma-retrieved story context for the current user turn (Stage 3).

    Replaces Stage 1's get_recent_context() ("last N lines of the flat-file
    log") outright — this does not run alongside it, per
    system-design-lightweight.md Stage 3. Degrades to no context (rather than
    failing the whole chat request) if the embeddings call or Chroma itself
    is unavailable — e.g. host Ollama not running — logging a warning so the
    cause is visible without taking chat down over a memory-layer hiccup.
    """
    try:
        sync_chapters()  # cheap no-op unless chapters/ changed since last check
        return retrieve_context(user_input)
    except Exception as exc:
        logger.warning("memory retrieval failed, continuing without context: %s", exc)
        return ""


def _last_user_message(messages: list[dict]) -> str:
    for msg in reversed(messages):
        if msg.get("role") == "user":
            return msg.get("content", "")
    return ""


@app.get("/health")
async def health():
    return {"status": "ok"}


@app.get("/v1/models")
async def get_models():
    """OpenAI-compatible model discovery. Lists every MODELS registry entry
    (local + cloud) so a UI's model dropdown can populate without a redeploy."""
    return {"object": "list", "data": list_models()}


# Filenames only -- no path separators, no "..", so a client can't write
# outside CHAPTERS_DIR. Matches the .md/.txt extensions memory.sync_chapters()
# already scans for.
_SAFE_CHAPTER_FILENAME = re.compile(r"^[A-Za-z0-9_\-]+\.(md|txt)$")


@app.get("/chapters")
async def list_chapters():
    """List chapter files currently in chapters/ (the bind-mounted directory
    on your host, not anything inside the container image)."""
    chapters_dir = Path(CHAPTERS_DIR)
    if not chapters_dir.is_dir():
        return {"chapters": []}
    files = sorted(
        f.name for f in chapters_dir.iterdir()
        if f.is_file() and f.suffix.lower() in (".md", ".txt")
    )
    return {"chapters": files}


@app.post("/chapters")
async def save_chapter(request: Request):
    """Save a chapter to chapters/ and index it immediately.

    Resolves Stage 3's Issue 0: prior to this, nothing in the codebase wrote
    files into chapters/ at all -- Chroma indexing only ever activated on
    files a user dropped in manually. This endpoint is that missing write
    path. chapters/ is already bind-mounted (docker-compose.yml, Stage 2),
    so anything written here through this endpoint lands on your host
    filesystem, not inside the container's image or its writable layer --
    "external to the docker image" was already true structurally; this
    endpoint is what actually exercises it.
    """
    body = await request.json()
    filename = body.get("filename", "")
    content = body.get("content", "")

    if not _SAFE_CHAPTER_FILENAME.match(filename):
        raise HTTPException(
            status_code=400,
            detail="filename must match ^[A-Za-z0-9_-]+\\.(md|txt)$ -- no path separators, "
            "no '..', extension must be .md or .txt.",
        )
    if not content.strip():
        raise HTTPException(status_code=400, detail="content must not be empty.")

    chunks_indexed = save_and_index_chapter(filename, content)
    return {
        "filename": filename,
        "path": str(Path(CHAPTERS_DIR) / filename),
        "chunks_indexed": chunks_indexed,
    }


@app.post("/v1/chat/completions")
async def chat_completions(request: Request):
    body = await request.json()

    requested_model = body.get("model") or DEFAULT_MODEL
    if requested_model not in MODELS:
        requested_model = DEFAULT_MODEL

    stream = body.get("stream", True)
    messages = body.get("messages", [])

    user_input = _last_user_message(messages)
    if not user_input:
        raise HTTPException(status_code=400, detail="No user message found in 'messages'.")

    try:
        client, backend_model = get_client_and_model(requested_model)
    except MissingAPIKeyError as exc:
        # Clean 400, never a silent fallback to a different backend, and the
        # key value itself never appears in this message.
        raise HTTPException(status_code=400, detail=str(exc)) from exc

    context = get_context(user_input)
    full_prompt = f"{context}\nUser request: {user_input}\n\nWrite your response:"
    chat_messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": full_prompt},
    ]

    if stream:
        return StreamingResponse(
            _stream_and_log(client, backend_model, chat_messages, user_input, requested_model),
            media_type="text/event-stream",
        )

    try:
        completion = client.chat.completions.create(
            model=backend_model,
            messages=chat_messages,
            temperature=0.75,
            max_tokens=MAX_TOKENS,
        )
    except Exception as exc:
        raise HTTPException(status_code=502, detail=f"Upstream model call failed: {exc}") from exc

    assistant_text = completion.choices[0].message.content or ""
    history.save("User", user_input)
    history.save("Assistant", assistant_text.strip())

    payload = completion.model_dump()
    payload["model"] = requested_model  # report the registry name, not the raw backend id
    return JSONResponse(payload)


async def _stream_and_log(
    client, backend_model: str, chat_messages: list[dict], user_input: str, requested_model: str
) -> AsyncGenerator[str, None]:
    """Proxy the backend's own OpenAI-schema stream chunks as SSE.

    Both local (Ollama) and cloud (NVIDIA) backends are called through the
    same OpenAI-compatible client, so their stream chunks already match the
    schema Open WebUI (Stage 4) expects — re-serializing them directly (rather
    than hand-building chunk JSON) is what guarantees byte-level fidelity.
    """
    full_response = ""
    try:
        completion_stream = client.chat.completions.create(
            model=backend_model,
            messages=chat_messages,
            temperature=0.75,
            max_tokens=MAX_TOKENS,
            stream=True,
        )
        for chunk in completion_stream:
            chunk_dict = chunk.model_dump()
            chunk_dict["model"] = requested_model
            choices = chunk_dict.get("choices") or [{}]
            delta = choices[0].get("delta") or {}
            full_response += delta.get("content") or ""
            yield f"data: {json.dumps(chunk_dict)}\n\n"
    except Exception as exc:
        error_chunk = {"error": {"message": f"Upstream model call failed: {exc}"}}
        yield f"data: {json.dumps(error_chunk)}\n\n"
    finally:
        yield "data: [DONE]\n\n"
        if full_response.strip():
            history.save("User", user_input)
            history.save("Assistant", full_response.strip())
```

</details>

**`app/requirements.txt`** — pinned Python dependencies, installed only inside the container image.

<details>

```text
# Japanese Light Novel Writer Agent — Lightweight design (system-design-lightweight.md)
# Pruned and pinned per Stage 0. Resolved conflict-free via `pip install --dry-run`
# in a scratch venv on 2026-07-13 (record real versions here per the stage's
# "pin to whatever the resolver produces" instruction).
#
# Deliberately NOT included (full design only, see system-design.md):
#   crewai, langgraph, langgraph-checkpoint-sqlite, langchain-community, langfuse, mcp
# Deliberately NOT included (dead weight per audit P2-6, unused by any code this
# design keeps): gradio, playwright, sentence-transformers, aiohttp, pypdf,
# pdfplumber, reportlab, python-pptx, openpyxl, python-docx, markitdown[all]

# HTTP service (Stage 1)
fastapi==0.139.0
uvicorn[standard]==0.51.0
pydantic==2.13.4
python-dotenv==1.2.2

# LLM clients — single OpenAI-compatible client type for both local Ollama and
# NVIDIA cloud models (Stage 1); langchain-ollama kept available for the native
# local/embeddings path (Stage 3).
openai==2.45.0
langchain-ollama==1.1.0

# Embedded vector memory (Stage 3)
chromadb==1.5.9

# Ollama embeddings HTTP calls (Stage 3)
httpx==0.28.1
```

</details>

**`app/Dockerfile`** — builds the agent service's container image.

<details>

```dockerfile
# Japanese Light Novel Writer Agent — lightweight design, Stage 2
# (system-design-lightweight.md). Base image, requirements install, and run
# command are exactly as specified in Stage 2 — no substitutions.
#
# Build context is ./app (see docker-compose.yml: `build: ./app`), so this
# file and everything it COPYs are the contents of the host's app/ directory
# (server.py, models.py, history_manager.py, __init__.py, requirements.txt).
#
# Those files are copied into /app/app/ inside the image (reconstructing the
# "app/" directory the design's compose volumes and run command both assume),
# with WORKDIR set one level up at /app. That's what makes both of these
# hold at once, unchanged from the design doc:
#   - `uvicorn app.server:app` resolves "app" as the package at /app/app/
#     (the same project-root-relative layout Stage 1 already verified works,
#     run from the host: https://app.server:app from a directory containing app/).
#   - the compose bind mounts (/app/chapters, /app/chat_history, /app/logs)
#     land as siblings of /app/app/, and history_manager.py's relative "."
#     base_dir (CWD = /app at runtime) resolves straight into them.

FROM python:3.11-slim

# curl is required by docker-compose.yml's healthcheck (`curl -f
# http://localhost:8000/health`). Not part of the app's own runtime
# dependencies — installed here because python:3.11-slim doesn't include it.
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt ./app/requirements.txt
RUN pip install --no-cache-dir -r ./app/requirements.txt

COPY . ./app/

CMD ["uvicorn", "app.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

</details>

At this point `app/` should have all seven files: `__init__.py`, `logging_config.py`, `history_manager.py`, `models.py`, `memory.py`, `server.py`, `requirements.txt`, and `Dockerfile`.

### Step 5 — Create `docker-compose.yml`

At the project root (a sibling of `app/`, not inside it):

<details>

```yaml
# Japanese Light Novel Writer Agent — lightweight design, Stages 2-4
# (system-design-lightweight.md). Reproduced exactly from the design doc's
# compose snippets — no substitutions.
#
# Both services are now defined: `agent` (Stages 1-3, the OpenAI-compatible
# backend) and `open-webui` (Stage 4, the browser chat UI that talks to it).

services:
  agent:
    build: ./app
    ports:
      - "8000:8000"
    env_file: .env                     # NVIDIA_API_KEY comes in from here, never inline
    environment:
      OLLAMA_BASE_URL: http://host.docker.internal:11434
      # If you prefer explicit interpolation over env_file for the secret:
      # NVIDIA_API_KEY: ${NVIDIA_API_KEY}   # value stays in .env; compose interpolates, no literal
    extra_hosts:
      - "host.docker.internal:host-gateway"   # explicit; harmless on Docker Desktop
    volumes:
      - ./chapters:/app/chapters
      - ./chat_history:/app/chat_history
      - ./logs:/app/logs
      - ./chroma_db:/app/chroma_db      # Stage 3: embedded Chroma persistent store
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 5

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports: [ "8080:8080" ]
    environment:
      OPENAI_API_BASE_URL: http://agent:8000/v1
      OPENAI_API_KEY: ollama            # agent ignores it; UI requires a value
    depends_on:
      agent: { condition: service_healthy }
    volumes: [ "./open-webui-data:/app/backend/data" ]   # bind-mount per blanket rule
```

</details>

### Step 6 — Create `env.example` and `.gitignore`

**`env.example`** (project root) — the safe-to-commit template:

<details>

```bash
# Japanese Light Novel Writer Agent — environment template
#
# Copy this file to `.env` (which is gitignored — never commit the real one)
# and fill in the value below. See README.md / TUTORIAL.md for details.
#
#   cp env.example .env

# Required only for the two NVIDIA cloud models (glm-5.2, gemma-4-31b-it).
# The local model (llama3.1-local, via Ollama) works with this left blank.
# Get a free key at https://build.nvidia.com/models?modal=signin
NVIDIA_API_KEY=Insert-API-KEY-HERE
```

</details>

**`.gitignore`** (project root) — keeps secrets and generated/runtime data out of version control:

<details>

```text
# Secrets — never commit real values
.env

# Python
__pycache__/
*.pyc
.venv/
venv/
*.egg-info/

# Generated / runtime data (bind-mount targets — real content stays local, not versioned)
chat_history/*.md
chroma_db/*
logs/*.log
chapters/*
open-webui-data/*

# install.sh / uninstall.sh — machine-specific record of what was installed here
install_manifest.txt
install_manifest.txt.uninstalled-*

# OS
.DS_Store

# Editors
.vscode/
.idea/
```

</details>

At this point every file the stack needs to build and run exists on disk.

### Step 7 — Create your `.env` file

```bash
cp env.example .env
```
Open `.env` in any text editor and set your NVIDIA key (skip this if you're only using the local model):
```
NVIDIA_API_KEY=nvapi-your-real-key-here
```
See [Configuring your API key](#configuring-your-api-key-env) below for more detail on this file specifically.

*Scripted alternative:* `install.sh` creates `.env` from the template for you and prompts you for the key value interactively (or lets you skip it and edit the file yourself later).

### Step 8 — Build and start the stack

```bash
docker compose up --build -d
```
This builds the `agent` image from `app/Dockerfile`, pulls the official `open-webui` image, and starts both. Docker Compose's `depends_on: condition: service_healthy` means `open-webui` won't start until `agent` passes its health check — so you never hit the UI before the backend it needs is actually ready.

*Scripted alternative:* `install.sh` can run this final command for you at the end of the install, after confirming with you.

### Step 9 — Verify everything is working

```bash
docker compose ps                          # both services should show "healthy"/"running"
curl http://localhost:8000/health           # {"status":"ok"}
curl http://localhost:8000/v1/models        # lists all three registered models
```
Then open **http://localhost:8080** in a browser. On first launch, Open WebUI will ask you to create a local admin account (its own onboarding — nothing this project configures) — do that, then confirm the model dropdown
lists `llama3.1-local`, `glm-5.2`, and `gemma-4-31b-it`, and that sending a message streams a reply back.

You're done — every file created by hand, stack running, same result as `install.sh` end to end.

### Step 10 (optional) — Add the `install.sh` / `uninstall.sh` convenience scripts

You've already done by hand everything these two scripts automate, so this step is optional — add them only if you (or someone else) will be reinstalling or uninstalling this project again later and want the scripted shortcut.

**`install.sh`** (project root):

<details>

```bash
#!/usr/bin/env bash
#
# install.sh — automated installer for the Japanese Light Novel Writer Agent.
#
# What this does, in order:
#   1. Detects your OS (macOS or Linux) and bails cleanly on anything else.
#   2. Checks for Docker, the Compose plugin, and Ollama.
#   3. Lists exactly what's missing and asks permission BEFORE installing anything.
#   4. Installs only what's missing, using your OS's normal package manager.
#   5. Pulls the two Ollama models this project needs.
#   6. Sets up .env from env.example and prompts for your NVIDIA API key.
#   7. Optionally builds and starts the Docker stack.
#
# Every dependency this script installs (not ones you already had) is recorded
# in install_manifest.txt, one line per item, so uninstall.sh can reverse
# exactly and only what this script added — nothing you had beforehand.
#
# Safe to re-run: already-satisfied checks are skipped, nothing is installed
# twice, and you're asked before every install action.

set -euo pipefail

# ---------------------------------------------------------------------------
# Setup
# ---------------------------------------------------------------------------

PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$PROJECT_DIR"

MANIFEST_FILE="$PROJECT_DIR/install_manifest.txt"
ENV_FILE="$PROJECT_DIR/.env"
ENV_EXAMPLE="$PROJECT_DIR/env.example"

BOLD="$(tput bold 2>/dev/null || true)"
RESET="$(tput sgr0 2>/dev/null || true)"
GREEN="$(tput setaf 2 2>/dev/null || true)"
YELLOW="$(tput setaf 3 2>/dev/null || true)"
RED="$(tput setaf 1 2>/dev/null || true)"

say()   { printf '%s\n' "$*"; }
info()  { printf '%s%s%s\n' "$GREEN" "$*" "$RESET"; }
warn()  { printf '%s%s%s\n' "$YELLOW" "$*" "$RESET"; }
error() { printf '%s%s%s\n' "$RED" "$*" "$RESET" >&2; }
heading() { printf '\n%s== %s ==%s\n' "$BOLD" "$*" "$RESET"; }

confirm() {
    # confirm "question" -> 0 (yes) or 1 (no). Defaults to No.
    local prompt="$1"
    local answer
    read -r -p "$prompt [y/N] " answer || answer=""
    case "$answer" in
        [yY]|[yY][eE][sS]) return 0 ;;
        *) return 1 ;;
    esac
}

manifest_add() {
    # manifest_add "<component>" "<method>" "<note>"
    printf '%s|%s|%s\n' "$1" "$2" "$3" >> "$MANIFEST_FILE"
}

command_exists() { command -v "$1" >/dev/null 2>&1; }

# ---------------------------------------------------------------------------
# 1. OS detection
# ---------------------------------------------------------------------------

heading "Detecting operating system"

OS_NAME="$(uname -s)"
case "$OS_NAME" in
    Darwin) PLATFORM="macos" ;;
    Linux)  PLATFORM="linux" ;;
    *)
        error "Unsupported OS: $OS_NAME. This installer supports macOS and Linux only."
        error "See TUTORIAL.md for manual install steps you can adapt."
        exit 1
        ;;
esac
info "Detected: $PLATFORM ($(uname -m))"

if [ "$PLATFORM" = "linux" ]; then
    if command_exists apt-get; then
        LINUX_PKG_MGR="apt"
    elif command_exists dnf; then
        LINUX_PKG_MGR="dnf"
    else
        error "No supported package manager found (looked for apt-get, dnf)."
        error "See TUTORIAL.md to install Docker and Ollama manually for your distro."
        exit 1
    fi
    info "Package manager: $LINUX_PKG_MGR"
fi

# Start the manifest file (append-only; keep prior runs' entries if any).
if [ ! -f "$MANIFEST_FILE" ]; then
    {
        echo "# JP-LN-Writer install manifest"
        echo "# Generated: $(date -u +%Y-%m-%dT%H:%M:%SZ) on $PLATFORM ($(uname -m))"
        echo "# Format: <component>|<install method>|<note>"
        echo "# uninstall.sh reads this file to remove ONLY what install.sh added."
    } > "$MANIFEST_FILE"
fi

# ---------------------------------------------------------------------------
# 2. Dependency check
# ---------------------------------------------------------------------------

heading "Checking dependencies"

MISSING=()

if command_exists docker && docker compose version >/dev/null 2>&1; then
    info "Docker + Compose plugin: found ($(docker --version))"
else
    warn "Docker (with the Compose plugin): NOT found"
    MISSING+=("docker")
fi

if command_exists ollama; then
    info "Ollama: found ($(ollama --version 2>/dev/null | head -n1))"
else
    warn "Ollama: NOT found"
    MISSING+=("ollama")
fi

if command_exists curl; then
    info "curl: found"
else
    warn "curl: NOT found"
    MISSING+=("curl")
fi

if [ ${#MISSING[@]} -eq 0 ]; then
    info "All dependencies already satisfied."
else
    heading "Missing dependencies"
    say "The following are required and not currently installed:"
    for dep in "${MISSING[@]}"; do
        say "  - $dep"
    done
    say ""
    if ! confirm "Install the above now?"; then
        error "Aborted. Nothing was installed. See TUTORIAL.md for manual install steps."
        exit 1
    fi
fi

# ---------------------------------------------------------------------------
# 3. Install missing dependencies
# ---------------------------------------------------------------------------

install_docker_macos() {
    if ! command_exists brew; then
        warn "Homebrew is required to install Docker automatically on macOS but isn't installed."
        if confirm "Install Homebrew now (runs the official install script from brew.sh)?"; then
            /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
            manifest_add "homebrew" "official_script" "Installed via brew.sh's official install script"
        else
            error "Cannot continue without Homebrew. Install Docker Desktop manually from docker.com, then re-run this script."
            exit 1
        fi
    fi
    brew install --cask docker
    manifest_add "docker" "brew_cask" "Installed via 'brew install --cask docker'"
    warn "Docker Desktop is installed but needs to be launched once manually to finish setup."
    open -a Docker || true
    say "Waiting for Docker to start (up to 60s)..."
    for _ in $(seq 1 30); do
        if docker info >/dev/null 2>&1; then break; fi
        sleep 2
    done
    if ! docker info >/dev/null 2>&1; then
        warn "Docker doesn't seem to be running yet. Open Docker Desktop manually, wait for it to say \"running\", then re-run this script."
    fi
}

install_docker_linux() {
    say "This will run Docker's official convenience script:"
    say "  curl -fsSL https://get.docker.com | sh"
    if ! confirm "Proceed?"; then
        error "Cannot continue without Docker. See TUTORIAL.md for manual steps."
        exit 1
    fi
    curl -fsSL https://get.docker.com | sh
    manifest_add "docker" "docker_convenience_script" "Installed via get.docker.com script ($LINUX_PKG_MGR-based repo)"
    if confirm "Add $USER to the 'docker' group (lets you run docker without sudo)?"; then
        sudo usermod -aG docker "$USER"
        manifest_add "docker-group-membership" "usermod" "Added $USER to the docker group — remove manually with 'sudo gpasswd -d $USER docker' if desired"
        warn "You must log out and back in for group membership to take effect."
    fi
    if command_exists systemctl; then
        sudo systemctl enable --now docker || true
    fi
}

install_ollama_macos() {
    if ! command_exists brew; then
        error "Homebrew is required to install Ollama automatically on macOS. Install it first (re-run this script) or install Ollama manually from ollama.com."
        exit 1
    fi
    brew install ollama
    manifest_add "ollama" "brew_formula" "Installed via 'brew install ollama'"
    brew services start ollama || (ollama serve >/dev/null 2>&1 &)
}

install_ollama_linux() {
    say "This will run Ollama's official install script:"
    say "  curl -fsSL https://ollama.com/install.sh | sh"
    if ! confirm "Proceed?"; then
        error "Cannot continue without Ollama. See TUTORIAL.md for manual steps."
        exit 1
    fi
    curl -fsSL https://ollama.com/install.sh | sh
    manifest_add "ollama" "ollama_official_script" "Installed via ollama.com/install.sh (installs a systemd service on most distros)"
}

install_curl() {
    case "$PLATFORM" in
        macos)
            brew install curl
            manifest_add "curl" "brew_formula" "Installed via 'brew install curl'"
            ;;
        linux)
            if [ "$LINUX_PKG_MGR" = "apt" ]; then
                sudo apt-get update && sudo apt-get install -y curl
                manifest_add "curl" "apt" "Installed via 'apt-get install curl'"
            else
                sudo dnf install -y curl
                manifest_add "curl" "dnf" "Installed via 'dnf install curl'"
            fi
            ;;
    esac
}

for dep in "${MISSING[@]:-}"; do
    [ -z "$dep" ] && continue
    case "$dep" in
        docker) [ "$PLATFORM" = "macos" ] && install_docker_macos || install_docker_linux ;;
        ollama) [ "$PLATFORM" = "macos" ] && install_ollama_macos || install_ollama_linux ;;
        curl)   install_curl ;;
    esac
done

# ---------------------------------------------------------------------------
# 4. Pull required Ollama models
# ---------------------------------------------------------------------------

heading "Ollama models"

pull_model_if_missing() {
    local model="$1"
    if ollama list 2>/dev/null | awk '{print $1}' | grep -xqE "${model}(:latest)?"; then
        info "$model: already pulled"
        return
    fi
    if confirm "Pull Ollama model '$model' now (this downloads several GB)?"; then
        ollama pull "$model"
        say "Note: pulled models are NOT recorded for removal by uninstall.sh —"
        say "your model downloads are kept regardless, so you never lose them"
        say "or have to re-download on a reinstall."
    else
        warn "Skipped pulling $model. The agent will still start, but requests using it will fail until you pull it manually: ollama pull $model"
    fi
}

pull_model_if_missing "llama3.1"
pull_model_if_missing "nomic-embed-text"

# ---------------------------------------------------------------------------
# 5. .env setup
# ---------------------------------------------------------------------------

heading "Environment configuration (.env)"

if [ -f "$ENV_FILE" ]; then
    info ".env already exists — leaving it as-is."
else
    cp "$ENV_EXAMPLE" "$ENV_FILE"
    info "Created .env from env.example."
    say ""
    say "The two NVIDIA cloud models (glm-5.2, gemma-4-31b-it) need an API key."
    say "The local model (llama3.1-local) works without one."
    say "Get a free key at: https://build.nvidia.com/models?modal=signin"
    say ""
    read -r -p "Paste your NVIDIA_API_KEY now, or press Enter to skip and edit .env later: " nvidia_key
    if [ -n "$nvidia_key" ]; then
        if [ "$PLATFORM" = "macos" ]; then
            sed -i '' "s|^NVIDIA_API_KEY=.*|NVIDIA_API_KEY=${nvidia_key}|" "$ENV_FILE"
        else
            sed -i "s|^NVIDIA_API_KEY=.*|NVIDIA_API_KEY=${nvidia_key}|" "$ENV_FILE"
        fi
        info "Saved to .env."
    else
        say "Skipped. Edit .env manually later: NVIDIA_API_KEY=your-real-key-here"
    fi
fi

# ---------------------------------------------------------------------------
# 6. Build and start
# ---------------------------------------------------------------------------

heading "Start the stack"

if confirm "Build and start the Docker stack now (docker compose up --build -d)?"; then
    docker compose up --build -d
    say ""
    info "Started. Check status with: docker compose ps"
    info "Open the UI at: http://localhost:8080"
    info "Or use the API directly at: http://localhost:8000"
else
    say "Skipped. Start it yourself whenever you're ready:"
    say "  docker compose up --build -d"
fi

# ---------------------------------------------------------------------------
# Summary
# ---------------------------------------------------------------------------

heading "Done"
say "Install manifest written to: $MANIFEST_FILE"
say "Run ./uninstall.sh at any time to cleanly remove only what this script installed."
say "See README.md and TUTORIAL.md for usage and troubleshooting."
```

</details>

**`uninstall.sh`** (project root) — cleanly reverses `install.sh`:

<details>

```bash
#!/usr/bin/env bash
#
# uninstall.sh — cleanly reverses install.sh.
#
# Reads install_manifest.txt (written by install.sh) and removes ONLY the
# dependencies that script actually installed on this machine — never
# anything that was already present before you ran it.
#
# This script NEVER touches: chapters/, chat_history/, chroma_db/, logs/,
# open-webui-data/, .env, or any Ollama models you've pulled. Re-running
# install.sh after this always picks up exactly where you left off — your
# chapters, chat history, and vector memory are never at risk from this
# script, under any option you choose below.

set -euo pipefail

PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$PROJECT_DIR"

MANIFEST_FILE="$PROJECT_DIR/install_manifest.txt"

BOLD="$(tput bold 2>/dev/null || true)"
RESET="$(tput sgr0 2>/dev/null || true)"
GREEN="$(tput setaf 2 2>/dev/null || true)"
YELLOW="$(tput setaf 3 2>/dev/null || true)"
RED="$(tput setaf 1 2>/dev/null || true)"

say()   { printf '%s\n' "$*"; }
info()  { printf '%s%s%s\n' "$GREEN" "$*" "$RESET"; }
warn()  { printf '%s%s%s\n' "$YELLOW" "$*" "$RESET"; }
error() { printf '%s%s%s\n' "$RED" "$*" "$RESET" >&2; }
heading() { printf '\n%s== %s ==%s\n' "$BOLD" "$*" "$RESET"; }

confirm() {
    local prompt="$1"
    local answer
    read -r -p "$prompt [y/N] " answer || answer=""
    case "$answer" in
        [yY]|[yY][eE][sS]) return 0 ;;
        *) return 1 ;;
    esac
}

command_exists() { command -v "$1" >/dev/null 2>&1; }

OS_NAME="$(uname -s)"
case "$OS_NAME" in
    Darwin) PLATFORM="macos" ;;
    Linux)  PLATFORM="linux" ;;
    *) PLATFORM="unknown" ;;
esac

if [ "$PLATFORM" = "linux" ]; then
    if command_exists apt-get; then LINUX_PKG_MGR="apt"
    elif command_exists dnf; then LINUX_PKG_MGR="dnf"
    else LINUX_PKG_MGR="unknown"
    fi
fi

# ---------------------------------------------------------------------------
# 1. Docker containers/images for this project only
# ---------------------------------------------------------------------------

heading "Project Docker resources"

if command_exists docker && docker compose version >/dev/null 2>&1; then
    if confirm "Stop the stack and remove this project's Docker images (agent + open-webui)? (docker compose down --rmi all)"; then
        docker compose down --rmi all
        info "Stopped and removed this project's containers and images."
        say "Note: no named Docker volumes are used by this project (everything is"
        say "bind-mounted to folders in this directory), so this cannot touch your"
        say "chapters, chat history, memory, logs, or Open WebUI data."
    else
        say "Skipped."
    fi
else
    warn "Docker not found or not running — skipping container/image cleanup."
fi

# ---------------------------------------------------------------------------
# 2. Read the install manifest
# ---------------------------------------------------------------------------

heading "Reading install manifest"

if [ ! -f "$MANIFEST_FILE" ]; then
    warn "No install_manifest.txt found — install.sh either wasn't used, or"
    warn "everything on this system predates this project. Nothing to remove."
    say "If you installed Docker/Ollama yourself, remove them manually via"
    say "your OS's normal package manager — see TUTORIAL.md."
    exit 0
fi

# Parse non-comment, non-blank lines: component|method|note
ENTRIES=()
while IFS= read -r line; do
    case "$line" in
        ''|'#'*) continue ;;
        *) ENTRIES+=("$line") ;;
    esac
done < "$MANIFEST_FILE"

if [ ${#ENTRIES[@]} -eq 0 ]; then
    info "Manifest exists but lists nothing to remove."
    exit 0
fi

say "install.sh recorded the following items on this machine:"
for entry in "${ENTRIES[@]}"; do
    component="${entry%%|*}"
    rest="${entry#*|}"
    method="${rest%%|*}"
    note="${rest#*|}"
    say "  - $component  (via $method)  — $note"
done
say ""
say "Your project data (chapters/, chat_history/, chroma_db/, logs/,"
say "open-webui-data/, .env, and any Ollama models) will NOT be touched,"
say "no matter what you choose below."
say ""

if ! confirm "Remove the items listed above?"; then
    say "Aborted. Nothing further was removed."
    exit 0
fi

# ---------------------------------------------------------------------------
# 3. Reverse each manifest entry
# ---------------------------------------------------------------------------

heading "Removing installed dependencies"

remove_docker_convenience_script() {
    warn "Docker was installed via the get.docker.com convenience script."
    if [ "$PLATFORM" = "linux" ] && [ "${LINUX_PKG_MGR:-unknown}" = "apt" ]; then
        if confirm "Purge Docker packages via apt now?"; then
            sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin || true
            sudo apt-get autoremove -y || true
            info "Docker packages purged."
        fi
    elif [ "$PLATFORM" = "linux" ] && [ "${LINUX_PKG_MGR:-unknown}" = "dnf" ]; then
        if confirm "Remove Docker packages via dnf now?"; then
            sudo dnf remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin || true
            info "Docker packages removed."
        fi
    else
        warn "Don't know how to auto-remove Docker on this system. Remove it manually via your package manager."
    fi
}

remove_ollama_official_script() {
    warn "Ollama was installed via ollama.com's official install script."
    if confirm "Stop and remove the Ollama service and binary now?"; then
        sudo systemctl stop ollama 2>/dev/null || true
        sudo systemctl disable ollama 2>/dev/null || true
        sudo rm -f /etc/systemd/system/ollama.service
        sudo rm -f "$(command -v ollama 2>/dev/null || echo /usr/local/bin/ollama)"
        sudo systemctl daemon-reload 2>/dev/null || true
        info "Ollama service and binary removed."
        warn "Ollama's model storage (/usr/share/ollama or ~/.ollama) was left in"
        warn "place deliberately — your pulled models are never removed by this script."
    fi
}

for entry in "${ENTRIES[@]}"; do
    component="${entry%%|*}"
    rest="${entry#*|}"
    method="${rest%%|*}"

    case "$method" in
        brew_cask)
            if confirm "Uninstall '$component' via Homebrew cask?"; then
                brew uninstall --cask "$component" || warn "brew uninstall --cask $component failed — remove manually if needed."
            fi
            ;;
        brew_formula)
            if confirm "Uninstall '$component' via Homebrew?"; then
                brew uninstall "$component" || warn "brew uninstall $component failed — remove manually if needed."
            fi
            ;;
        apt)
            if confirm "Remove '$component' via apt?"; then
                sudo apt-get remove -y "$component" || warn "apt-get remove $component failed — remove manually if needed."
            fi
            ;;
        dnf)
            if confirm "Remove '$component' via dnf?"; then
                sudo dnf remove -y "$component" || warn "dnf remove $component failed — remove manually if needed."
            fi
            ;;
        docker_convenience_script)
            remove_docker_convenience_script
            ;;
        ollama_official_script)
            remove_ollama_official_script
            ;;
        usermod)
            warn "'$component' was a group-membership change, not a package —"
            warn "not reversed automatically. To revert: sudo gpasswd -d \"\$USER\" docker"
            ;;
        official_script)
            if [ "$component" = "homebrew" ]; then
                warn "Homebrew itself is shared infrastructure — it may be used by other"
                warn "projects on this machine, so it is NOT removed automatically."
                warn "To remove it yourself: https://docs.brew.sh/FAQ#how-do-i-uninstall-homebrew"
            else
                warn "Don't know how to auto-remove '$component' (method: official_script). Skipping — remove manually if needed."
            fi
            ;;
        *)
            warn "Unknown install method '$method' for '$component' — skipping. Remove manually if needed."
            ;;
    esac
done

# ---------------------------------------------------------------------------
# 4. Archive the manifest (don't silently delete the audit trail)
# ---------------------------------------------------------------------------

ARCHIVE_NAME="install_manifest.txt.uninstalled-$(date -u +%Y%m%dT%H%M%SZ)"
mv "$MANIFEST_FILE" "$PROJECT_DIR/$ARCHIVE_NAME"
info "Manifest archived to $ARCHIVE_NAME (kept for your records)."

# ---------------------------------------------------------------------------
# Summary
# ---------------------------------------------------------------------------

heading "Done"
say "Removed only what install.sh added to this machine."
say ""
say "Left untouched, on purpose:"
say "  - chapters/          (your story files)"
say "  - chat_history/      (your saved conversations)"
say "  - chroma_db/         (your indexed memory)"
say "  - logs/              (your application logs)"
say "  - open-webui-data/   (your UI account and conversations)"
say "  - .env               (your configuration)"
say "  - Any Ollama models you pulled"
say ""
say "To pick up exactly where you left off, just run ./install.sh again —"
say "none of the above ever needs to be rebuilt or re-saved."
```

</details>

Make both executable:
```bash
chmod +x install.sh uninstall.sh
```

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
