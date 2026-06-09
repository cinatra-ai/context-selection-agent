# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra Platform API (internal):**
- `/api/context-resolve` — called by the `resolve_context` flow node. Resolves a context slot (`slotId`) for a given `parentRunId` / `parentPackageName` / `projectId` to a set of eligible artifact candidates.
- `/api/context-finalize` — called by both `finalize_interactive` and `finalize_autonomous` flow nodes. Writes `run_context_selections` audit rows and finalises the chosen pinned artifact refs for the slot.
  - Auth: provided by the Cinatra platform runtime at invocation time (not a caller-supplied secret)
  - Source: `cinatra/oas.json` (nodes: `resolve_context`, `finalize_interactive`, `finalize_autonomous`)

**Cinatra Marketplace / Registry:**
- `registry.cinatra.ai` — target publish registry for the extension package
- Submission goes through the `CINATRA_MARKETPLACE_VENDOR_TOKEN` GitHub org secret and the reusable release workflow (`cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`)
- Source: `.github/workflows/release.yml`

## Data Storage

**Databases:**
- Not applicable — this agent does not directly access a database. Audit writes (`run_context_selections` rows) are persisted by the Cinatra platform via `/api/context-finalize`.

**File Storage:**
- Not applicable

**Caching:**
- Not applicable

## Authentication & Identity

**Auth Provider:**
- Cinatra platform runtime — authentication for all `/api/*` calls is injected by the platform at agent invocation time. The agent itself declares no auth configuration; credentials are never held in this repo.

## Monitoring & Observability

**Error Tracking:**
- Not detected — no third-party error tracking SDK is declared

**Logs:**
- Runtime logs are handled by the Cinatra platform; no local logging library is configured

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace / `registry.cinatra.ai` (private registry)

**CI Pipeline:**
- GitHub Actions — two workflows:
  - `.github/workflows/ci.yml` — runs on push/PR to `main`; validates package shape (first-party dep classification), runs `extension-kind-gate.mjs` for kind-specific OAS/BPMN sanity checks, installs + typechecks + tests only for truly standalone repos
  - `.github/workflows/release.yml` — triggers on GitHub Release publication or `workflow_dispatch` from a tag; delegates to `cinatra-ai/.github` reusable workflow for build, pack, gate, and marketplace submission

## Environment Configuration

**Required env vars:**
- None required in the extracted repo itself. The Cinatra platform supplies runtime API credentials at agent invocation.
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — GitHub org-level secret used only by the release workflow; never present in this repo's code

**Secrets location:**
- GitHub org secrets (not in-repo)

## Webhooks & Callbacks

**Incoming:**
- Not applicable — this agent is invoked by the Cinatra platform runtime as an A2A sub-agent (explicit FlowNode in calling agent), not via webhook

**Outgoing:**
- Not applicable — agent communicates only via the Cinatra platform API calls (`/api/context-resolve`, `/api/context-finalize`)

## HITL (Human-in-the-Loop) Screens

**ContextSelector:**
- Screen ID: `@cinatra-ai/context-selection-agent:context-selector`
- Triggered when `selectionMode=interactive` (autonomous slots skip this gate via branch)
- Purpose: surfaces eligible artifacts in an interactive picker for user confirmation before proceeding
- Source: `cinatra/oas.json` → `metadata.cinatra.hitlScreens` and `context_select_gate` node

---

*Integration audit: 2026-06-09*
