# Testing Patterns

**Analysis Date:** 2026-06-09

## Overview

This is a minimal extracted Cinatra extension repo (a "source mirror"). No test files are present in the repository. The `package.json` declares no `test` script and no test framework dependency. CI skips standalone tests for source-mirror repos (those with host-internal `@cinatra-ai/*` optional peer dependencies) — tests are run by the cinatra monorepo, not here.

## Test Framework

**Runner:** Not applicable — no test runner configured in this repo.

**Assertion Library:** Not applicable.

**Run Commands:**
```bash
# CI command (skipped for source mirrors):
corepack pnpm test --if-present
# Because no test script is defined, this is a no-op.
```

## Test File Organization

**Location:** No test files detected under any path in this repo.

**Naming:** Not applicable.

## Test Structure

Not applicable — no tests exist in this repo.

## Mocking

Not applicable.

## Fixtures and Factories

Not applicable.

## Coverage

**Requirements:** None enforced.

**View Coverage:** Not configured.

## Test Types

**Unit Tests:** Not present in this repo. The cinatra monorepo is expected to provide unit tests for the exported functions in `extension-kind-gate.mjs` (e.g., `validateAgent`, `validateBpmnSanity`, `validateWorkflowPackageShape`, `runGate`).

**Integration Tests:** Not present.

**E2E Tests:** Not present.

## CI Gate (Substitute for Tests)

The primary quality gate in this repo is the `extension-kind-gate.mjs` script itself, run by CI in `.github/workflows/ci.yml`. It validates `cinatra/oas.json` against banned retired CRM primitives for `kind: "agent"` repos. This is a structural correctness check, not a unit test suite.

**CI validation steps:**
1. First-party dep shape check (inline Node.js script in CI) — validates no `@cinatra-ai/*` packages leaked into `dependencies`/`devDependencies`.
2. `node extension-kind-gate.mjs --package-root .` — validates `cinatra/oas.json` for banned primitives.
3. `npm pack --dry-run` — validates publishable package shape.

**Testable exports (for monorepo tests):**
All validation logic is exported as named pure functions from `extension-kind-gate.mjs`:
- `parseArgs` — argument parsing
- `validateAgent` — OAS banned-primitive scan
- `validateWorkflowPackageShape` — workflow package.json shape check
- `validateBpmnSanity` — light BPMN XML well-formedness check
- `findWorkflowSidecars` — recursive BPMN sidecar discovery
- `validateWorkflow` — full workflow extension validation
- `runGate` — top-level dispatcher

---

*Testing analysis: 2026-06-09*
