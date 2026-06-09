# Coding Conventions

**Analysis Date:** 2026-06-09

## Overview

This is a minimal extracted Cinatra extension repo. The sole source file is `extension-kind-gate.mjs` — a self-contained, zero-dependency Node.js ES module gate script. There is no `src/` directory and no TypeScript source files committed. Conventions are derived from `extension-kind-gate.mjs`, `package.json`, `tsconfig.json`, and `.github/workflows/ci.yml`.

## Naming Patterns

**Files:**
- kebab-case with `.mjs` extension for standalone ES module scripts: `extension-kind-gate.mjs`
- kebab-case for directories: `cinatra/`, `.planning/`

**Functions:**
- camelCase for all exported and internal functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `walkLlmStrings`, `scanOasString`, `wordBoundary`
- Descriptive verb-noun names: `validateWorkflowPackageShape`, `findWorkflowSidecars`

**Variables:**
- camelCase for local variables: `packageRoot`, `findings`, `bpmnPrefixes`, `openTags`
- SCREAMING_SNAKE_CASE for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`, `OBJECTS_LIST_CRM_RE`

**Types:**
- No TypeScript sources committed; `tsconfig.json` is present for future use or monorepo integration. Type-like shapes conveyed via JSDoc comments (e.g., `/** Pure: returns string[] errors. */`).

## Code Style

**Formatting:**
- No `.prettierrc` or `eslint.config.*` detected — no formatter config committed.
- Indentation: 2 spaces (consistent in `extension-kind-gate.mjs`).
- Double quotes for strings throughout.
- Trailing commas on multi-line arrays/objects (ES2017+ style).
- Semicolons used consistently.

**Linting:**
- No ESLint or Biome config detected. Linting is not enforced in CI for this repo.

## Import Organization

**Order (as seen in `extension-kind-gate.mjs`):**
1. Node built-in modules only — `node:fs` and `node:path` with explicit `node:` prefix.

**Path Aliases:**
- Not applicable — no bundler, no aliases. Plain `node:` built-in imports only.

**Module System:**
- ES modules (`"type": "module"` in `package.json`).
- `import` / `export` syntax; no CommonJS `require` in source (CI inline scripts use `require` only in Node.js `-e` one-liners within shell).

## Error Handling

**Patterns:**
- Functions return `string[]` error arrays rather than throwing. Example: `validateAgent(packageRoot)` and `validateBpmnSanity(xml)` return `errors` arrays.
- `try/catch` wraps all file I/O (`readFileSync`, `readdirSync`). Caught errors are pushed as strings into the errors array and the function returns early: `errors.push(\`…: \${err instanceof Error ? err.message : String(err)}\`); return errors;`
- `main()` is wrapped in a top-level `try/catch` that catches unexpected errors and calls `process.exit(1)`.
- Exit codes are documented: `0` = pass, `1` = violations found.

## Logging

**Framework:** `console.log` / `console.error` (stdlib only)

**Patterns:**
- Success messages to `console.log`.
- Violation lists and errors to `console.error`.
- Bullet-point format for violations: `• ${e}`.

## Comments

**When to Comment:**
- File-level block comment explains purpose, scope, limitations, and usage (lines 1–34 of `extension-kind-gate.mjs`).
- Section dividers use `// ----------` banners to separate logical sections.
- JSDoc-style `/** … */` on every exported function explaining what it does and its purity contract.
- Inline comments explain non-obvious decisions (e.g., why `npx` is used instead of `pnpm dlx`).

## Function Design

**Size:** Functions are focused and small-to-medium. Complex logic (BPMN XML walk) is kept in one function (~80 lines) with inline comments explaining each phase.

**Parameters:** Functions take a single `packageRoot` string or a single value (e.g., `validateBpmnSanity(xml)`, `validateWorkflowPackageShape(pkg)`). No options objects.

**Return Values:** Pure functions return `string[]` errors. The dispatcher `runGate` returns `{ kind, errors }`. No thrown exceptions in the happy path.

**Purity:** Functions are documented as pure where applicable. Side effects (file I/O, `process.exit`) are isolated to `validateAgent`, `validateWorkflow`, `findWorkflowSidecars`, `runGate`, and `main`.

## Module Design

**Exports:** Named exports only — no default export. All validation functions and helpers are exported for testability: `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`.

**Entry Point Guard:** `main()` is only called when the file is invoked directly (checked via `import.meta.url` vs `process.argv[1]`), allowing the module to be imported without side effects.

## Self-Containment Constraint

A hard constraint documented in the file header: **zero external dependencies**. Only `node:fs` and `node:path` built-ins are permitted. This is enforced by design (no `package.json` `dependencies` field) and validated by CI's first-party dep shape check.

---

*Convention analysis: 2026-06-09*
