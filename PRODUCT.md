# PRODUCT.md

What VoiceRAG is, the problem it solves, and what is in and out of scope. See
ARCHITECTURE.md for how it is built and CLAUDE.md for the operating manual.

## What it does

VoiceRAG is a working sample of a voice-first RAG (Retrieval-Augmented
Generation) application. A user opens a web page, presses one button, and talks.
Their speech streams to Azure OpenAI's GPT-4o Realtime API through a small
Python middle tier; the model answers questions **only** from a private
knowledge base indexed in Azure AI Search, and the answer comes back as spoken
audio within the same session, along with citations to the exact source chunks
it used.

The shipped knowledge base is a fictional Contoso Electronics HR corpus (health
plan PDFs, employee handbook, role library, company overview) under `data/`,
but any documents dropped into the blob container are picked up by the same
indexing pipeline.

## The problem it solves

Realtime voice models are conversational but ungrounded: on their own they
hallucinate and know nothing about private data. Classic RAG stacks are
grounded but text-first, with latency and orchestration patterns that do not
fit a live audio session. VoiceRAG demonstrates the pattern that joins the two:

1. **Grounding a realtime voice model.** The backend injects a strict system
   prompt and two function tools into every session. The model must call
   `search` before answering and `report_grounding` to cite sources; if the
   knowledge base has no answer, it says it does not know.
2. **Keeping the RAG machinery server-side.** The browser never sees the system
   prompt, the tools, or the search credentials. The `RTMiddleTier` proxy
   intercepts the realtime protocol, executes tool calls on the server, and
   strips them from what the client receives. This is the part most teams get
   wrong when they connect a browser directly to the realtime API.
3. **Trustworthy answers in an audio medium.** Because spoken answers cannot
   carry footnotes, citations arrive as a parallel data channel: the UI shows
   the actual retrieved chunks as clickable "grounding files" so a user can
   verify what the voice just said.

## Key features

- **One-button voice conversation** with echo-resistant turn handling
  (server-side voice activity detection; playback stops when the user starts
  speaking).
- **Hybrid retrieval**: full-text + vector search (via AI Search integrated
  vectorization, `text-embedding-3-large`) with optional semantic ranker,
  top-5 chunks per query.
- **Citations UI**: per-answer grounding chips that open the underlying chunk
  text in a viewer.
- **Configurable voice** (alloy default; echo and shimmer supported) and
  localized UI (English, Spanish, French, Japanese).
- **One-command deployment**: `azd up` provisions Azure OpenAI, AI Search,
  Blob Storage, Log Analytics, and Azure Container Apps, builds the container
  remotely, deploys it, creates the search index with integrated
  vectorization, and ingests the sample documents.
- **Secretless by default**: Entra ID locally, managed identity in Azure;
  key-based auth is disabled on the provisioned services.
- **Bring-your-own services**: documented paths for reusing an existing Azure
  OpenAI deployment or an existing index (including one built by
  azure-search-openai-demo).

## Users and use cases

- Developers evaluating or building voice interfaces over private data:
  internal helpdesks, HR/benefits assistants, kiosk or hands-free scenarios,
  call-center prototypes.
- Teams who want a reference implementation of the realtime-API-plus-tools
  pattern before writing their own middle tier.

## Scope

**In scope**
- The realtime middle-tier pattern (proxy, server-enforced config, server-side
  tool execution) and a minimal, readable implementation of it.
- A complete but small Azure footprint deployable and destroyable in minutes.
- Sample data and an automated indexing pipeline good enough to demo RAG
  quality features (hybrid + semantic ranking).

**Out of scope (deliberately not built)**
- User authentication, per-user document ACLs, multi-tenancy.
- Conversation persistence or chat history (a `history-panel` UI component
  exists but is not wired up).
- Document upload from the UI; ingestion is offline via blob storage + indexer.
- Evaluation harnesses, content safety filtering, rate limiting, cost caps.
- Production hardening in general: this is a sample/pattern, not a product SKU.

## Success criteria

- `azd up` on a clean subscription yields a working spoken Q&A app over the
  sample corpus with no manual steps.
- Answers are grounded: the model declines questions outside the knowledge
  base and every substantive answer carries at least one citation.
- The pattern is portable: swapping the index, fields, voice, or model
  deployment requires only environment variables, not code changes.
