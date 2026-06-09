# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- TypeScript — compiled to ESNext via `tsconfig.json`; target ES2023, strict mode enabled

**Secondary:**
- JavaScript (ESM) — `extension-kind-gate.mjs` is a zero-dependency Node.js ESM script used in CI

## Runtime

**Environment:**
- Node.js 24 (specified in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (corepack-managed; CI runs `corepack enable` before any install step)
- `.npmrc` present with `auto-install-peers=false`
- No lockfile detected in repo root (source mirror — monorepo owns lockfile)

## Frameworks

**Core:**
- Cinatra Flow Agent runtime — this package is a `cinatra.kind: agent` / `type: flow` extension published to `registry.cinatra.ai`. The runtime is NOT a local dependency; it is provided by the Cinatra monorepo workspace when the repo is cloned in.

**Build/Dev:**
- TypeScript compiler (`tsc`) — configured in `tsconfig.json` to emit to `dist/`, source maps and declaration maps enabled
- No bundler explicitly declared (module resolution: `bundler` strategy in tsconfig suggests Vite or similar is expected from the host workspace)

**Testing:**
- No test framework declared in `package.json`; CI explicitly skips tests for first-party-peer repos (this is a source mirror)

## Key Dependencies

**Critical:**
- No `dependencies` or `devDependencies` declared in `package.json` — this is intentional. All `@cinatra-ai/*` and `@cinatra/*` packages that this agent depends on must appear as `peerDependencies` with `peerDependenciesMeta.optional: true` (enforced by CI gate in `.github/workflows/ci.yml`)

**Infrastructure:**
- `@cinatra-ai/context-selection-agent` itself (this package) — version `0.1.0`, Apache-2.0 license, scoped to the `cinatra-ai` npm org

## Configuration

**Environment:**
- `.env` files: not detected in repo
- Runtime configuration is provided by the Cinatra platform at agent invocation time; no local env config required for the extracted repo itself

**Build:**
- `tsconfig.json` — standalone strict TypeScript config; targets `src/`, outputs to `dist/`; `isolatedModules: true`, `verbatimModuleSyntax: true`

**Cinatra manifest fields** (from `package.json` `cinatra` block):
- `apiVersion: cinatra.ai/v1`
- `packageType: agent`
- `manifestVersion: 1`
- `sourceTemplateId: context-agent-flow`
- `riskLevel: low`
- `hasApprovalGates: true`
- `toolAccess: []` (no tool access declared)

## Platform Requirements

**Development:**
- Node.js 24+
- pnpm via corepack
- Must be cloned into the Cinatra monorepo workspace to resolve `@cinatra-ai/*` peer dependencies for typecheck/build

**Production:**
- Published to `registry.cinatra.ai` (Cinatra Marketplace) via the release workflow in `.github/workflows/release.yml`
- Marketplace submission goes through the `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret and the reusable workflow at `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`
- Pre-release tags are automatically skipped by the reusable workflow

---

*Stack analysis: 2026-06-09*
