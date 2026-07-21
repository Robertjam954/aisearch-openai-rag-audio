---
name: VoiceRAG (aisearch-openai-rag-audio)
description: Voice-first RAG sample - browser audio streams to a Python middle tier that proxies the Azure OpenAI GPT-4o Realtime API and executes search and grounding tools against Azure AI Search server-side.
languages:
- python
- typescript
- bicep
- azdeveloper
products:
- azure-openai
- azure-cognitive-search
- azure-container-apps
- azure-storage-accounts
page_type: sample
urlFragment: aisearch-openai-rag-audio
---
<!-- YAML front-matter schema: https://review.learn.microsoft.com/en-us/help/contribute/samples/process/onboarding?branch=main#supported-metadata-fields-for-readmemd -->

# VoiceRAG: RAG over a Voice Interface with Azure AI Search and the GPT-4o Realtime API

- [User story](#user-story)
  - [About this repo](#about-this-repo)
  - [When should you use this repo?](#when-should-you-use-this-repo)
  - [Key features](#key-features)
  - [Target end users](#target-end-users)
  - [Industry scenario](#industry-scenario)
- [Architecture](#architecture)
  - [Outputs](#outputs)
- [Deploy](#deploy)
  - [Pre-requisites](#pre-requisites)
  - [Products used](#products-used)
  - [Required licenses](#required-licenses)
  - [Pricing considerations](#pricing-considerations)
  - [Deploy instructions](#deploy-instructions)
  - [Testing the deployment](#testing-the-deployment)
- [Automated self-documentation](#automated-self-documentation)
- [Supporting documentation](#supporting-documentation)
  - [Resource links](#resource-links)
  - [Licensing](#licensing)
- [Disclaimers](#disclaimers)

## User story

### About this repo

VoiceRAG is a working sample of a voice-first Retrieval-Augmented Generation application. A user opens a web page, presses one button, and talks. The browser streams microphone audio over a WebSocket to a small Python (aiohttp) middle tier, which proxies the session to the Azure OpenAI GPT-4o Realtime API. The middle tier injects a server-enforced system prompt and two function tools (`search` and `report_grounding`) that query Azure AI Search; tool calls execute server-side and are hidden from the browser, so the client only ever sees audio, transcripts, and citation payloads. Answers come back as streamed spoken audio plus grounding citations rendered as clickable file chips.

This is a fork of [Azure-Samples/aisearch-openai-rag-audio](https://github.com/Azure-Samples/aisearch-openai-rag-audio), kept as a portfolio repo. The sample knowledge base is a fictional Contoso Electronics HR/benefits corpus in `data/`. This README is updated at the end of each working session and verified by the automated Monday documentation workflow.

### When should you use this repo?

- You are building a voice interface over private data and want a reference implementation of the realtime-API-plus-tools pattern before writing your own middle tier.
- You want to see how to keep RAG machinery (system prompt, tools, search credentials) server-side while the browser speaks the realtime protocol directly.
- You need a small Azure footprint, deployable with one command and destroyable in minutes, to evaluate GPT-4o Realtime grounded on Azure AI Search.

Out of scope by design: user authentication, per-user ACLs, conversation persistence, document upload from the UI, eval harnesses, and production hardening. See [PRODUCT.md](PRODUCT.md).

### Key features

- **Realtime WebSocket proxy** (`app/backend/rtmt.py`, class `RTMiddleTier`): terminates the browser's `/realtime` WebSocket, opens a second WebSocket to the Azure OpenAI Realtime API, and intercepts traffic in both directions - injecting the server-enforced system prompt, voice choice, and tool schemas into `session.update`, and scrubbing instructions, tools, and all `function_call` items from what the client receives. Output: the streamed audio response the browser plays, with the RAG machinery invisible to the client.
- **Server-side `search` tool** (`app/backend/ragtools.py`): hybrid Azure AI Search query - full text plus a `VectorizableTextQuery` (k=50) on the embedding field when `AZURE_SEARCH_USE_VECTOR_QUERY` is true, semantic ranking when `AZURE_SEARCH_SEMANTIC_CONFIGURATION` is set, top 5 results. Output: `[chunk_id]: chunk` blocks returned to the model as a `function_call_output` item so it answers only from the knowledge base.
- **Server-side `report_grounding` tool** (`app/backend/ragtools.py`): re-fetches the chunks the model cites and pushes `{sources: [{chunk_id, title, chunk}]}` to the browser as a synthetic `extension.middle_tier_tool_response` message. Output: the citation chips (`GroundingFiles` / `GroundingFileView`) the user clicks to verify what the voice said.
- **Integrated-vectorization indexing** (`app/backend/setup_intvect.py`, run by the `azd up` postprovision hook or `scripts/setup_intvect.sh`): idempotently creates an AI Search data source, index (`chunk_id`, `parent_id`, `title`, `chunk`, `text_vector` 3072-dim HNSW cosine with a `text-embedding-3-large` vectorizer and semantic config `default`), skillset (SplitSkill 2000 chars / 500 overlap + AzureOpenAIEmbeddingSkill), and indexer, then uploads everything in `data/` to the blob container and runs the indexer. Output: the populated search index and the `content` blob container.
- **Audio worklets** (`app/frontend/public/audio-processor-worklet.js`, `audio-playback-worklet.js`): capture mic audio as PCM16 at 24 kHz for `input_audio_buffer.append` messages and play `response.audio.delta` chunks, with barge-in (playback stops when the user starts speaking). Output: the live full-duplex voice conversation in the browser.
- **Static frontend build** (`app/frontend/`, React 18 + TypeScript + Vite 7 + Tailwind, localized en/es/fr/ja): `npm run build` emits into `app/backend/static/`, which the backend serves at `/`. Output: the single-page app with mic button, status messages, and grounding chips.
- **One-command infrastructure** (`infra/main.bicep` + `azure.yaml`): subscription-scope Bicep deployed with azd, provisioning Azure OpenAI (gpt-4o-realtime-preview + text-embedding-3-large), Azure AI Search, Blob Storage, Log Analytics, and an Azure Container App built remotely from `app/Dockerfile`. Output: the running app URL printed by `azd up`, secretless by default (managed identity; key auth disabled on provisioned services).

### Target end users

- Developers evaluating or building voice interfaces over private data: internal helpdesks, HR/benefits assistants, kiosk or hands-free scenarios, call-center prototypes.
- Teams who want a minimal, readable implementation of the realtime middle-tier pattern.

### Industry scenario

The shipped demo is an HR/benefits assistant: employees ask spoken questions about health plans, perks, and policies from the fictional Contoso Electronics corpus and get spoken answers with citations. This fork is maintained as a portfolio/reference repo, not a supported product.

## Architecture

```
                                  Browser (React 18 + Vite)
                 +---------------------------------------------------------+
                 |  mic --> audio-processor worklet (PCM16 @ 24 kHz)       |
                 |  speaker <-- audio-playback worklet                     |
                 |  citation chips (GroundingFiles / GroundingFileView)    |
                 +----------------------------+----------------------------+
                                              |
                                              | WebSocket /realtime
                                              | (function calls hidden from client)
                                              v
                 +---------------------------------------------------------+
                 |  Python middle tier (aiohttp, Azure Container Apps)     |
                 |  app.py ......... app factory, env/credential wiring,   |
                 |                   serves static frontend at /           |
                 |  rtmt.py ........ RTMiddleTier: bidirectional WS proxy, |
                 |                   server-enforced prompt + tool config, |
                 |                   server-side tool execution            |
                 |  ragtools.py .... 'search' + 'report_grounding' tools   |
                 +--------------+---------------------------+--------------+
                                |                           |
              wss /openai/realtime                SearchClient (aio)
                                |                           |
                                v                           v
                 +-----------------------+    +-----------------------------+
                 |  Azure OpenAI         |    |  Azure AI Search index      |
                 |  gpt-4o-realtime-     |    |  chunk_id | parent_id |     |
                 |    preview            |    |  title | chunk |            |
                 |  text-embedding-3-    |    |  text_vector (3072, HNSW)   |
                 |    large (used by the |    |  hybrid + semantic ranker,  |
                 |    Search vectorizer) |    |  integrated vectorization   |
                 +-----------------------+    +--------------+--------------+
                                                             ^
                                                             | indexer + skillset
                                                             | (offline ingestion)
                                              +--------------+--------------+
                                              |  Azure Blob Storage         |
                                              |  'content' container        |
                                              |  <-- data/ uploaded by      |
                                              |      setup_intvect.py       |
                                              +-----------------------------+
```

A voice turn: the frontend streams `input_audio_buffer.append` messages to `/realtime`; `RTMiddleTier` forwards them to Azure OpenAI while injecting session config on the way in and scrubbing tool machinery on the way out. When the model completes a `function_call`, the middle tier runs the matching tool from `ragtools.py` - `search` results feed the model, `report_grounding` results feed the browser - then issues `response.create` so the model finishes the spoken answer. Ingestion is a separate offline path: `azd up`'s postprovision hooks run `setup_intvect.py`, which builds the index/skillset/indexer and uploads `data/` to blob storage. Full detail in [ARCHITECTURE.md](ARCHITECTURE.md).

### Outputs

| Artifact | Where it lands |
|---|---|
| Streamed audio answers + live transcripts | browser session (played via the playback worklet) |
| Grounding citation payloads (`extension.middle_tier_tool_response`) | browser session (rendered as clickable chips) |
| Azure AI Search index + data source + skillset + indexer | the provisioned AI Search service (created by `setup_intvect.py`) |
| Uploaded knowledge-base documents | `content` blob container (from `data/`) |
| Static frontend build | `app/backend/static/` (from `npm run build`) |
| Local runtime env file | `app/backend/.env` (generated by `scripts/write_env.sh`, gitignored) |
| Azure resources (OpenAI, Search, Storage, Log Analytics, Container App) | resource group created by `azd up` |

## Deploy

### Pre-requisites

- An Azure subscription with access to Azure OpenAI (gpt-4o-realtime-preview is region-limited: the infra allows eastus2 and swedencentral).
- [Azure Developer CLI (azd)](https://aka.ms/azure-dev/install)
- Node.js 20
- Python 3.11+ (the Dockerfile and devcontainer use 3.12)
- Git; PowerShell 7 for the `.ps1` scripts on Windows
- Alternatively, the included `.devcontainer/` provides all tools (forwards port 8765).

### Products used

- Azure OpenAI Service (gpt-4o-realtime-preview, text-embedding-3-large)
- Azure AI Search (hybrid + semantic ranking, integrated vectorization)
- Azure Container Apps (+ Azure Container Registry, remote build)
- Azure Blob Storage
- Azure Log Analytics

### Required licenses

None beyond an Azure subscription. The repo itself is MIT licensed.

### Pricing considerations

`azd up` creates resources that incur costs immediately, primarily the Azure AI Search standard tier; Azure OpenAI is billed per token, Container Apps per consumption, Storage and Log Analytics per usage. Costs accrue even if you interrupt the deployment. Run `azd down` (or delete the resource group) when finished.

### Deploy instructions

Deploy to Azure (provisions everything, builds and deploys the container, creates the index, ingests `data/`):

```bash
azd auth login
azd env new          # names the resource group / environment
azd up               # provision + deploy + postprovision indexing hooks
azd down             # tear down when done
```

Optional pre-`azd up` customization: reuse existing services or change the voice via azd environment variables - see [docs/existing_services.md](docs/existing_services.md) and [docs/customizing_deploy.md](docs/customizing_deploy.md).

Run locally (after `azd up` has written `app/backend/.env`, or with your own services per [docs/existing_services.md](docs/existing_services.md)):

```bash
./scripts/start.sh           # Linux/Mac: venv + npm install + build + backend on http://localhost:8765
pwsh .\scripts\start.ps1     # Windows
```

Note the two ports: local `app.py` serves on 8765; the deployed container runs gunicorn on 8000 (Container Apps targetPort 8000). For frontend HMR, `cd app/frontend && npm run dev` proxies `/realtime` to `ws://localhost:8765`. Regenerate the local env file with `./scripts/write_env.sh`; re-run indexing with `./scripts/setup_intvect.sh`.

### Testing the deployment

1. Open the URL printed by `azd up` (or `http://localhost:8765` locally).
2. Click the start/mic button and say "Hello", then ask a question about the sample data, e.g. "What is included in the Northwind Health Plus plan?" or "What is Contoso's whistleblower policy?".
3. Expect a spoken audio answer and one or more grounding citation chips below it; clicking a chip opens the retrieved source chunk.
4. Ask something outside the corpus - the assistant should say it does not know rather than hallucinate.

## Automated self-documentation

This repository keeps its own documentation current on a fixed loop:

- End of every working session: CLAUDE.md, this README, `app/backend/requirements.txt`, and any affected prep docs are updated to match reality.
- Every Monday at 09:00 UTC: the GitHub Actions workflow [`update-claude-md.yml`](.github/workflows/update-claude-md.yml) runs Claude Code with the prompt in [`claude-md-review-prompt.md`](.github/workflows/claude-md-review-prompt.md). It verifies CLAUDE.md and this README against the code, checks `app/backend/requirements.txt` against actual imports, regenerates the prioritized [TODO.md](TODO.md), and opens a pull request with any corrections. It can also be triggered manually from the Actions tab.
- The workflow requires the `CLAUDE_CODE_OAUTH_TOKEN` repository secret (generate with `claude setup-token`).

## Supporting documentation

### Resource links

- Prep docs: [PRODUCT.md](PRODUCT.md) (what and why), [ARCHITECTURE.md](ARCHITECTURE.md) (components and data flow), [CONTRIBUTING.md](CONTRIBUTING.md) (dev setup), [CLAUDE.md](CLAUDE.md) (agent operating manual), [AGENTS.md](AGENTS.md) (upstream onboarding doc, known drift), [TODO.md](TODO.md) (machine-refreshed weekly backlog)
- Guides: [docs/existing_services.md](docs/existing_services.md), [docs/customizing_deploy.md](docs/customizing_deploy.md), [docs/manual_setup.md](docs/manual_setup.md)
- Upstream: [Azure-Samples/aisearch-openai-rag-audio](https://github.com/Azure-Samples/aisearch-openai-rag-audio), [VoiceRAG blog post](https://aka.ms/voicerag), [demo video](https://youtu.be/vXJka8xZ9Ko)
- [Azure OpenAI Realtime SDK samples](https://github.com/Azure-Samples/aoai-realtime-audio-sdk/)

### Licensing

This repository is licensed under the [MIT License](LICENSE) (Copyright (c) 2024 Azure Samples).

## Disclaimers

This is sample/portfolio code provided as-is, without warranty of any kind. It is a pattern demonstration, not a production-hardened product: there is no authentication, no content safety filtering, no rate limiting, and no automated test suite. You are responsible for all costs of any Azure resources you provision; `azd up` starts billing immediately and `azd down` removes it. The sample documents in `data/` are fictional, LLM-generated Contoso Electronics content for demonstration only; the repository contains no real personal or sensitive data, and none should be added.
