# ARCHITECTURE.md

Components, data flow, and the infra-to-app mapping for VoiceRAG. Grounded in
the code (`app/`) and the Bicep templates (`infra/`). See PRODUCT.md for scope
and CLAUDE.md for commands and conventions.

## Components

```
Browser (React 18 + Vite, app/frontend)
  - useAudioRecorder: mic -> AudioWorklet -> PCM16 @ 24 kHz -> base64
  - useRealtime: WebSocket client for /realtime (react-use-websocket)
  - useAudioPlayer: playback worklet for response.audio.delta chunks
  - GroundingFiles / GroundingFileView: citation chips + chunk viewer
        |
        |  WebSocket /realtime (realtime API protocol, function calls hidden)
        v
Python middle tier (aiohttp, app/backend, Azure Container Apps)
  - app.py: app factory; env + credential wiring; serves static frontend at /
  - rtmt.py (RTMiddleTier): bidirectional WebSocket proxy to Azure OpenAI,
    server-enforced session config, server-side tool execution
  - ragtools.py: 'search' and 'report_grounding' tools over Azure AI Search
        |                                   |
        |  wss /openai/realtime             |  SearchClient (aio)
        v                                   v
Azure OpenAI                         Azure AI Search index
  gpt-4o-realtime-preview              chunk_id | parent_id | title | chunk
  text-embedding-3-large               | text_vector (3072, HNSW, cosine)
  (embeddings used by the              hybrid + optional semantic ranker,
   Search vectorizer/skillset)         integrated vectorization
                                            ^
                                            |  indexer + skillset (offline)
                                     Azure Blob Storage 'content' container
                                       <- data/ uploaded by setup_intvect.py
```

## Runtime data flow (a voice turn)

1. The user clicks the mic button. The frontend sends `session.update` with
   `turn_detection: server_vad` and starts streaming
   `input_audio_buffer.append` messages to `/realtime`.
2. `RTMiddleTier` holds two sockets: client <-> backend and backend <-> Azure
   OpenAI (`{AZURE_OPENAI_ENDPOINT}/openai/realtime`, api-version
   `2024-10-01-preview`, deployment from env). Auth is an api-key header or an
   Entra bearer token (`get_bearer_token_provider`).
3. Client-to-server interception: on `session.update` the middle tier injects
   the system prompt (answer only from the knowledge base, keep it short,
   always search, always report grounding), the voice choice, and the two tool
   schemas.
4. Server-to-client interception: `session.created` is scrubbed of
   instructions/tools; `function_call` conversation items and argument deltas
   are swallowed so the browser never sees the RAG machinery.
5. When the model completes a `function_call`, the middle tier runs the tool:
   - `search(query)` (ragtools.py): hybrid query against AI Search - full-text
     plus `VectorizableTextQuery` (k=50) on `text_vector` when
     `AZURE_SEARCH_USE_VECTOR_QUERY` is true, semantic ranking when
     `AZURE_SEARCH_SEMANTIC_CONFIGURATION` is set, top 5; results serialized as
     `[chunk_id]: chunk` blocks and returned to the model as
     `function_call_output`.
   - `report_grounding(sources)`: sanitizes ids against `^[a-zA-Z0-9_=\-]+$`,
     re-fetches the cited chunks by keyword search on the id field, and emits a
     synthetic `extension.middle_tier_tool_response` message to the browser
     with `{sources: [{chunk_id, title, chunk}]}`.
6. After tool output, the middle tier sends `response.create` so the model
   finishes the answer; `function_call` entries are stripped from the final
   `response.done` payload the client receives.
7. The browser plays `response.audio.delta` chunks and renders grounding files.
   Barge-in: `input_audio_buffer.speech_started` stops local playback.

## Ingestion flow (offline)

`azd up` -> postprovision hooks (`azure.yaml`) -> `scripts/write_env.sh` then
`scripts/setup_intvect.sh` -> `app/backend/setup_intvect.py`, which idempotently
creates on the Search service (all named after `AZURE_SEARCH_INDEX`):

- a blob **data source** pointing at the `content` container;
- the **index** described above, with an Azure OpenAI vectorizer
  (`text-embedding-3-large`, 3072 dimensions) and semantic configuration
  `default`;
- a **skillset**: SplitSkill (pages of 2000 chars, 500 overlap) +
  AzureOpenAIEmbeddingSkill, with index projections keyed by `parent_id`;
- an **indexer** wiring data source -> skillset -> index.

It then uploads every file in `data/` to the container and runs the indexer.
Setting `AZURE_SEARCH_REUSE_EXISTING=true` skips all of this.

## Azure services and infra mapping (infra/main.bicep, subscription scope)

| Resource | Bicep source | Purpose / app mapping |
|---|---|---|
| Resource group | `main.bicep` | container for everything, named from `AZURE_ENV_NAME` |
| Azure OpenAI | AVM `cognitive-services/account:0.8.0` | deployments `gpt-4o-realtime-preview` (GlobalStandard) and `text-embedding-3-large`; region limited to eastus2/swedencentral; `disableLocalAuth: true` |
| Azure AI Search | AVM `search/search-service:0.7.1` | the RAG index; semantic ranker (free level by default, off on free SKU); system-assigned MI for integrated vectorization; `disableLocalAuth: true` |
| Storage account | AVM `storage/storage-account:0.9.1` | `content` blob container = knowledge base source; shared-key access disabled |
| Log Analytics | AVM `operational-insights/workspace:0.7.0` | Container Apps environment logs |
| Container Apps env + ACR | `infra/core/host/container-apps.bicep` | hosting environment; registry for the remotely built image |
| Container app `backend` | `infra/core/host/container-app-upsert.bicep` | the aiohttp app; 1 CPU / 2 Gi, targetPort 8000, user-assigned identity (`infra/core/security/aca-identity.bicep`); all `AZURE_*` runtime env vars are set here, plus `RUNNING_IN_PRODUCTION` |
| Role assignments | `infra/core/security/role.bicep` | backend MI: Cognitive Services OpenAI User + Search Index Data Reader; search MI: Storage Blob Data Reader + OpenAI User (for the skillset); deploying principal: data-plane roles on Search and Storage |

`azure.yaml` defines the single azd service `backend` (project `./app`, host
`containerapp`, `remoteBuild: true`). `app/Dockerfile` builds the frontend with
node:20-slim, copies `backend/static` plus the backend into python:3.12-slim,
and runs `gunicorn app:create_app -b 0.0.0.0:8000 --worker-class
aiohttp.GunicornWebWorker`.

Identity model: no secrets anywhere in the happy path. Locally the app uses
`AzureDeveloperCliCredential` (when `AZURE_TENANT_ID` is set) or
`DefaultAzureCredential`; deployed it uses the user-assigned managed identity
(`AZURE_CLIENT_ID`). `AZURE_OPENAI_API_KEY` / `AZURE_SEARCH_API_KEY` exist as
escape hatches for externally provisioned services only, since the template
disables local auth.

## Deployment paths

- `azd up` locally or in Codespaces/devcontainer (recommended).
- `.github/workflows/azure-dev.yml`: `azd provision` + `azd deploy` on push to
  main, using OIDC federated credentials and repository variables mirroring
  `infra/main.parameters.json`.
- `.github/workflows/template-validation.yaml`: Microsoft-internal template
  linting, manual trigger.
- Reuse of existing OpenAI/Search resources via `AZURE_OPENAI_REUSE_EXISTING`
  and `AZURE_SEARCH_REUSE_EXISTING` (docs/existing_services.md).
