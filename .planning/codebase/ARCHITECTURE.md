<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌──────────────────────────────────────────────────────────────────┐
│              Calling Agent (parent, context-using)               │
│  declares @cinatra-ai/context-selection-agent in agentDependencies│
│  and adds an explicit context FlowNode per slot                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ explicit A2A invocation (per slot)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│          Context Selection Agent — Flow Agent                    │
│          `cinatra/oas.json`  (component_type: Flow)              │
│                                                                  │
│  ┌─────────┐   ┌────────────────┐   ┌──────────────┐            │
│  │  start  │──▶│resolve_context │──▶│  select_mode │            │
│  │ (inputs)│   │ (ApiNode POST) │   │ (BranchNode) │            │
│  └─────────┘   └────────────────┘   └──────┬───────┘            │
│                     │ /api/context-resolve  │                    │
│                     ▼                       │ branch: autonomous │
│              ┌──────────────────┐           ▼                    │
│              │emit_context_     │  ┌───────────────────┐         │
│              │payload           │  │finalize_autonomous│         │
│              │(OutputMsgNode)   │  │(ApiNode POST)     │         │
│              └────────┬─────────┘  └─────────┬─────────┘        │
│                       │                      │ /api/context-     │
│                       ▼                      │ finalize          │
│              ┌────────────────┐              │                   │
│              │context_select_ │              │                   │
│              │gate (HITL)     │              │                   │
│              │(InputMsgNode)  │              │                   │
│              └────────┬───────┘              │                   │
│                       │ userResponse          │                   │
│                       ▼                      │                   │
│              ┌────────────────┐              │                   │
│              │finalize_       │              │                   │
│              │interactive     │◀─────────────┘                   │
│              │(ApiNode POST)  │                                   │
│              └────────┬───────┘                                  │
│                       │ /api/context-finalize                    │
│                       ▼                                          │
│              ┌────────────────┐                                  │
│              │   end (output) │ → contextSlotBindings[]          │
│              └────────────────┘                                  │
└──────────────────────────────────────────────────────────────────┘
                             │
         ┌───────────────────┴───────────────────┐
         ▼                                       ▼
 {{CINATRA_BASE_URL}}/api/context-resolve   {{CINATRA_BASE_URL}}/api/context-finalize
 (platform API — resolves eligible            (platform API — writes audit rows,
  artifact candidates for a slot)              returns pinned contextSlotBindings)
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| `start` (StartNode) | Accept inputs: parentRunId, parentPackageName, slotId, projectId | `cinatra/oas.json` |
| `resolve_context` (ApiNode) | POST to `/api/context-resolve`; returns candidates, slotMeta, selectedRefs, selectionMode, resolutionMode | `cinatra/oas.json` |
| `select_mode` (BranchingNode) | Route to HITL path (default) or autonomous bypass path based on selectionMode value | `cinatra/oas.json` |
| `emit_context_payload` (OutputMessageNode) | Render candidates + slotMeta + selectedRefs as a JSON agent message for the HITL picker UI | `cinatra/oas.json` |
| `context_select_gate` (InputMessageNode) | HITL approval gate — surfaces the ContextSelector UI (`@cinatra-ai/context-selection-agent:context-selector`); requires user approval | `cinatra/oas.json` |
| `finalize_interactive` (ApiNode) | POST to `/api/context-finalize` with userResponse; returns pinned contextSlotBindings | `cinatra/oas.json` |
| `finalize_autonomous` (ApiNode) | POST to `/api/context-finalize` with pre-resolved selectedRefs (no user interaction); returns pinned contextSlotBindings | `cinatra/oas.json` |
| `end` (EndNode) | Emit contextSlotBindings array as output_data to the calling agent | `cinatra/oas.json` |
| `extension-kind-gate.mjs` | Standalone CI gate: validates OAS for retired primitives (agent kind) or BPMN shape (workflow kind) | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Declarative Flow Agent — the agent logic is encoded entirely as a directed graph of typed nodes (ApiNode, BranchingNode, InputMessageNode, OutputMessageNode, EndNode) in `cinatra/oas.json`. There is no imperative runtime code in this repo.

**Key Characteristics:**
- All control flow is defined as `control_flow_connections` (ControlFlowEdge entries) in `cinatra/oas.json`.
- All data routing is defined as `data_flow_connections` (DataFlowEdge entries); no implicit shared state.
- The HITL gate (`context_select_gate`) uses `requiresApproval: true` and a named renderer; the autonomous branch bypasses it entirely.
- The agent is non-uninstallable — it is a core platform sub-agent, not an optional extension.
- Invocation is explicit: calling agents must declare the dependency in `agentDependencies` AND add a FlowNode per slot; there is no runtime auto-wiring.

## Layers

**Cinatra Flow Definition Layer:**
- Purpose: Encode the complete agent logic as a typed node graph
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNodes, BranchingNode, OutputMessageNode, InputMessageNode, EndNode, plus ControlFlowEdge and DataFlowEdge wiring
- Depends on: Platform runtime that interprets `agentspec_version: 26.1.0` Flow components
- Used by: Cinatra marketplace and any calling agent that declares this package in agentDependencies

**CI Gate Layer:**
- Purpose: Pre-publish sanity checks for OAS and package shape; runs unauthenticated before the @cinatra-ai registry is reachable
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate` — all pure functions operating on file contents
- Depends on: Node.js builtins only (`node:fs`, `node:path`); zero external dependencies
- Used by: `.github/workflows/ci.yml` (kind-gates job) and the cinatra monorepo extraction script

**Package Manifest Layer:**
- Purpose: Declare agent identity, cinatra metadata, and npm package shape
- Location: `package.json`
- Contains: `cinatra.kind: "agent"`, `cinatra.type: "flow"`, `cinatra.hasApprovalGates: true`, `cinatra.riskLevel: "low"`, `cinatra.sourceTemplateId`
- Depends on: Nothing (no runtime dependencies)
- Used by: Marketplace publish/install pipeline, CI classification step

## Data Flow

### Interactive (HITL) Path

1. Calling agent invokes agent with `parentRunId`, `parentPackageName`, `slotId`, `projectId` → `start` node (`cinatra/oas.json`)
2. `resolve_context` ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/context-resolve` → receives `candidates[]`, `slotMeta`, `selectedRefs[]`, `selectionMode="interactive"`, `resolutionMode`
3. `select_mode` BranchingNode reads `selectionMode`; routes to `default` branch (not "autonomous")
4. `emit_context_payload` OutputMessageNode renders candidates + slotMeta + selectedRefs as JSON message
5. `context_select_gate` InputMessageNode pauses for user interaction via the `context-selector` HITL renderer; emits `userResponse`
6. `finalize_interactive` ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/context-finalize` with userResponse → receives `contextSlotBindings[]`
7. `end` EndNode emits `contextSlotBindings[]` to calling agent as `{{context_<slotId>.contextSlotBindings}}`

### Autonomous Path

1. Same start through `resolve_context` as above, but `selectionMode="autonomous"`
2. `select_mode` routes to `autonomous` branch — skips `emit_context_payload` and `context_select_gate`
3. `finalize_autonomous` ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/context-finalize` with pre-resolved `selectedRefs` and `resolutionMode` (no `userResponse`)
4. `end` EndNode emits `contextSlotBindings[]`

**State Management:**
- No mutable state in the agent; all data flows via explicit DataFlowEdge wiring between nodes. The platform runtime holds transient run state.

## Key Abstractions

**contextSlotBindings:**
- Purpose: Pinned artifact references (artifactId + representationRevisionId + semanticAssertionId) produced for a single context slot; consumed by the calling agent
- Examples: Output of `finalize_interactive` and `finalize_autonomous` nodes; terminal output of `end` node
- Pattern: Array of objects passed as DataFlowEdge from finalize nodes → end node

**selectionMode:**
- Purpose: Determines whether the HITL picker is shown (`"interactive"`) or bypassed (`"autonomous"`)
- Examples: Output of `resolve_context`; input to `select_mode` BranchingNode
- Pattern: String value driving BranchingNode routing via `mapping: { autonomous: "autonomous" }`

**HITL Renderer:**
- Purpose: Named UI component (`@cinatra-ai/context-selection-agent:context-selector`) rendered in the a2ui surface `context-selection:select` during the approval gate
- Examples: `context_select_gate` node metadata
- Pattern: `requiresApproval: true` + `renderer` metadata key on an InputMessageNode

## Entry Points

**Flow Agent Invocation:**
- Location: `cinatra/oas.json` → `start_node: { $component_ref: "start" }`
- Triggers: Explicit A2A call from a parent agent's FlowNode; one invocation per context slot
- Responsibilities: Accept `parentRunId`, `parentPackageName`, `slotId`, `projectId`; complete slot resolution and return `contextSlotBindings[]`

**CI Gate Entry:**
- Location: `extension-kind-gate.mjs` → `main()` function (invoked when `process.argv[1]` matches the file)
- Triggers: `node extension-kind-gate.mjs --package-root .` in the `kind-gates` CI job
- Responsibilities: Detect `cinatra.kind`, dispatch to `validateAgent` or `validateWorkflow`, exit 0/1

## Architectural Constraints

- **Threading:** Not applicable — no imperative runtime code; the flow is interpreted by the platform runtime.
- **Global state:** None. `extension-kind-gate.mjs` exports pure functions with no module-level mutable state.
- **Circular imports:** Not applicable — no import graph; the gate file uses only Node builtins.
- **Dependency constraint:** `extension-kind-gate.mjs` MUST remain zero-dependency (Node builtins only). CI runs before the @cinatra-ai registry is reachable; any external import would break unauthenticated CI.
- **First-party peer constraint:** This repo declares no `dependencies` or `devDependencies`; all @cinatra-ai/* packages must be optional peerDependencies if added (enforced by the `build` CI job classification step).
- **No standalone install/test:** Because this is a source mirror with host-internal peers, CI skips standalone install, typecheck, and test — those run in the cinatra monorepo.

## Anti-Patterns

### Wiring context slots via runtime auto-detection

**What happens:** A calling agent assumes the platform will automatically invoke this agent when a context slot exists on the calling agent.
**Why it's wrong:** There is no runtime auto-wiring. Missing the explicit FlowNode + agentDependencies entry means the slot is never resolved.
**Do this instead:** Add `@cinatra-ai/context-selection-agent` to the calling agent's `agentDependencies` AND insert an explicit context FlowNode before each slot-consuming node. (`cinatra/oas.json` description field)

### Adding @cinatra-ai/* packages to dependencies or devDependencies

**What happens:** A first-party package is added to `dependencies` or `devDependencies` in `package.json`.
**Why it's wrong:** The CI classification step in `.github/workflows/ci.yml` treats this as a shape regression and exits with code 2, failing the build.
**Do this instead:** Declare host-internal packages only as optional peerDependencies with `peerDependenciesMeta.<pkg>.optional: true`.

## Error Handling

**Strategy:** Fail-fast at CI gate boundaries; no runtime error handling in the flow definition itself (errors surface through the platform runtime).

**Patterns:**
- `extension-kind-gate.mjs` accumulates errors as `string[]` and returns them from pure validator functions; `main()` prints all violations and exits with code 1.
- Early returns on parse failure in `validateAgent` and `validateWorkflow` (return errors immediately rather than continuing on broken state).
- `validateBpmnSanity` returns on the first structural malformation (mismatched tags) but continues collecting semantic errors (missing namespace, missing process) after structure is confirmed valid.

## Cross-Cutting Concerns

**Logging:** CI gate uses `console.log` for pass messages and `console.error` for violations; no logging framework.
**Validation:** Retired-primitive scanning via regex word-boundary patterns in `extension-kind-gate.mjs`; LLM-visible fields (`system`, `user`, `description`) are the only scanned fields.
**Authentication:** Not applicable within this repo — API calls in the flow use `{{CINATRA_BASE_URL}}` which the platform runtime resolves with its own auth context.

---

*Architecture analysis: 2026-06-09*
