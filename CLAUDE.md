# CLAUDE.md

Operating manual for Claude Code working in this repository. Keep this file 100% truthful: if reality diverges, edit this file first, then the code.

---

## 1. Project overview

**VoiceRAG** (aisearch-openai-rag-audio) is a sample application pattern for doing Retrieval-Augmented Generation over a voice interface. The browser captures microphone audio and streams it over a WebSocket to a small Python middle tier, which proxies the session to the **Azure OpenAI GPT-4o Realtime API**. The middle tier injects a server-enforced system prompt and two function tools (`search` and `report_grounding`) that query **Azure AI Search**; tool calls are executed server-side and hidden from the browser, so the client only ever sees audio, transcripts, and citation payloads. Answers come back as streamed audio plus grounding citations rendered as clickable file chips.

This is a fork of [Azure-Samples/aisearch-openai-rag-audio](https://github.com/Azure-Samples/aisearch-openai-rag-audio), kept as a portfolio repo. `AGENTS.md` is the older coding-agent onboarding doc from upstream; this file is the canonical operating manual. Companion docs: `PRODUCT.md` (what and why), `ARCHITECTURE.md` (components and data flow), `CONTRIBUTING.md` (dev setup, points to `.github/CONTRIBUTING.md` for the upstream CLA/process).

Key capabilities:

- Voice in / voice out chat over a knowledge base (sample data: Contoso HR/benefits PDFs in `data/`).
- Server-side RAG function calling: hybrid (text + vector) search with optional semantic ranker against Azure AI Search.
- Citation reporting: the model calls `report_grounding`, the backend fetches the cited chunks and pushes them to the client as a custom `extension.middle_tier_tool_response` message.
- One-command Azure deployment (`azd up`) to Azure Container Apps, including index creation via AI Search integrated vectorization.

## 2. Tech stack

### Backend (`app/backend/`)
- Python (>=3.11 per README; Dockerfile and devcontainer use 3.12).
- **aiohttp 3.9.3** web server (WebSocket handling + static file serving). No Flask/Quart/FastAPI.
- `azure-identity 1.18.0` (AzureDeveloperCliCredential / DefaultAzureCredential / key auth fallback).
- `azure-search-documents 11.6.0b4` (beta pin, needed for integrated-vectorization index classes used by `setup_intvect.py`).
- `azure-storage-blob 12.23.1` (document upload in `setup_intvect.py`).
- `python-dotenv 1.0.1`, `gunicorn` (production server), `rich` (logging in setup script).
- Dependencies are plain pinned `requirements.txt` (no uv/pip-compile, no lock file).

### Frontend (`app/frontend/`)
- React 18.3 + TypeScript 5.5, built with **Vite 7**.
- Tailwind CSS 3.4 + Radix UI primitives + `class-variance-authority` (shadcn-style components in `src/components/ui/`), `framer-motion`, `lucide-react` icons.
- `react-use-websocket` for the `/realtime` WebSocket.
- `i18next` + `react-i18next` with locales en, es, fr, ja (`src/locales/`).
- Raw Web Audio API worklets (`public/audio-processor-worklet.js` for mic capture at 24 kHz PCM16, `public/audio-playback-worklet.js` for playback).
- Build output goes to `app/backend/static/` (see `vite.config.ts`), which the backend serves at `/`.

### Infra (`infra/`)
- Bicep, subscription-scope `main.bicep` + `main.parameters.json`, deployed with **Azure Developer CLI (azd)** per `azure.yaml`.
- Mix of Azure Verified Modules (`br/public:avm/res/...` for OpenAI, Search, Storage, Log Analytics) and local modules under `infra/core/` (container apps host, ACR access, identity, role assignments).
- App hosting: Azure Container Apps, remote Docker build (`azure.yaml: docker.remoteBuild: true`), image from `app/Dockerfile` (node:20-slim build stage -> python:3.12-slim runtime, gunicorn with `aiohttp.GunicornWebWorker` on port 8000).

### Tooling
- Ruff config in `pyproject.toml` (target py39, rules E/F/I/UP, ignores E501/E701). Ruff itself is not in requirements; install separately.
- Prettier (+ tailwind plugin) for the frontend: `npm run format`.
- No automated tests exist (backend or frontend).

## 3. Repository layout

```
aisearch-openai-rag-audio/
├── CLAUDE.md PRODUCT.md ARCHITECTURE.md CONTRIBUTING.md AGENTS.md README.md TODO.md
├── azure.yaml                     # azd config: backend service -> container app, postprovision hooks
├── pyproject.toml                 # ruff config only
├── app/
│   ├── Dockerfile                 # 2-stage: Vite build -> python:3.12-slim + gunicorn :8000
│   ├── backend/
│   │   ├── app.py                 # aiohttp app factory, env/credential wiring, routes
│   │   ├── rtmt.py                # RTMiddleTier: realtime WebSocket proxy + tool execution
│   │   ├── ragtools.py            # search + report_grounding tools over Azure AI Search
│   │   ├── setup_intvect.py       # creates index/skillset/indexer, uploads data/, runs indexer
│   │   └── requirements.txt       # NOTE: UTF-16 encoded
│   └── frontend/
│       ├── src/index.tsx          # React entrypoint (not main.tsx)
│       ├── src/App.tsx            # single-page UI: mic button, grounding files
│       ├── src/hooks/             # useRealtime, useAudioRecorder, useAudioPlayer
│       ├── src/components/ui/     # button, card, grounding-file(s|-view), status-message, history-panel (unused)
│       ├── src/locales/{en,es,fr,ja}/translation.json
│       └── public/*-worklet.js    # audio capture/playback worklets
├── infra/
│   ├── main.bicep main.parameters.json abbreviations.json
│   └── core/{host,security}/      # container-app(-upsert|s), aca-identity, registry-access, role
├── scripts/
│   ├── start.sh|ps1               # venv + npm install + build + run backend on :8765
│   ├── load_python_env.sh|ps1     # create .venv, pip install requirements
│   ├── write_env.sh|ps1           # dump azd env values -> app/backend/.env
│   └── setup_intvect.sh|ps1       # run app/backend/setup_intvect.py
├── data/                          # sample knowledge base: 6 PDFs + 1 Markdown file
├── docs/                          # existing_services.md, customizing_deploy.md, manual_setup.md, images
├── .devcontainer/                 # python:3.12 image + azd/az/node 20 features, forwards 8765
├── .github/
│   ├── CONTRIBUTING.md            # upstream Microsoft contribution/CLA doc (do not overwrite)
│   ├── copilot-instructions.md    # doc index for Copilot
│   └── workflows/
│       ├── azure-dev.yml          # azd provision + deploy on push to main (OIDC)
│       ├── template-validation.yaml  # Microsoft internal template validator (manual)
│       ├── update-claude-md.yml   # Monday docs verification + TODO regeneration
│       └── claude-md-review-prompt.md  # prompt for the Monday run
└── .vscode/                       # launch/tasks for backend + frontend
```

## 4. Architecture and data flow

Verified against the code (see `ARCHITECTURE.md` for the longer version):

1. Browser: `useAudioRecorder` captures mic audio via an AudioWorklet, base64-encodes PCM16 @ 24 kHz, and `useRealtime` sends `input_audio_buffer.append` messages over a WebSocket to `/realtime` (Vite dev server proxies `/realtime` to `ws://localhost:8765`).
2. Backend: `RTMiddleTier._websocket_handler` (rtmt.py) accepts the client socket and opens a second WebSocket to `{AZURE_OPENAI_ENDPOINT}/openai/realtime?api-version=2024-10-01-preview&deployment=...`, authenticated with either an api-key header or a bearer token from `get_bearer_token_provider` (scope `https://cognitiveservices.azure.com/.default`).
3. Messages are forwarded in both directions with interception:
   - to server: `session.update` gets the server-enforced system message, voice choice, and the tool schemas injected (`tool_choice: auto`).
   - to client: `session.created` is scrubbed (instructions/tools hidden); all `function_call` items and argument deltas are suppressed.
4. When a `response.output_item.done` carries a completed `function_call`, the middle tier executes the matching `Tool` from `ragtools.py`:
   - `search`: hybrid Azure AI Search query - full-text + `VectorizableTextQuery` (k=50) on the embedding field when `AZURE_SEARCH_USE_VECTOR_QUERY` is true, semantic ranking when `AZURE_SEARCH_SEMANTIC_CONFIGURATION` is set, top 5, formatted as `[chunk_id]: chunk` blocks. Result goes back to the model (`TO_SERVER`) as a `function_call_output` item.
   - `report_grounding`: looks up the cited chunk ids in the index and returns `{sources: [{chunk_id, title, chunk}]}` to the browser (`TO_CLIENT`) as a synthetic `extension.middle_tier_tool_response` message.
5. On `response.done` with pending tool calls, the middle tier issues `response.create` so the model continues after tool output; function_call items are stripped from the response the client sees.
6. Browser plays `response.audio.delta` chunks through the playback worklet and renders grounding files as chips (`GroundingFiles`, `GroundingFileView`).

Ingestion/indexing is a separate offline path: `azd up` postprovision hook -> `scripts/setup_intvect.sh` -> `app/backend/setup_intvect.py`, which creates (idempotently) an AI Search data source, index (`chunk_id`, `parent_id`, `title`, `chunk`, `text_vector` 3072-dim HNSW cosine + `text-embedding-3-large` vectorizer + semantic config `default`), a skillset (SplitSkill 2000 chars / 500 overlap + AzureOpenAIEmbeddingSkill), and an indexer, then uploads everything in `data/` to the blob container and runs the indexer.

## 5. Commands

There is no Makefile and no test suite. These are the commands that exist:

```bash
# One-time local setup pieces (start.sh does all of this)
python3 -m venv .venv
.venv/bin/python -m pip install -r app/backend/requirements.txt
cd app/frontend && npm install

# Run everything locally (build frontend + serve backend on http://localhost:8765)
./scripts/start.sh          # Linux/Mac
pwsh .\scripts\start.ps1    # Windows

# Frontend dev server with HMR (proxies /realtime to ws://localhost:8765)
cd app/frontend && npm run dev

# Frontend build (emits to app/backend/static/) and format
cd app/frontend && npm run build
cd app/frontend && npm run format

# Backend directly (needs app/backend/.env unless RUNNING_IN_PRODUCTION is set)
./.venv/bin/python app/backend/app.py

# Lint backend (install ruff first: pip install ruff)
ruff check app/backend

# Azure: provision + deploy + index sample data
azd auth login
azd env new
azd up
azd down                    # tear down (resources cost money immediately)

# Regenerate app/backend/.env from the current azd environment
./scripts/write_env.sh      # pwsh ./scripts/write_env.ps1 on Windows

# (Re)create the search index + upload data/ + run indexer
./scripts/setup_intvect.sh  # pwsh ./scripts/setup_intvect.ps1 on Windows
```

## 6. Environment variables

Names only, never values. Loaded from `app/backend/.env` via python-dotenv unless `RUNNING_IN_PRODUCTION` is set. `scripts/write_env.sh` generates the file from azd outputs.

Consumed by `app/backend/app.py` (runtime):

| Variable | Purpose |
|---|---|
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI resource endpoint (realtime WS base URL) |
| `AZURE_OPENAI_REALTIME_DEPLOYMENT` | gpt-4o-realtime deployment name |
| `AZURE_OPENAI_REALTIME_VOICE_CHOICE` | voice (alloy default; echo/shimmer also valid) |
| `AZURE_OPENAI_API_KEY` | optional key auth; omit to use Entra ID |
| `AZURE_SEARCH_ENDPOINT` `AZURE_SEARCH_INDEX` | AI Search service + index |
| `AZURE_SEARCH_API_KEY` | optional key auth; omit to use Entra ID |
| `AZURE_SEARCH_SEMANTIC_CONFIGURATION` | semantic config name; unset disables semantic ranking |
| `AZURE_SEARCH_IDENTIFIER_FIELD` `AZURE_SEARCH_CONTENT_FIELD` `AZURE_SEARCH_TITLE_FIELD` `AZURE_SEARCH_EMBEDDING_FIELD` | index field names (defaults: chunk_id, chunk, title, text_vector) |
| `AZURE_SEARCH_USE_VECTOR_QUERY` | "true"/"false", adds vector leg to the hybrid query |
| `AZURE_TENANT_ID` | selects AzureDeveloperCliCredential for local Entra auth |
| `RUNNING_IN_PRODUCTION` | set in the container app; disables .env loading |
| `AZURE_CLIENT_ID` | set by infra for the user-assigned managed identity |

Additionally consumed by `app/backend/setup_intvect.py` (indexing, reads the azd env): `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`, `AZURE_OPENAI_EMBEDDING_MODEL`, `AZURE_STORAGE_ENDPOINT`, `AZURE_STORAGE_CONNECTION_STRING`, `AZURE_STORAGE_CONTAINER`, `AZURE_SEARCH_REUSE_EXISTING`.

Infra-side azd variables (see `infra/main.parameters.json` and `docs/existing_services.md`): `AZURE_ENV_NAME`, `AZURE_LOCATION`, `AZURE_OPENAI_SERVICE_LOCATION`, `AZURE_OPENAI_REUSE_EXISTING`, `AZURE_OPENAI_REALTIME_DEPLOYMENT_CAPACITY`, `AZURE_OPENAI_REALTIME_DEPLOYMENT_VERSION`, `AZURE_OPENAI_EMB_DEPLOYMENT_CAPACITY`, `AZURE_SEARCH_SERVICE_SKU`, `AZURE_SEARCH_SEMANTIC_RANKER`, `AZURE_SEARCH_REUSE_EXISTING`, `AZURE_STORAGE_SKU`, `AZURE_CONTAINER_APPS_WORKLOAD_PROFILE`, and the resource-name/RG overrides listed in the parameters file.

## 7. Conventions

- Backend: async-first aiohttp, type hints on public signatures, `logging.getLogger("voicerag")`. Ruff rules E/F/I/UP; line length unenforced (E501 ignored).
- Frontend: functional components, Prettier-formatted (`npm run format`), `@/` path alias to `src/`, shadcn-style UI components in `src/components/ui/`, user-facing strings go through i18next (`src/locales/*/translation.json` - keep all four locales in sync).
- Tool results use `ToolResultDirection.TO_SERVER` (feed the model) vs `TO_CLIENT` (feed the UI); keep new tools within this pattern in `ragtools.py` + registered on `rtmt.tools`.
- Never commit secrets or `.env` files; prefer Entra ID / managed identity over API keys.
- No em dashes anywhere - single hyphen `-` only. No emojis in code or docs.

## 8. Gotchas

- **`app/backend/requirements.txt` is UTF-16 LE encoded.** pip handles it, but `grep`/`cat` show mangled text and naive edits can corrupt it. If you rewrite it, keep or deliberately normalize the encoding.
- **Deployed infra disables key auth**: `main.bicep` sets `disableLocalAuth: true` on both Azure OpenAI and AI Search, so `AZURE_OPENAI_API_KEY`/`AZURE_SEARCH_API_KEY` only work against services provisioned some other way. Use Entra ID (`azd auth login` locally, managed identity in Azure).
- **Two different ports**: local `app.py` runs on 8765; the container runs gunicorn on 8000 (Container Apps targetPort 8000). The Vite dev proxy targets 8765.
- **Frontend must be built before running the backend standalone** - `app.py` serves `app/backend/static/`, which only exists after `npm run build` (start.sh does this for you).
- **`RTMiddleTier.tools` and `_tools_pending` are class-level mutable attributes**, shared across instances. Fine with the single instance created in `app.py`, a trap if you instantiate more than one.
- **gpt-4o-realtime region list is short**: infra allows only eastus2 and swedencentral for the OpenAI resource.
- **`azd up` costs money immediately** (AI Search standard tier is the big one). `azd down` when done.
- **Index schema is convention-coupled**: `setup_intvect.py`, the `AZURE_SEARCH_*_FIELD` env vars, and `ragtools.py` must agree on field names. The grounding lookup relies on `chunk_id` being searchable with the keyword analyzer (see comment in `ragtools.py`).
- **`infra/main.parameters.json` declares `embeddingDimensions`** but `main.bicep` has no such parameter; the embedding dimension (3072) is hardcoded in `setup_intvect.py`.
- **AGENTS.md has known drift** (upstream doc): the frontend entrypoint is `src/index.tsx` (not `main.tsx`), and `data/` is mostly PDFs (not Markdown). Trust this file and the code.

## 9. Self-documentation protocol

This repository is self-documenting. Two mechanisms keep the docs honest:

1. **End of every session:** before finishing any working session that changed
   code, commands, dependencies, structure, or conventions, update CLAUDE.md (and
   the affected prep docs: PRODUCT.md, ARCHITECTURE.md, CONTRIBUTING.md) so they
   match reality. Reality wins over stale documentation. This applies to human and
   agent sessions alike.
2. **Every Monday:** the `.github/workflows/update-claude-md.yml` workflow runs an
   automated verification pass (09:00 UTC). It re-analyzes the codebase, corrects
   any drift in CLAUDE.md that session updates missed, regenerates the prioritized
   `TODO.md` at the repo root, and opens a PR for review. It requires the
   `CLAUDE_CODE_OAUTH_TOKEN` repository secret (generate with `claude setup-token`).

`TODO.md` is machine-refreshed weekly: treat it as the current backlog, edit it
freely during the week, and expect the Monday run to re-prioritize it.
