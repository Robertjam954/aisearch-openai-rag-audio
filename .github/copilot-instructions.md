# Copilot instructions

Read these documents before making changes; they are kept in sync with the
code (see the self-documentation protocol in CLAUDE.md):

- [CLAUDE.md](../CLAUDE.md) - the operating manual: tech stack, layout, commands, env var names, conventions, and gotchas. Start here.
- [PRODUCT.md](../PRODUCT.md) - what VoiceRAG does, the problem it solves, key features, and what is deliberately out of scope.
- [ARCHITECTURE.md](../ARCHITECTURE.md) - components, the realtime WebSocket data flow, the ingestion pipeline, and how infra Bicep maps to the app.
- [CONTRIBUTING.md](../CONTRIBUTING.md) - repo-specific dev setup and PR checklist; links to [.github/CONTRIBUTING.md](CONTRIBUTING.md) for the upstream Microsoft CLA/process.
- [AGENTS.md](../AGENTS.md) - upstream coding-agent onboarding doc; useful background but CLAUDE.md wins where they disagree.

Hard style rules: single hyphen `-` only (no em dashes), no emojis, never
write secrets or environment variable values (names only).
