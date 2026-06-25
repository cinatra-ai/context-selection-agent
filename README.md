# Context Selection Agent

Let people choose which workspace artifacts feed into another agent's context before it runs. When a larger agent needs supporting material — an ICP, a brand voice, or a strategy document — this helper surfaces the eligible artifacts in a quick picker, locks the user's choice to a specific revision, and returns those pinned references so the calling agent's run stays reproducible later.

This is a non-uninstallable core sub-agent. A calling agent adds it to its `agentDependencies` and places an explicit context FlowNode for each context slot it needs. There is no automatic wiring: adding a context slot requires an OAS change plus the FlowNode and dependency entry.

To install, publish this package to a Cinatra instance's registry and declare it in the calling agent's manifest. The package ships a prebuilt `cinatra/oas.json` — no build step is needed. To develop or modify it, clone the repo and run `node extension-kind-gate.mjs` to run the pre-publish local gate before publishing.

Configuration is driven by the calling agent's context slot declarations. The sub-agent receives four inputs per invocation: `parentRunId`, `parentPackageName`, `slotId`, and an optional `projectId`. It calls `/api/context-resolve` to fetch candidates, opens the interactive picker when `selectionMode` is `interactive`, writes the selection audit via `/api/context-finalize`, and emits `contextSlotBindings` for the calling agent to consume as `{{context_<slotId>.contextSlotBindings}}`. Autonomous slots skip the picker and finalize immediately. Handle an empty `contextSlotBindings` array as a failure mode in the calling agent.

## Works with

- Any Cinatra agent that declares a context slot in its OAS and lists `@cinatra-ai/context-selection-agent` in its `agentDependencies`

## Capabilities

- Surface the artifacts eligible for a given context slot in an interactive picker
- Confirm the user's selection before the calling agent continues
- Pin each chosen artifact to a specific revision so the run is reproducible
- Record an audit trail of which artifacts were chosen, for which slot, and by whom
- Return the pinned references for the calling agent to consume via `contextSlotBindings`
- Support autonomous slots that skip the picker and resolve immediately without user interaction
