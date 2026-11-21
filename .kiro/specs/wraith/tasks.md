# Implementation Plan: Wraith

## Overview

Build Wraith in five ordered phases that mirror the hackathon brief: (1) `packages/core` with
property-based tests, (2) `packages/overlay` and demo pages, (3) `apps/web` polish and Vercel
deploy, (4) `apps/extension` (stretch goal), and (5) E2E tests, self-audit, and README. Each task
builds on the previous so there is no orphaned code.

## Tasks

- [ ] 1. Monorepo scaffold and shared types
  - Create `pnpm-workspace.yaml`, root `tsconfig.base.json` with `strict: true`, and root
    `package.json` with workspaces for `packages/*` and `apps/*`.
  - Create `packages/core/package.json`, `packages/core/tsconfig.json`, and
    `packages/overlay/package.json`, `packages/overlay/tsconfig.json`.
  - Create `apps/web/package.json` with the web app's name, Vite as a dev dependency, and
    scripts for `dev`, `build`, and `preview`.
  - Create a root `.gitignore` that excludes `node_modules`, `dist`, and `.vercel`, but
    explicitly does **NOT** list `.kiro` (per Requirement 23.5).
  - Create `packages/core/src/types.ts` with all shared types: `Rect`, `RuleName`,
    `IssueStatus`, `Issue`, `ElementRef`, `Fix`, `AuditResult`, `RuleBreakdown`,
    `GhostStyle`, `RGBAColor`, `OKLCHColor`, `PossessType`.
  - Create `packages/core/src/dom-interface.ts` exporting the `DOMInterface` interface and
    the opaque `ElementHandle` type (all 12 methods including `hasDirectText`, `isRendered`,
    `isInlineAnchor`).
  - _Requirements: 15.1, 15.2, 23.1_

- [ ] 2. Core — color utilities and contrast ratio
  - [ ] 2.1 Implement `packages/core/src/color-utils.ts`
    - Export `parseColor(css: string): RGBAColor` handling hex, `rgb()`, `rgba()`, named
      colors `transparent`, and returning `rgba(0,0,0,0)` on empty or unknown input.
    - Export `alphaComposite(over: RGBAColor, under: RGBAColor): RGBAColor`.
    - Export `relativeLuminance(c: RGBAColor): number`.
    - Export `contrastRatio(a: RGBAColor, b: RGBAColor): number` (symmetric, range [1, 21]).
    - Export OKLCH conversion pipeline: `toOKLCH`, `fromOKLCH`, `gamutClamp` (reduce C in
      steps of 0.001 at constant L and H).
    - _Requirements: 1.1, 16.1, 16.2_

  - [ ] 2.2 Write property tests for contrast ratio (Properties 1 and 2)
    - Create `packages/core/test/arbitraries.ts` with `arbRGBColor`, `arbSeverity`,
      `arbIssueSeverity`, `arbIssues`, `arbThreshold`.
    - Create `packages/core/test/contrast.property.test.ts` using `@fast-check/vitest`.
    - **Property 1: Contrast ratio symmetry** — `contrastRatio(a, b) === contrastRatio(b, a)`
    - **Property 2: Contrast ratio range** — `contrastRatio(a, b) ∈ [1, 21]`
    - **Validates: Requirements 16.1, 16.2**

- [ ] 3. Core — background resolver and contrast rules
  - [ ] 3.1 Implement `packages/core/src/rules/contrast.ts`
    - Export `resolveBackground(el, dom)` with the three-phase algorithm: collect ancestor
      chain outward (innermost first), detect `background-image`/gradient returning
      `hasGradient: true` immediately, alpha-composite outermost→innermost, composite
      semi-transparent foreground.
    - Export `contrastTextRule(dom: DOMInterface): Issue[]` applying `hasDirectText` and
      `isRendered` guards, large-text detection (24 px or 18.67 px + weight ≥ 700),
      linear severity formula, and `needs-review` status for gradients.
    - Export `contrastNonTextRule(dom: DOMInterface): Issue[]` for
      `input, select, textarea` (excluding `input[type="hidden"]`), using `border-top-color`
      vs adjacent parent background, 3:1 threshold.
    - _Requirements: 1.1–1.7, 5.2_

  - [ ]* 3.2 Write unit tests for contrast rules
    - Create `packages/core/test/rules/contrast.unit.test.ts`.
    - Fixtures: known color pairs above/below each threshold; alpha-compositing scenario;
      gradient background → `needs-review`; non-rendered element → no issue;
      large-text below 4.5 but above 3 → no issue; non-text border below 3:1 → issue.
    - _Requirements: 20.1_

- [ ] 4. Core — alt-text, tap-target, and label rules
  - [ ] 4.1 Implement `packages/core/src/rules/alt-text.ts`
    - Selector `img`, `isRendered` guard.
    - No `alt` → severity 1.0; empty non-decorative `alt` → severity 0.6; non-empty or
      `role="presentation"` / `aria-hidden="true"` → no issue.
    - _Requirements: 2.1–2.3, 5.3, 5.4_

  - [ ] 4.2 Implement `packages/core/src/rules/tap-target.ts`
    - Selector `a, button, input:not([type="hidden"]), select, [role="button"], [onclick]`.
    - `isRendered` guard; `isInlineAnchor` exception.
    - Area-based severity: `clamp(0.2 + (576 - actualArea) / 576 * 0.8, 0.2, 1.0)`.
    - _Requirements: 3.1–3.4, 5.5_

  - [ ] 4.3 Implement `packages/core/src/rules/label.ts`
    - Input check: `aria-labelledby` → `aria-label` → associated `<label>` → issue (severity 1.0).
    - Button check (button, input[type="button|submit|reset"]): innerText → `aria-label`
      → `aria-labelledby` → `value` → issue (severity 1.0).
    - Both selectors apply `isRendered` guard and skip `input[type="hidden"]`.
    - _Requirements: 4.1–4.3, 5.6_

  - [ ]* 4.4 Write unit tests for alt-text, tap-target, and label rules
    - Create `packages/core/test/rules/alt-text.unit.test.ts`.
    - Create `packages/core/test/rules/tap-target.unit.test.ts`.
    - Create `packages/core/test/rules/label.unit.test.ts`.
    - Fixtures per design: missing/empty/present alt; aria-hidden img; 23×23 (fail) / 24×24
      (pass) / 0×0 (fail, max severity); inline anchor exempt; input with label / aria-label /
      aria-labelledby valid+invalid / naked; button with text / empty / aria-labeled.
    - _Requirements: 20.2–20.4_

- [ ] 5. Core — severity, GhostStyleMapper, and ScoreEngine
  - [ ] 5.1 Implement `packages/core/src/severity.ts`
    - Export `computeContrastSeverity(ratio, threshold): number`.
    - Export `computeTapTargetSeverity(rect: Rect): number`.
    - All values clamped to [0.2, 1.0].
    - _Requirements: 5.1–5.6_

  - [ ] 5.2 Implement `packages/core/src/ghost-style.ts`
    - Export `computeGhostStyle(severity: number): GhostStyle` with:
      `elementOpacity = 1 - 0.85 * s`, `elementBlur = s * 8`,
      `glowSpread = s * 12`, `glowAlpha = 0.3 + s * 0.7`.
    - _Requirements: 6.1–6.3, 17.1, 17.2_

  - [ ] 5.3 Write property tests for ghost opacity (Properties 3 and 4)
    - Create `packages/core/test/opacity.property.test.ts`.
    - **Property 3: Ghost element opacity range** — `computeGhostStyle(s).elementOpacity ∈ [0.15, 1]`
    - **Property 4: Ghost element opacity monotonicity** — `s1 ≤ s2 ⟹ computeGhostStyle(s1).elementOpacity ≥ computeGhostStyle(s2).elementOpacity`
    - **Validates: Requirements 17.1, 17.2**

  - [ ] 5.4 Implement `packages/core/src/score.ts`
    - Export `computeHauntingScore(issues: Issue[]): number` using
      `round(100 * exp(-0.15 * weightedSum))`, integer, clamped [0, 100], returns 100 for
      empty array.
    - Export `computeBreakdown(issues: Issue[]): RuleBreakdown`.
    - _Requirements: 10.1–10.4, 19.1, 19.2_

  - [ ] 5.5 Write property tests for haunting score (Properties 7 and 8)
    - Create `packages/core/test/score.property.test.ts`.
    - **Property 7: Haunting score bounds** — `computeHauntingScore(issues) ∈ [0, 100]`; non-empty → strictly < 100
    - **Property 8: Haunting score monotonicity** — `computeHauntingScore([...A, i]) ≤ computeHauntingScore(A)`
    - **Validates: Requirements 19.1, 19.2**

- [ ] 6. Core — ExorciseFinder and Auditor
  - [ ] 6.1 Implement `packages/core/src/exorcise.ts`
    - Export `findFix(fg: RGBAColor, bg: RGBAColor, threshold: number): Fix`.
    - Bidirectional L-grid (step = 0.005), vary L only, `gamutClamp` reduces C only,
      pick candidate with smallest `|ΔL|` (then highest contrast on tie).
    - Fallback to white or black (whichever contrasts more) when no grid candidate passes.
    - _Requirements: 9.1–9.4, 9.9_

  - [ ] 6.2 Write property tests for ExorciseFinder (Properties 5 and 6)
    - Create `packages/core/test/exorcise.property.test.ts`.
    - **Property 5: Exorcise fix passes threshold** — `contrastRatio(fix, bg) ≥ threshold`
    - **Property 6: Exorcise fix minimality** — no grid color with strictly smaller `|ΔL|` also passes
    - **Validates: Requirements 18.1, 18.2**

  - [ ] 6.3 Implement `packages/core/src/auditor.ts`
    - Export `audit(dom: DOMInterface, options?: AuditOptions): AuditResult`.
    - Runs each active rule (default all five), collects `Issue[]`, calls
      `computeHauntingScore`, `computeBreakdown`.
    - No `document`, `window`, or browser globals imported.
    - _Requirements: 15.2–15.4_

- [ ] 7. Checkpoint — core tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Overlay — OverlayInjector and ghost rendering
  - [ ] 8.1 Implement `packages/overlay/src/injector.ts`
    - `OverlayInjector` class with `attachToAuditedDocument(auditedDocument)` (ghost shadow
      root on `iframe.contentDocument.body`) and `attachToShell(shellDocument)` (interactive
      UI in parent shell).
    - `renderGhosts(issues, dom)`: for each issue, resolve element via `selectorPath`, create
      ghost `<div>`, apply `elementOpacity` and `filter: blur()` as inline styles on the
      **target element** (save original to `data-wraith-original-style`), apply glow/border
      CSS on the ghost div.
    - `clearGhosts()`: restore `data-wraith-original-style` on each target element and remove
      ghost divs.
    - _Requirements: 7.1, 7.3–7.5, 6.4, 6.5_

  - [ ] 8.2 Implement `packages/overlay/src/ghost.ts`
    - `createGhostElement(issue, rect, ghostStyle)`: create `<div>` with `position: fixed`,
      `pointer-events: none`, `box-shadow` glow, border, `z-index: 2147483647`.
    - `updateGhostPosition(ghostEl, rect)`: update `top`, `left`, `width`, `height` inline.
    - Single `ResizeObserver` + `scroll` listener on `iframe.contentWindow`; batch updates
      within 100 ms.
    - _Requirements: 6.4, 7.5_

- [ ] 9. Overlay — PossessFilter and ExorciseUI
  - [ ] 9.1 Implement `packages/overlay/src/possess.ts`
    - Insert a hidden `<svg>` with `<feColorMatrix>` filter definitions into the **parent shell
      document's light DOM** (appended to `document.body` of the shell, not into any shadow root
      or iframe), so that `filter: url(#wraith-possess-{type})` applied to the `<iframe>` element
      resolves correctly from the parent frame's rendering context. Filters cover protanopia
      (Vienot 1999), deuteranopia (Vienot 1999), tritanopia (Brettel 1997).
    - `applyPossessFilter(type, iframeEl)`: set `filter: url(#wraith-possess-{type})` or
      CSS `blur(2px) contrast(0.6)` / `grayscale(1)` on the iframe element only (not shell).
    - `removePossessFilter(iframeEl)`: clear the CSS filter.
    - Only one type active at a time (replace on second call).
    - _Requirements: 8.1–8.8_

  - [ ] 9.2 Implement `packages/overlay/src/exorcise-ui.ts`
    - `ExorciseUI` panel in parent shell: original color swatch, suggested fix swatch,
      contrast ratio, "Confirm Fix" / "Cancel" buttons.
    - On render: call `findFix`, apply live preview to element in iframe document.
    - On confirm: apply `element.style.color` (contrast-text) or
      `element.style.borderColor` (contrast-non-text); trigger solidification animation
      (400 ms `@keyframes solidify`; skip if `prefers-reduced-motion`); re-run
      `computeHauntingScore` on remaining issues and update panel.
    - _Requirements: 9.5–9.8_

- [ ] 10. Overlay — ReportPanel and AxeAdapter
  - [ ] 10.1 Implement `packages/overlay/src/report-panel.ts`
    - `ReportPanel` docked in parent shell: haunting score as large numerals with gothic copy
      ("22 souls haunting this page"), per-rule breakdown table with count and mean severity,
      color-coded severity bar (ectoplasm green → amber → red).
    - Updates when `computeHauntingScore` is called after a fix.
    - _Requirements: 10.5_

  - [ ] 10.2 Implement `packages/overlay/src/axe-adapter.ts`
    - `runAxeCrossCheck(element: Element): Promise<AxeIssue[]>` using dynamic `import()` so
      axe-core tree-shakes out when unused.
    - Maps axe results to Wraith `Issue` format for display alongside core issues.
    - _Requirements: 15.5_

  - [ ] 10.3 Create `packages/overlay/styles/overlay.css`
    - Scoped ghost / panel styles injected into shadow DOM.
    - Ghost div styles (outline, glow, `pointer-events: none`).
    - `@keyframes solidify` (opacity + filter to 0 over 400 ms).
    - `@media (prefers-reduced-motion)` block skipping the animation.
    - _Requirements: 9.6, 9.7_

- [ ] 11. Demo pages
  - Create `apps/web/public/demos/cafe/index.html` for a fictional café brand with deliberate
    violations: low-contrast text, images missing alt attributes, and a button without an
    accessible name.
  - Create `apps/web/public/demos/university/index.html` for a fictional university portal with
    deliberate violations: low-contrast non-text UI, tap targets below 24 × 24, and an
    unlabeled input.
  - Create `apps/web/public/demos/shop/index.html` for a fictional shop with deliberate violations
    spanning contrast-text, alt-text, and label rules.
  - Each demo uses only fictional brand names with no real company names, logos, trademarks,
    or copyrighted assets; each covers ≥ 3 of the 5 audited rules.
  - Placing the demo files under `apps/web/public/demos/` causes Vite to copy them verbatim to
    the build output; iframe `src` attributes in `Demo.ts` must use the paths
    `/demos/cafe/index.html`, `/demos/university/index.html`, `/demos/shop/index.html`.
  - _Requirements: 12.1–12.3_

- [ ] 12. Checkpoint — overlay and demos ready
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Web app — design system and routing
  - [ ] 13.1 Create `apps/web/src/design-tokens.css`
    - Define CSS custom properties: `--color-bg: #1a0a2e`, `--color-text: #f0ead6`,
      `--color-accent: #39ff14`, `--color-warning: #ffb347`, `--color-surface: #2a1545`,
      `--color-border: #4a2d6e`.
    - `--font-display: 'Cinzel', Georgia, serif`, `--font-body: 'Inter', system-ui, sans-serif`.
    - `--motion-fog: 2s ease-in-out infinite alternate`, `--motion-grain: 0.1s steps(1) infinite`.
    - `@media (prefers-reduced-motion)` block setting both motion tokens to `none`.
    - _Requirements: 11.2, 11.3, 11.5, 22.4_

  - [ ] 13.2 Create `apps/web/src/gothic-fonts.css`
    - Self-hosted Cinzel and Inter loaded from Fontsource (no Google Fonts CDN call).
    - _Requirements: 11.3_

  - [ ] 13.3 Create `apps/web/src/main.ts` and hash router
    - Hash router resolving `/#` → `LandingPage`, `/#demo` → `DemoPage`,
      `/#paste` → `PasteHTMLPage`.
    - App shell static HTML with `<header>`, `<nav>`, `<main>`, ARIA landmark roles and
      labels on all major sections.
    - _Requirements: 22.2, 22.3_

  - [ ] 13.4 Create `apps/web/vite.config.ts`
    - Vite config for the web app; include Vercel output configuration.
    - Add a Vitest snapshot test that verifies every design token color pair meets AA contrast
      using `contrastRatio` from `packages/core`.
    - _Requirements: 11.6, 22.1_

- [ ] 14. Web app — pages
  - [ ] 14.1 Create `apps/web/src/pages/Landing.ts`
    - Headline, one-paragraph product description, call-to-action link to `/#demo`.
    - Fog and grain texture animations using `--motion-fog` and `--motion-grain` tokens;
      reduced-motion CSS applied via media query.
    - _Requirements: 11.1, 11.4, 11.5_

  - [ ] 14.2 Create `apps/web/src/pages/Demo.ts`
    - Tab switcher for café, university, and shop iframes; switching does not reload the shell.
    - Issue list and ReportPanel rendered in parent shell (not in iframes).
    - Haunt / Possess / Exorcise controls in parent shell.
    - Wires `OverlayInjector` to the active iframe's `contentDocument`.
    - The iframe `src` paths are `/demos/cafe/index.html`, `/demos/university/index.html`,
      and `/demos/shop/index.html`, rooted at the Vite public base.
    - _Requirements: 12.2, 12.4, 12.5_

  - [ ] 14.3 Create `apps/web/src/pages/PasteHTML.ts`
    - Textarea for pasting raw HTML; submit triggers sanitization pipeline:
      DOMParser pass (remove `<script>` elements) + regex sweep (strip `on*` attributes).
    - Inject sanitized HTML into `<iframe sandbox="allow-same-origin">`.
    - Wire `OverlayInjector` to the sandbox iframe; display informative message for empty
      or no-auditable-elements input; handle DOMParser failure gracefully.
    - _Requirements: 13.1–13.6_

- [ ] 15. Checkpoint — web app pages complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Deploy apps/web to Vercel
  - Add a `vercel.json` at the repo root (or `apps/web/vercel.json`) configuring:
    - Build command: `pnpm --filter apps/web build`
    - Output directory: `apps/web/dist`
    - Framework preset: Vite
  - Ensure the pnpm workspace and lockfile are committed so Vercel's install step succeeds.
  - Deploy to Vercel and verify:
    - The landing page loads at the live URL.
    - All three demo iframes (`/demos/cafe/`, `/demos/university/`, `/demos/shop/`) load and
      Haunt Mode produces ghosts on each.
    - Paste HTML mode works with no errors (sandboxed iframe, no login required).
    - The live URL requires no login or account creation for judges.
  - Add the live Vercel URL to `README.md` (replace the placeholder).
  - _Requirements: 14.8, 23.2_

- [ ] 17. E2E test
  - Create `e2e/haunt-exorcise.spec.ts` using Playwright.
  - Navigate to `http://localhost:5173/#demo`; wait for the café iframe to load.
  - Click the Haunt button in the parent app shell.
  - Assert at least one element matching `[data-wraith-ghost]` is present in the
    **iframe's document shadow root** (via `page.frameLocator` + shadow piercing into
    `iframe.contentDocument`), not in the parent page.
  - Save initial ghost count; click Exorcise in the **parent shell's issue list** for the
    first contrast issue; confirm the fix in `ExorciseUI`.
  - Assert ghost count decreased by at least 1 and HauntingScore in ReportPanel increased
    or stayed the same.
  - _Requirements: 21.1–21.4_

- [ ] 18. Self-audit and repository docs
  - [ ] 18.1 Add self-audit Vitest test
    - Write a test in `apps/web` that instantiates a mock `DOMInterface` for the app shell's
      own color pairs and asserts each passes `contrastRatio` at AA level.
    - Verify keyboard navigation paths and ARIA label presence via DOM assertions.
    - _Requirements: 22.1–22.3_

  - [ ] 18.2 Create `README.md`
    - Sections: what Wraith is, live demo URL placeholder, setup/run/test/deploy steps,
      Mermaid architecture diagram, "Third-party code" section disclosing axe-core (MPL-2.0),
      fast-check, Vite, Playwright, Cinzel (SIL OFL), Inter (SIL OFL).
    - _Requirements: 23.2_

  - [ ] 18.3 Create `KIRO_LOG.md`
    - Record where specs, hooks, steering, and property tests helped or caught bugs.
    - _Requirements: 23.3_

  - [ ] 18.4 Verify `.gitignore` does not list `.kiro`
    - Check root `.gitignore` and remove any `.kiro` entry if present.
    - _Requirements: 23.4, 23.5_

- [ ] 19. Final checkpoint — all tests pass
  - Ensure all tests pass (unit, property-based, E2E), ask the user if questions arise.

- [ ] 20. Chrome extension (stretch goal)
  - [ ] 20.1 Create `apps/extension/manifest.json`
    - Manifest V3 with `permissions: ["activeTab", "scripting"]`, action pointing to
      `popup/popup.html`, background service worker (`service_worker.js`).
    - _Requirements: 14.1_

  - [ ] 20.2 Create `apps/extension/content-script.ts`
    - Import `OverlayInjector` and `audit` from `packages/core` and `packages/overlay`.
    - Listen for `chrome.runtime.onMessage` for `{action: 'haunt'}`, `{action: 'possess', type}`,
      `{action: 'exorcise'}`; activate/deactivate accordingly.
    - Return `HauntingScore` to popup after audit.
    - _Requirements: 14.2, 14.4–14.6_

  - [ ] 20.3 Create `apps/extension/popup/popup.html` and `popup.ts`
    - Haunt button, Possess dropdown + button, Exorcise button using same design tokens.
    - Display HauntingScore returned from content script.
    - Passes its own Wraith AA audit.
    - _Requirements: 14.3, 14.4, 14.7_

  - [ ] 20.4 Create empty `apps/extension/service_worker.js`
    - Message relay only between popup and content script.
    - _Requirements: 14.1_

- [ ] 21. Checkpoint — extension builds without errors
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP.
- Each task references specific requirements for traceability.
- Checkpoints ensure incremental validation at the end of each phase.
- Property tests validate universal correctness properties; unit tests validate specific fixtures and edge cases.
- The Chrome extension (tasks 20–21) is a stretch goal; the hosted Web_App alone satisfies the hackathon entry requirement.
- Ghost `elementOpacity` and `filter: blur()` are applied to the **target element** (not the ghost div); the ghost div carries only the glow outline.
- The paste sandbox iframe has scripting disabled, so all interactive UI (ExorciseUI, ReportPanel, issue list) must render in the parent app shell.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["2.2", "3.1"] },
    { "id": 3, "tasks": ["3.2", "4.1", "4.2", "4.3"] },
    { "id": 4, "tasks": ["4.4", "5.1", "5.2"] },
    { "id": 5, "tasks": ["5.3", "5.4"] },
    { "id": 6, "tasks": ["5.5", "6.1"] },
    { "id": 7, "tasks": ["6.2", "6.3"] },
    { "id": 8, "tasks": ["7"] },
    { "id": 9, "tasks": ["8.1", "8.2"] },
    { "id": 10, "tasks": ["9.1", "9.2"] },
    { "id": 11, "tasks": ["10.1", "10.2", "10.3"] },
    { "id": 12, "tasks": ["11"] },
    { "id": 13, "tasks": ["12"] },
    { "id": 14, "tasks": ["13.1", "13.2"] },
    { "id": 15, "tasks": ["13.3", "13.4"] },
    { "id": 16, "tasks": ["14.1", "14.2", "14.3"] },
    { "id": 17, "tasks": ["15"] },
    { "id": 18, "tasks": ["16"] },
    { "id": 19, "tasks": ["17"] },
    { "id": 20, "tasks": ["18.1", "18.2", "18.3", "18.4"] },
    { "id": 21, "tasks": ["19"] },
    { "id": 22, "tasks": ["20.1", "20.2"] },
    { "id": 23, "tasks": ["20.3", "20.4"] },
    { "id": 24, "tasks": ["21"] }
  ]
}
```

