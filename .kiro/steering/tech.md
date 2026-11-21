# Tech Stack

## Monorepo

- **Package manager:** pnpm workspaces
- **Workspace config:** `pnpm-workspace.yaml` with `packages/*` and `apps/*`
- **TypeScript:** strict mode throughout; shared `tsconfig.base.json` at root

## Packages

| Package | Purpose | Key deps |
|---|---|---|
| `packages/core` | Pure accessibility logic, no browser globals | `fast-check`, `vitest` |
| `packages/overlay` | Shadow DOM injection, ghost rendering, Possess filters, Exorcise UI | `axe-core` (optional, dynamic import only) |

## Apps

| App | Purpose | Key deps |
|---|---|---|
| `apps/web` | Vite SPA, hash router, demo iframes | Vite, Vercel, Fontsource (Cinzel + Inter) |
| `apps/extension` | Chrome MV3 extension | Manifest V3 |

## Testing

- **Unit/property tests:** Vitest + `@fast-check/vitest` (runs in Node; no browser required)
- **E2E tests:** Playwright (`e2e/haunt-exorcise.spec.ts`)
- Property tests cover: contrast ratio symmetry/range, ghost opacity bounds/monotonicity, exorcise fix validity/minimality, haunting score bounds/monotonicity

## Build & Deploy

- **Web app bundler:** Vite (`apps/web/vite.config.ts`)
- **Deploy target:** Vercel
  - Build command: `pnpm --filter apps/web build`
  - Output directory: `apps/web/dist`
  - Framework preset: Vite
- **Vercel config:** `vercel.json` at repo root (or `apps/web/vercel.json`)

## Common Commands

```sh
# Install all workspace dependencies
pnpm install

# Dev server for web app
pnpm --filter apps/web dev

# Build web app
pnpm --filter apps/web build

# Run all unit and property tests
pnpm --filter packages/core test

# Run E2E tests (requires dev server running on :5173)
pnpm exec playwright test

# Preview production build
pnpm --filter apps/web preview
```

## Design Tokens

Defined in `apps/web/src/design-tokens.css` as CSS custom properties:

| Token | Value |
|---|---|
| `--color-bg` | `#1a0a2e` (near-black violet) |
| `--color-text` | `#f0ead6` (bone white) |
| `--color-accent` | `#39ff14` (ectoplasm green) |
| `--color-warning` | `#ffb347` (candlelight amber) |
| `--color-surface` | `#2a1545` |
| `--color-border` | `#4a2d6e` |
| `--font-display` | `'Cinzel', Georgia, serif` |
| `--font-body` | `'Inter', system-ui, sans-serif` |

Fonts are self-hosted via Fontsource — no Google Fonts CDN calls.

## Key Constraints

- `packages/core` must **never** import `document`, `window`, or any browser global
- All DOM access in core goes through the injected `DOMInterface` interface
- `axe-core` lives only in `packages/overlay` behind a dynamic `import()` — it must not appear in core
- All color pairs in the app shell must pass AA contrast (verified at build time by a Vitest snapshot test)
- `.kiro` must NOT be listed in `.gitignore`
