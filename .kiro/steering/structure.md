# Project Structure

## Monorepo Layout

```
wraith/
├── packages/
│   ├── core/                         # Pure accessibility logic (no browser globals)
│   │   ├── src/
│   │   │   ├── types.ts              # Shared types: Rect, Issue, Fix, AuditResult, etc.
│   │   │   ├── dom-interface.ts      # DOMInterface + ElementHandle (the only DOM abstraction)
│   │   │   ├── auditor.ts            # Orchestrates all rules → AuditResult
│   │   │   ├── color-utils.ts        # parseColor, contrastRatio, OKLCH conversion pipeline
│   │   │   ├── severity.ts           # computeContrastSeverity, computeTapTargetSeverity
│   │   │   ├── ghost-style.ts        # computeGhostStyle(severity) → GhostStyle
│   │   │   ├── score.ts              # computeHauntingScore, computeBreakdown
│   │   │   ├── exorcise.ts           # findFix — OKLCH lightness grid search
│   │   │   └── rules/
│   │   │       ├── contrast.ts       # contrast-text + contrast-non-text rules + resolveBackground
│   │   │       ├── alt-text.ts       # alt-text rule
│   │   │       ├── tap-target.ts     # tap-target rule
│   │   │       └── label.ts          # label rule
│   │   └── test/
│   │       ├── arbitraries.ts        # fast-check arbitraries (arbRGBColor, arbSeverity, etc.)
│   │       ├── contrast.property.test.ts
│   │       ├── opacity.property.test.ts
│   │       ├── exorcise.property.test.ts
│   │       ├── score.property.test.ts
│   │       └── rules/
│   │           ├── contrast.unit.test.ts
│   │           ├── alt-text.unit.test.ts
│   │           ├── tap-target.unit.test.ts
│   │           └── label.unit.test.ts
│   └── overlay/                      # Browser-side: shadow DOM, ghosts, filters, UI panels
│       ├── src/
│       │   ├── injector.ts           # OverlayInjector — attaches shadow root, renders ghosts
│       │   ├── ghost.ts              # createGhostElement, updateGhostPosition, ResizeObserver
│       │   ├── possess.ts            # PossessFilter — SVG feColorMatrix + CSS filter
│       │   ├── exorcise-ui.ts        # ExorciseUI panel (fix preview + confirm/cancel)
│       │   ├── report-panel.ts       # ReportPanel (haunting score + per-rule breakdown)
│       │   └── axe-adapter.ts        # Optional axe-core cross-check (dynamic import only)
│       └── styles/
│           └── overlay.css           # Ghost/panel styles for injection into shadow DOM
├── apps/
│   ├── web/                          # Vite SPA — primary deliverable
│   │   ├── src/
│   │   │   ├── main.ts               # App entry, hash router (#, #demo, #paste)
│   │   │   ├── design-tokens.css     # CSS custom properties (colors, fonts, motion)
│   │   │   ├── gothic-fonts.css      # Self-hosted Cinzel + Inter via Fontsource
│   │   │   └── pages/
│   │   │       ├── Landing.ts        # Landing page with fog/grain animations
│   │   │       ├── Demo.ts           # iframe tab switcher (café, university, shop)
│   │   │       └── PasteHTML.ts      # Sanitize + sandbox iframe for pasted markup
│   │   ├── public/
│   │   │   └── demos/                # Static demo pages (copied verbatim by Vite)
│   │   │       ├── cafe/index.html
│   │   │       ├── university/index.html
│   │   │       └── shop/index.html
│   │   └── vite.config.ts
│   └── extension/                    # Chrome MV3 extension (stretch goal)
│       ├── manifest.json
│       ├── content-script.ts
│       ├── service_worker.js
│       └── popup/
│           ├── popup.html
│           └── popup.ts
├── e2e/
│   └── haunt-exorcise.spec.ts        # Playwright E2E test
├── pnpm-workspace.yaml
├── tsconfig.base.json                # Shared TS config (strict: true)
├── vercel.json                       # Vercel deploy config
├── README.md
├── KIRO_LOG.md
└── LICENSE                           # MIT, do not modify
```

## Key Boundaries

- **`packages/core`** — zero browser dependencies; all DOM access via `DOMInterface`; testable in pure Node
- **`packages/overlay`** — browser-only; wraps `document`/`window`; adapts to `DOMInterface` for core calls
- **`apps/web`** — wires overlay to iframe documents; all interactive UI (ExorciseUI, ReportPanel, issue list) lives in the **parent shell**, not inside iframes
- **Demo pages** live under `apps/web/public/demos/` so Vite copies them as static assets; iframe `src` paths are `/demos/cafe/index.html`, etc.
- **Ghost elements** attach to `iframe.contentDocument.body` shadow root; **UI panels** attach to parent shell document

## Naming Conventions

- TypeScript files use camelCase filenames (`color-utils.ts`, `ghost-style.ts`)
- Test files are colocated under `test/` within each package, mirroring `src/` structure
- Property tests: `*.property.test.ts`; unit tests: `*.unit.test.ts`
- CSS custom properties follow `--color-*`, `--font-*`, `--motion-*` prefixes
- Ghost data attributes on DOM elements: `data-wraith-ghost`, `data-wraith-original-style`
- SVG filter IDs: `wraith-possess-{type}` (e.g., `wraith-possess-protanopia`)
