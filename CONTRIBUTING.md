# Contributing

This repo is a fork of the Azure-Samples VoiceRAG sample. For the upstream
contribution process, CLA, and code-of-conduct requirements, see
[.github/CONTRIBUTING.md](.github/CONTRIBUTING.md) (Microsoft's document, kept
verbatim). This file covers the repo-specific development workflow.

## Dev setup

Prerequisites: Python >= 3.11, Node.js 20+, Azure Developer CLI (azd), Git.
The devcontainer (`.devcontainer/`) has all of these preinstalled.

```bash
# Backend
python3 -m venv .venv
.venv/bin/python -m pip install -r app/backend/requirements.txt

# Frontend
cd app/frontend && npm install && cd ../..

# Environment: after `azd up`, generate app/backend/.env
./scripts/write_env.sh

# Run everything (builds frontend, serves backend at http://localhost:8765)
./scripts/start.sh
```

For frontend work with hot reload, run `npm run dev` in `app/frontend` (it
proxies `/realtime` to `ws://localhost:8765`, so keep the backend running).

## Before opening a PR

There is no test suite yet, so the bar is: it must run.

- `ruff check app/backend` passes (`pip install ruff` if needed).
- `cd app/frontend && npm run format` applied, and `npm run build` succeeds.
- Backend starts cleanly: `./.venv/bin/python app/backend/app.py`.
- Manually exercise a voice turn if you touched audio, websocket, or search
  code; confirm grounding chips still appear.
- No secrets or `.env` files committed; env var names only in docs.
- Infra changes are reflected in `infra/main.bicep` and
  `infra/main.parameters.json` together.

## Conventions

- Follow [CLAUDE.md](CLAUDE.md) sections 7-8 (conventions and gotchas); it is
  the operating manual and is kept current at the end of every working session.
- Single hyphen `-` only, never em dashes. No emojis in code or docs.
- Update the prep docs (CLAUDE.md, and PRODUCT.md / ARCHITECTURE.md when
  behavior or structure changes) in the same PR as the code change - see the
  self-documentation protocol in CLAUDE.md.
- Watch the encoding of `app/backend/requirements.txt` (UTF-16 LE) when
  editing dependencies.
