# Elders of the Internet

*A local‑first, modular multi‑agent AI system with a Fallout‑style training terminal.*

> **Modes:** `/` Council • `/train` Training/Annotation • `/admin` Ingest/Datasets (later)

---

## 0) Quickstart

```bash
# 1) Generate repo skeleton (safe & idempotent)
bash scripts/scaffold.sh

# 2) Open in VS Code\ ncode .
```

> The scaffold only creates structure and placeholder files. No heavy code.

---

## 1) Vision (recap)

Create a local‑first, modular multi‑agent system — a “Council of Elders” — that:

* Archives & serves historical + current knowledge with traceable citations.
* Debates with personality (Archivist, Herald, Philosopher, Jester).
* Streams witty banter while thinking; desktop‑first, terminal UI.
* Runs in containers; extensible via personas and content packs.

---

## 2) /train — Desktop UX (spec)

* **Layout:** Four horizontal elder rows + one bottom user terminal.
* **Panels:** Title, persona badge, (optional) score, Approved/Rejected state, transcript, citations, actions, "Mark as Best".
* **User box:** Prompt, tags, difficulty, temperature, auto‑cite, Train, Save Draft, Export JSONL.
* **Hotkeys:** `Ctrl+Enter` (Train) • `1–4` (Choose Best) • `g` (advanced) • `[` `]` (temp).
* **Style:** Phosphor‑green mono, subtle CRT bloom/scanlines (toggle), flicker on state change.

---

## 3) One‑Click Setup Launcher (spec)

**Entry:** `elders setup | start | doctor | reset | wipe | update | logs | gpu-test`

**Preflight:** OS/arch → permissions → Docker/Compose → GPU path (opt‑in) → ports → network → disk/volumes → env → images → index init → seed corpus → personas → TTS (optional) → observability → security bind (localhost) → telemetry off by default.

**Smoke tests:** Coordinator `/healthz`; search roundtrip (OpenSearch + Vector DB); elder handshake + token stream; WS stream ≥ 10 tokens; TTS ping (if enabled); export JSONL write/read.

**Failure UX:** green checks, red X with remediation; one‑click fix where possible; log bundle command.

---

## 4) Services (names only)

* `frontend` (web UI)
* `coordinator` (API + WS hub)
* `elder-archivist`, `elder-herald`, `elder-philosopher`, `elder-jester`
* `opensearch`, `qdrant` (or `weaviate`, pick one for MVP)
* `dataset-registry` (simple file API)
* `tts` (optional)
* `scraper` (later)

---

## 5) Data Schemas (authoritative)

**ElderPanel**

```json
{
  "id": "archivist|herald|philosopher|jester",
  "title": "Archivist",
  "persona": "Historian",
  "voice": "optional",
  "transcript": "",
  "citations": [{"id":"w123","label":"Wikipedia: …","href":"…"}],
  "score": 0.92,
  "approved": true
}
```

**TrainSample (JSONL line)**

```json
{
  "sessionId": "uuid",
  "userText": "…",
  "elders": {"archivist": {"…ElderPanel…"}, "herald": {"…"}},
  "chosenElder": "herald",
  "tags": ["ethics","ai"],
  "difficulty": "medium",
  "createdAt": 1734048000000
}
```

---

## 6) API/WS Contracts (MVP)

**HTTP**

* `POST /train/start` → `{ sessionId }`
* `POST /train/export` → `{ ok, fileId }`
* `POST /train/save-draft` → `{ ok, draftId }`
* `GET /train/session/:id` → `TrainSample`

**WS events**

* Client→Server: `train.start`, `train.pause`, `train.resume`, `train.rerun`, `train.annotate`, `train.chooseBest`
* Server→Client: `token`, `status`, `citations`, `score`, `done`, `error`

---

## 7) Directory Skeleton

```
/elders-internet
  /frontend
    /routes            # "/", "/train", "/admin" (stub)
    /state             # store slices
    /ui                # ElderPanel, UserBox, TopBar, CitationChip
    /ws                # client, events
    /public            # favicon, fonts
    README.md
  /coordinator
    /http              # routes: train, export (stubs)
    /ws                # broker, topics (stubs)
    /orchestration     # fanout/gather/scoring (stubs)
    /models            # schemas
    /adapters          # opensearch, vectordb, tts (stubs)
    README.md
  /elders
    /archivist
      persona.yaml
      README.md
    /herald
      persona.yaml
      README.md
    /philosopher
      persona.yaml
      README.md
    /jester
      persona.yaml
      README.md
    /shared            # prompt templates & tool adapters
      README.md
  /search
    /opensearch        # index templates, init scripts (stubs)
    /vectordb          # collections init (stubs)
    README.md
  /dataset-registry
    /exports           # JSONL dumps
    /fixtures          # seed corpus
    README.md
  /tts                 # optional service docs
    README.md
  /launcher
    /checks            # docker, gpu, ports, env, indexes, tts
    /scripts           # platform-specific helpers
    /fixtures          # seed data & test prompts
    config.yaml        # ordered checks + remediation policy
    profiles.yaml      # cpu-only | nvidia-gpu | apple-silicon
    README.md
  /infra
    docker-compose.yml # names only, commented stubs
    .env.example
  scripts/
    scaffold.sh        # creates all of the above
  .gitignore
  README.md            # this file
```

---

## 8) Open Decisions

* **Vector DB:** Qdrant vs Weaviate (default Qdrant for local‑first).
* **TTS:** Piper (light) vs Coqui (richer). Default off with opt‑in.
* **GPU profile:** opt‑in during setup; CPU fallback.

---

## 9) Acceptance Criteria (MVP /train)

* 4 elder panels + 1 user terminal render.
* Session state persists chosen/approved per elder.
* Export produces valid JSONL line.
* Launcher `setup → start` lands you on `/train`.

---

## 10) License & Credits

TBD. Default to MIT unless constraints require otherwise.

---

# scripts/scaffold.sh

> **What it does**: creates the repo skeleton above with placeholder files, sensible defaults, and guardrails. Safe to re‑run. No destructive actions.

**Usage**

```bash
bash scripts/scaffold.sh           # create structure
bash scripts/scaffold.sh --force   # also overwrite placeholder files
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Elders of the Internet – Repo Scaffolder
# Safe to re-run; use --force to overwrite placeholder files.

FORCE=0
if [[ ${1:-} == "--force" ]]; then FORCE=1; fi

ROOT_DIR=$(pwd)
PROJECT_NAME="elders-internet"

say() { printf "\033[1;32m▸\033[0m %s\n" "$*"; }
warn() { printf "\033[1;33m!\033[0m %s\n" "$*"; }
make_dir() { mkdir -p "$1"; }
write_file() {
  local path="$1"; shift
  if [[ -f "$path" && $FORCE -eq 0 ]]; then
    warn "exists: $path (use --force to overwrite)"; retur
```
