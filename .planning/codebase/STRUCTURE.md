# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
context-selection-agent/
├── cinatra/                  # Cinatra platform artifacts (OAS definition)
│   └── oas.json              # Authoritative flow agent spec (agentspec v26.1.0)
├── .github/
│   └── workflows/
│       ├── ci.yml            # Standalone CI: build + kind-gates jobs
│       └── release.yml       # Release workflow
├── extension-kind-gate.mjs   # Zero-dependency CI gate for agent/workflow kinds
├── package.json              # npm manifest + cinatra metadata block
├── tsconfig.json             # TypeScript config (no TS sources currently)
├── .npmrc                    # npm registry config
├── LICENSE                   # Apache-2.0
└── README.md                 # User-facing description and capabilities
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Houses all Cinatra platform artifacts for this extension
- Contains: `oas.json` — the complete flow agent definition including all nodes, control-flow edges, data-flow edges, and referenced components
- Key files: `cinatra/oas.json`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipelines
- Contains: `ci.yml` (build + kind-gates), `release.yml` (publish)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: The agent's complete runtime definition. The `start_node` field references the `start` component, which is the flow's entry point. This is what the Cinatra platform executes.
- `extension-kind-gate.mjs`: CI gate entry point. Exports pure validator functions and a `main()` that runs when invoked directly via `node extension-kind-gate.mjs --package-root .`.

**Configuration:**
- `package.json`: npm package identity plus the `cinatra` metadata block (`kind`, `apiVersion`, `type`, `riskLevel`, `hasApprovalGates`, `sourceTemplateId`, `sourceVersionId`).
- `tsconfig.json`: TypeScript configuration; present for tooling compatibility even though the repo currently has no `.ts` source files.
- `.npmrc`: npm/pnpm registry configuration.

**Core Logic:**
- `cinatra/oas.json`: All agent logic (resolve → branch → HITL or autonomous → finalize → output).
- `extension-kind-gate.mjs`: All CI validation logic (OAS banned-primitive scan for agents, BPMN shape validation for workflows).

**Testing:**
- No test files detected. Tests for the gate functions run in the cinatra monorepo, not in this extracted repo (CI skips standalone test due to host-internal peers).

## Naming Conventions

**Files:**
- Cinatra platform artifacts live in `cinatra/` with fixed names: `oas.json` (agent), `workflow.bpmn` (workflow kind — not present here).
- The CI gate file is named `extension-kind-gate.mjs` — this name is fixed by the cinatra monorepo extraction script contract; do not rename it.
- CI workflows use kebab-case: `ci.yml`, `release.yml`.

**Directories:**
- `cinatra/` — always lowercase, singular; mandated by the platform sidecar convention.
- `.github/workflows/` — standard GitHub Actions location.

**Nodes in `cinatra/oas.json`:**
- Node IDs use snake_case: `resolve_context`, `select_mode`, `emit_context_payload`, `context_select_gate`, `finalize_interactive`, `finalize_autonomous`.
- Edge names use snake_case describing the connection: `start_to_resolve`, `resolve_to_select`, `gate_to_finalize_interactive`.

**Exported functions in `extension-kind-gate.mjs`:**
- camelCase verbs: `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `parseArgs`.

## Where to Add New Code

**New flow node:**
- Add the node definition to `$referenced_components` in `cinatra/oas.json`.
- Add a `$component_ref` entry to the `nodes` array.
- Wire control flow via a new `ControlFlowEdge` in `control_flow_connections`.
- Wire data via new `DataFlowEdge` entries in `data_flow_connections`.

**New CI validation rule:**
- Add to `extension-kind-gate.mjs` inside the appropriate existing validator (`validateAgent` for OAS checks, `validateBpmnSanity` for BPMN checks) or add a new exported pure function and call it from `runGate`.
- Keep all additions zero-dependency (Node builtins only).

**New input/output to the agent:**
- Add to the `inputs`/`outputs` arrays on the top-level OAS object AND on the `start`/`end` nodes in `$referenced_components`.
- Add corresponding DataFlowEdge entries.

**New GitHub Actions step:**
- Add to `.github/workflows/ci.yml` in the appropriate job (`build` for package-shape checks, `kind-gates` for OAS/BPMN validation).

## Special Directories

**`cinatra/`:**
- Purpose: Platform artifact sidecar — `oas.json` is the agent's published specification consumed by the Cinatra marketplace
- Generated: `oas.json` is generated/maintained by the cinatra monorepo compilation pipeline; do not hand-edit nodes without understanding the full DataFlowEdge implications
- Committed: Yes — `cinatra/oas.json` is committed and validated in CI

**`.planning/codebase/`:**
- Purpose: GSD codebase analysis documents for planning and execution tooling
- Generated: Yes (by gsd-map-codebase)
- Committed: Project discretion

---

*Structure analysis: 2026-06-09*
