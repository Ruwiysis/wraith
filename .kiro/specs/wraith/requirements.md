# Requirements Document

## Introduction

Wraith is a haunted accessibility auditor submitted to the Kiroween 2026 hackathon (Costume Contest
category). It audits web pages for WCAG 2.2 violations and renders failing elements as ghosts: each
ghost's opacity, blur, and spectral glow encode how inaccessible that element is. The "spooky
effect IS the information." Wraith ships as a hosted demo web app and a Chrome Manifest V3
extension, sharing a pure-logic core and a shadow-DOM overlay package.

The repo is a pnpm monorepo with TypeScript strict mode throughout. Task order follows the brief:
core + property tests → overlay + demo pages → web app polish + deploy → Chrome extension →
E2E tests + self-audit + README.

## Glossary

- **Wraith**: The product as a whole.
- **Core**: The `packages/core` package — pure accessibility logic with no direct DOM access.
- **Overlay**: The `packages/overlay` package — injects ghost visuals, Possess filters, and Exorcise UI into a document via shadow DOM.
- **Web_App**: The `apps/web` Vite application hosted on Vercel.
- **Extension**: The `apps/extension` Chrome Manifest V3 browser extension.
- **Auditor**: The subsystem inside Core that runs the four custom pure rules (contrast-text, contrast-non-text, alt-text, tap-target, label) with no axe-core dependency.
- **Ghost**: A visual overlay element rendered by Overlay on top of a failing DOM element.
- **Severity**: A floating-point value in [0, 1] representing how inaccessible an element is, where 1 is maximally inaccessible.
- **Ghost_Opacity**: The CSS opacity applied to a Ghost, computed as `1 - 0.85 * severity`.
- **Haunt_Mode**: The operating mode in which the Auditor runs checks and the Overlay draws Ghosts on failing elements.
- **Possess_Mode**: The operating mode that applies full-page vision-simulation filters.
- **Exorcise**: The workflow that suggests and optionally applies a contrast-passing foreground color fix for a failing element.
- **Haunting_Score**: A 0–100 integer representing how accessible a page is; higher is better.
- **Report_Panel**: The gothic UI panel that displays the Haunting_Score and per-rule breakdown.
- **WCAG**: Web Content Accessibility Guidelines 2.2.
- **AA**: WCAG Level AA conformance.
- **Contrast_Ratio**: The luminance-based ratio defined in WCAG 2.2 success criterion 1.4.3, in the range [1, 21].
- **OKLCH**: The OKLCH perceptual color space (lightness, chroma, hue), used for minimal-change fix search.
- **Issue**: A single accessibility violation found on one element, produced by the Auditor. An Issue carries an optional `status` field: `"needs-review"` when the Auditor cannot definitively determine a failure (e.g., because the element has a gradient background); otherwise the status is `"fail"` or absent.
- **Rule**: A named check (e.g., `contrast-text`, `alt-text`, `tap-target`, `label`) that produces zero or more Issues.
- **Fix**: A suggested replacement foreground color that passes the required Contrast_Ratio.
- **Demo_Page**: One of three deliberately inaccessible pages for fictional brands, embedded in the Web_App.
- **Sandbox_iframe**: A same-origin iframe with `sandbox="allow-same-origin"` used to audit pasted HTML.
- **Shadow_DOM**: The browser's Shadow DOM used by Overlay to isolate its styles from the host page.
- **prefers-reduced-motion**: The CSS media query indicating the user has requested reduced motion.
- **axe-core**: The open-source accessibility engine (MPL-2.0) used in `packages/overlay` as an optional browser-side cross-check only; Core does not depend on axe-core.

---

## Requirements

### Requirement 1: Accessibility Auditing — Contrast (WCAG 1.4.3 and 1.4.11)

**User Story:** As a developer, I want the Auditor to flag text and non-text elements whose
foreground/background contrast ratio falls below WCAG AA thresholds, so that I can identify and
fix contrast failures.

#### Acceptance Criteria

1. WHEN the Auditor checks a text element, THE Auditor SHALL compute the Contrast_Ratio between the
   computed foreground color and the computed background color.
2. WHEN the computed font size is at least 18 pt (24 CSS px) or at least 14 pt bold (approximately
   18.67 CSS px bold), THE Auditor SHALL apply the large-text threshold of 3:1; otherwise THE
   Auditor SHALL apply the normal-text threshold of 4.5:1.
3. WHEN the Contrast_Ratio of a text element is below the applicable threshold, THE Auditor SHALL
   produce an Issue for that element with Rule `contrast-text`.
4. WHEN the Auditor checks a non-text UI component or graphical object, THE Auditor SHALL apply the
   non-text threshold of 3:1 (WCAG 1.4.11).
5. WHEN the Contrast_Ratio of a non-text UI component is below 3:1, THE Auditor SHALL produce an
   Issue for that element with Rule `contrast-non-text`.
6. WHEN the Auditor computes the effective background color of an element, THE Auditor SHALL walk
   ancestor elements and alpha-composite their background colors.
7. IF an ancestor element has a `background-image` or CSS gradient, THE Auditor SHALL produce an
   Issue with Rule `contrast-text` or `contrast-non-text` and status `needs-review` rather than a
   definitive failure, since the exact background color cannot be determined.

---

### Requirement 2: Accessibility Auditing — Alt Text (WCAG 1.1.1)

**User Story:** As a developer, I want the Auditor to detect images with missing or empty alt
attributes, so that screen-reader users receive meaningful descriptions.

#### Acceptance Criteria

1. WHEN an `<img>` element has no `alt` attribute, THE Auditor SHALL produce an Issue for that
   element with Rule `alt-text` and Severity 1.0.
2. WHEN an `<img>` element has an `alt` attribute whose trimmed value is the empty string and the
   element is not marked with `role="presentation"` or `aria-hidden="true"`, THE Auditor SHALL
   produce an Issue for that element with Rule `alt-text` and Severity 0.6.
3. WHEN an `<img>` element has a non-empty `alt` attribute or is explicitly decorative
   (`role="presentation"` or `aria-hidden="true"`), THE Auditor SHALL produce no Issue for that
   element under Rule `alt-text`.

---

### Requirement 3: Accessibility Auditing — Tap Targets (WCAG 2.5.8)

**User Story:** As a developer, I want the Auditor to flag interactive elements whose tap target
is smaller than the WCAG 2.5.8 minimum, so that touch users can activate them reliably.

#### Acceptance Criteria

1. WHEN an interactive element (links, buttons, inputs, selects, or elements with a click handler)
   has a rendered bounding box smaller than 24 CSS px in width or 24 CSS px in height, THE Auditor
   SHALL produce an Issue for that element with Rule `tap-target`.
2. THE Auditor SHALL compute tap-target dimensions from the element's bounding client rect provided
   through the injected DOM interface.
3. WHEN an interactive element meets or exceeds 24 × 24 CSS px, THE Auditor SHALL produce no Issue
   for that element under Rule `tap-target`.
4. WHEN an interactive element is a hyperlink (`<a>` element) that is inline within a sentence or
   block of text (i.e., surrounded by non-interactive text content in the same block container),
   THE Auditor SHALL NOT produce a `tap-target` Issue for that element, in accordance with the
   WCAG 2.5.8 inline exception.

---

### Requirement 4: Accessibility Auditing — Unlabeled Inputs and Buttons (WCAG 4.1.2)

**User Story:** As a developer, I want the Auditor to flag inputs and buttons that lack an
accessible name, so that assistive technology can announce them correctly.

#### Acceptance Criteria

1. WHEN an `<input>` element (excluding `type="hidden"`) has no associated `<label>`, no
   `aria-label` attribute, and no `aria-labelledby` referencing a non-empty element, THE Auditor
   SHALL produce an Issue for that element with Rule `label`.
2. WHEN a `<button>` or `<input type="button|submit|reset">` has no accessible name (empty inner
   text, no `aria-label`, no `aria-labelledby` referencing a non-empty element, and no `value`
   attribute for input buttons), THE Auditor SHALL produce an Issue for that element with Rule
   `label`.
3. WHEN an `<input>` or `<button>` has a valid accessible name, THE Auditor SHALL produce no Issue
   for that element under Rule `label`.

---

### Requirement 5: Severity Scoring

**User Story:** As a developer, I want each Issue to carry a Severity in [0, 1], so that the
ghost visualization can scale its visual weight proportionally.

#### Acceptance Criteria

1. THE Auditor SHALL assign a Severity value in the closed interval [0, 1] to every Issue; every
   Issue SHALL have a Severity of at least 0.2 so that every Ghost is always visible.
2. WHEN computing Severity for a contrast Issue, THE Auditor SHALL map Severity linearly from 0.2
   (when the Contrast_Ratio is just below the applicable threshold) to 1.0 (when the
   Contrast_Ratio is 1:1).
3. WHEN computing Severity for an alt-text Issue where the `alt` attribute is absent, THE Auditor
   SHALL assign Severity 1.0.
4. WHEN computing Severity for an alt-text Issue where the `alt` attribute is present but empty
   and not decorative, THE Auditor SHALL assign Severity 0.6.
5. WHEN computing Severity for a tap-target Issue, THE Auditor SHALL derive Severity from the
   ratio of the missing area relative to the minimum required area of 576 CSS px² (24 × 24), capped
   at 1.0.
6. WHEN computing Severity for a label Issue, THE Auditor SHALL assign Severity 1.0.

---

### Requirement 6: Ghost Style Mapping

**User Story:** As a developer, I want the Overlay to map each Issue's Severity to visual ghost
properties, so that viewers can read inaccessibility at a glance.

#### Acceptance Criteria

1. THE Overlay SHALL compute Ghost_Opacity for each Ghost as `1 - 0.85 * severity`, yielding
   values in [0.15, 1].
2. THE Overlay SHALL apply a CSS blur to each Ghost that scales with Severity; a Severity of 0
   SHALL produce no blur and a Severity of 1 SHALL produce a blur of no more than 8 CSS px.
3. THE Overlay SHALL apply a spectral glow (CSS `box-shadow` or `drop-shadow`) to each Ghost that
   increases with Severity.
4. THE Overlay SHALL position each Ghost to exactly cover the bounding rect of its associated
   failing element, updating position when the page is scrolled or resized.
5. WHEN Haunt_Mode is deactivated, THE Overlay SHALL remove all Ghost elements from the Shadow_DOM.

---

### Requirement 7: Haunt Mode

**User Story:** As a user, I want to activate Haunt Mode on a page and immediately see ghost
overlays on every failing element, so that I can perceive the accessibility quality of the page
visually.

#### Acceptance Criteria

1. WHEN the user activates Haunt_Mode, THE Overlay SHALL invoke the Auditor and render a Ghost for
   every Issue returned.
2. WHEN the Auditor returns no Issues, THE Overlay SHALL display a confirmation message indicating
   no accessibility ghosts were found.
3. WHEN the user deactivates Haunt_Mode, THE Overlay SHALL remove all Ghosts and the confirmation
   message.
4. THE Overlay SHALL inject all Ghost elements inside a Shadow_DOM root attached to the host
   document body, so that Ghost styles do not leak into or inherit from the host page.
5. WHEN Haunt_Mode is active and the host page layout changes, THE Overlay SHALL recompute and
   update Ghost positions within 100 ms.

---

### Requirement 8: Possess Mode — Vision Simulations

**User Story:** As a user, I want to simulate how a page appears to users with color-vision
deficiencies or low vision, so that I can empathize with those users and spot contrast problems.

#### Acceptance Criteria

1. WHEN the user activates Possess_Mode with simulation type `protanopia`, THE Overlay SHALL apply
   an SVG color-matrix filter to the host page that approximates protanopia vision.
2. WHEN the user activates Possess_Mode with simulation type `deuteranopia`, THE Overlay SHALL
   apply an SVG color-matrix filter that approximates deuteranopia vision.
3. WHEN the user activates Possess_Mode with simulation type `tritanopia`, THE Overlay SHALL
   apply an SVG color-matrix filter that approximates tritanopia vision.
4. WHEN the user activates Possess_Mode with simulation type `low-vision`, THE Overlay SHALL apply
   a CSS blur of 2 px combined with a contrast reduction filter to the host page.
5. WHEN the user activates Possess_Mode with simulation type `grayscale`, THE Overlay SHALL apply a
   CSS grayscale(1) filter to the host page.
6. WHEN the user deactivates Possess_Mode, THE Overlay SHALL remove all vision-simulation filters
   from the host page.
7. THE Overlay SHALL allow only one Possess_Mode simulation type to be active at a time; activating
   a second type SHALL replace the first.
8. WHEN Possess_Mode is active in the Web_App, THE Overlay SHALL apply the vision-simulation
   filter only to the active demo or Paste HTML iframe element, and SHALL NOT apply any filter to
   the Web_App shell or landing page.

---

### Requirement 9: Exorcise — Contrast Fix Search

**User Story:** As a developer, I want Wraith to suggest the nearest-passing foreground color for
a contrast failure, so that I can fix it with the smallest perceptual change to the design.

#### Acceptance Criteria

1. WHEN the user triggers Exorcise on an Issue with Rule `contrast-text` or `contrast-non-text`,
   THE Core SHALL search OKLCH color space for the foreground color nearest to the original that
   produces a Contrast_Ratio at or above the applicable threshold.
2. THE Core SHALL vary OKLCH lightness only when searching for a Fix, keeping hue constant and
   reducing chroma only as needed to stay within the sRGB gamut; THE Core SHALL NOT adjust hue.
3. THE Core SHALL return a Fix color that, when used as the foreground against the original
   background, produces a Contrast_Ratio at or above the applicable threshold (4.5:1 for normal
   text, 3:1 for large text or non-text).
4. WHEN no Fix is found within the OKLCH search space, THE Core SHALL return a fallback Fix of
   pure white or pure black, whichever has the higher Contrast_Ratio against the background.
5. THE Overlay SHALL display a live preview of the Fix color applied to the failing element before
   the user confirms.
6. WHEN the user confirms a Fix, THE Overlay SHALL apply the Fix color as an inline style on the
   element and play a "solidification" animation on the Ghost.
7. WHEN `prefers-reduced-motion` is active, THE Overlay SHALL skip the solidification animation
   and apply the Fix immediately without transition.
8. WHEN a Fix is applied to an element, THE Overlay SHALL remove the Ghost for that element.
9. THE Core SHALL ensure minimality: no other passing color at the same or smaller |ΔL| step
   (absolute change in OKLCH lightness) SHALL exist in the search grid for the returned Fix.

---

### Requirement 10: Haunting Score

**User Story:** As a user, I want a 0–100 score and a per-rule breakdown shown in a gothic report
panel, so that I can compare pages and track accessibility improvement over time.

#### Acceptance Criteria

1. THE Core SHALL compute the Haunting_Score as an integer in [0, 100] based on the weighted count
   and severity of Issues found.
2. WHEN no Issues are found, THE Core SHALL return a Haunting_Score of 100.
3. WHEN one or more Issues are found, THE Core SHALL return a Haunting_Score strictly less than 100.
4. THE Core SHALL provide a per-rule breakdown object mapping each Rule name to the count of Issues
   and the mean Severity for that Rule.
5. THE Overlay SHALL render the Haunting_Score and per-rule breakdown in the Report_Panel using
   gothic-flavored copy (e.g., "Souls trapped: 12").
6. WHEN the Haunting_Score is recomputed after a Fix is applied, THE Core SHALL return a score
   greater than or equal to the score before the Fix.

---

### Requirement 11: Web App — Gothic Landing Page

**User Story:** As a visitor, I want to land on a visually striking gothic page that explains what
Wraith is and links to the demo, so that I am immediately engaged and understand the product.

#### Acceptance Criteria

1. THE Web_App SHALL display a landing page with a headline, a one-paragraph product description,
   and a call-to-action link to the demo section.
2. THE Web_App SHALL use a near-black violet background, bone-white body text, ectoplasm-green
   accent color, and candlelight amber for warnings, consistently across all pages.
3. THE Web_App SHALL use an open-licensed display serif for headings and a clean sans-serif for
   body text.
4. THE Web_App SHALL include subtle fog or grain texture and tasteful motion on the landing page,
   with no abrupt or startling animations.
5. WHEN `prefers-reduced-motion` is active, THE Web_App SHALL disable or reduce all decorative
   animations to static or cross-fade equivalents.
6. THE Web_App SHALL pass its own Wraith audit at AA level for every color pair, keyboard
   navigation path, ARIA label, and motion preference.

---

### Requirement 12: Web App — Demo Pages

**User Story:** As a user, I want to audit three deliberately inaccessible demo pages for fictional
brands inside the Web_App, so that I can see Wraith working on realistic content.

#### Acceptance Criteria

1. THE Web_App SHALL provide exactly three Demo_Pages for fictional brands: a café, a university
   portal, and a shop; none SHALL use real company names, logos, trademarks, or copyrighted assets.
2. Each Demo_Page SHALL be embedded in a same-origin iframe within the Web_App.
3. Each Demo_Page SHALL contain deliberate accessibility violations spanning at least three of the
   four audited Rules (contrast-text, alt-text, tap-target, label).
4. WHEN the user activates Haunt_Mode on a Demo_Page, THE Overlay SHALL render Ghosts inside the
   iframe's document via the injected DOM interface.
5. THE Web_App SHALL allow the user to switch between Demo_Pages without reloading the outer
   application shell.

---

### Requirement 13: Web App — Paste HTML Mode

**User Story:** As a developer, I want to paste arbitrary HTML into Wraith and audit it, so that I
can test markup that is not yet deployed to a live URL.

#### Acceptance Criteria

1. THE Web_App SHALL provide a text area where the user can paste raw HTML markup.
2. WHEN the user submits pasted markup, THE Web_App SHALL inject it into a Sandbox_iframe and
   invoke the Auditor against the iframe's document.
3. THE Sandbox_iframe SHALL use `sandbox="allow-same-origin"` and SHALL NOT allow scripts,
   popups, top-level navigation, or form submission outside the iframe.
4. WHEN the Auditor returns Issues for pasted markup, THE Overlay SHALL render Ghosts inside the
   Sandbox_iframe.
5. WHEN the pasted markup is empty or contains no auditable elements, THE Web_App SHALL display an
   informative message rather than an empty ghost overlay.
6. BEFORE injecting pasted markup, THE Web_App SHALL strip all `<script>` elements and all inline
   event-handler attributes (any attribute whose name begins with `on`) from the pasted HTML.

---

### Requirement 14: Chrome Extension

**User Story:** As a developer, I want a Chrome extension that lets me run Haunt, Possess, and
Exorcise on any page I am browsing, so that I can audit real sites without deploying a demo app.
Note: the Chrome extension is a stretch goal; the hosted Web_App alone satisfies the hackathon
entry requirement and judges can test it without any login.

#### Acceptance Criteria

1. THE Extension SHALL be a Chrome Manifest V3 extension that declares the minimum necessary
   permissions.
2. THE Extension SHALL inject Core and Overlay into the active tab's content script context.
3. THE Extension popup SHALL provide buttons for Haunt_Mode, Possess_Mode (with simulation type
   selection), and Exorcise.
4. WHEN the user clicks the Haunt button in the Extension popup, THE Extension SHALL activate
   Haunt_Mode in the active tab and display the Haunting_Score in the popup.
5. WHEN the user clicks the Possess button in the Extension popup, THE Extension SHALL apply the
   selected vision simulation to the active tab.
6. WHEN the user clicks the Exorcise button in the Extension popup, THE Extension SHALL open the
   Exorcise UI for the first unresolved contrast Issue in the active tab.
7. THE Extension popup UI SHALL pass the Wraith audit at AA level.
8. THE Web_App SHALL be accessible to hackathon judges without any login, account creation, or
   local installation.

---

### Requirement 15: Packages/Core — Injected DOM Interface

**User Story:** As a developer, I want Core to access the DOM only through an injected interface
and implement only four custom pure rules with no axe-core dependency, so that it can be tested in
a Node environment without a real browser and its logic remains framework-independent.

#### Acceptance Criteria

1. THE Core SHALL define a `DOMInterface` TypeScript interface that abstracts all DOM queries
   needed by the Auditor (element enumeration, computed style access, bounding rect retrieval,
   attribute access).
2. THE Core SHALL accept a `DOMInterface` instance as a parameter to all audit functions and SHALL
   NOT import or reference `document`, `window`, or any browser global directly.
3. WHEN tests provide a mock `DOMInterface`, THE Core SHALL produce deterministic results based
   solely on mock data.
4. THE Core SHALL implement only the four custom pure rules (contrast-text, contrast-non-text,
   alt-text, tap-target, label) and SHALL NOT depend on axe-core or any other third-party
   accessibility engine.
5. WHERE axe-core cross-checking is desired, an optional axe-core adapter SHALL reside in
   `packages/overlay` as a browser-side supplement only, with no dependency on the Core package.

---

### Requirement 16: Property-Based Tests — Contrast Ratio

**User Story:** As a developer, I want property-based tests verifying mathematical invariants of
the Contrast_Ratio function, so that I can trust it is correct for all color inputs.

#### Acceptance Criteria

1. THE Core test suite SHALL include a Vitest + fast-check property test asserting that
   `contrastRatio(a, b) === contrastRatio(b, a)` for all color pairs (symmetry).
2. THE Core test suite SHALL include a property test asserting that `contrastRatio(a, b)` is always
   in the closed interval [1, 21] for all valid color inputs.

---

### Requirement 17: Property-Based Tests — Ghost Opacity

**User Story:** As a developer, I want property-based tests verifying that Ghost_Opacity is
monotone and bounded, so that the visual encoding is always valid.

#### Acceptance Criteria

1. THE Core test suite SHALL include a property test asserting that `ghostOpacity(s)` is in the
   closed interval [0.15, 1] for all Severity values in [0, 1].
2. THE Core test suite SHALL include a property test asserting that if `s1 <= s2` then
   `ghostOpacity(s1) >= ghostOpacity(s2)` (non-increasing as Severity rises) for all s1, s2 in
   [0, 1].

---

### Requirement 18: Property-Based Tests — Exorcise Fix Color

**User Story:** As a developer, I want property-based tests verifying that every suggested Fix
passes the required Contrast_Ratio and minimises perceptual change, so that the fix search is
correct and trustworthy.

#### Acceptance Criteria

1. THE Core test suite SHALL include a property test asserting that for any background color and
   applicable threshold, the Fix color returned by the Exorcise search produces a Contrast_Ratio
   at or above the threshold.
2. THE Core test suite SHALL include a property test asserting that no color in the OKLCH search
   grid with a smaller |ΔL| (absolute OKLCH lightness change) than the returned Fix also passes the
   required threshold (minimality at search resolution).

---

### Requirement 19: Property-Based Tests — Haunting Score

**User Story:** As a developer, I want property-based tests verifying that the Haunting_Score is
bounded and monotone, so that adding an issue never improves the score.

#### Acceptance Criteria

1. THE Core test suite SHALL include a property test asserting that `hauntingScore(issues)` is
   always in the closed interval [0, 100] for any array of Issues.
2. THE Core test suite SHALL include a property test asserting that for any array of Issues `A` and
   any additional Issue `i`, `hauntingScore([...A, i]) <= hauntingScore(A)` (score never increases
   when an issue is added).

---

### Requirement 20: Unit Tests — Per-Rule Fixture Tests

**User Story:** As a developer, I want unit tests for each Rule using fixture HTML, so that I can
verify each rule's detection logic in isolation.

#### Acceptance Criteria

1. THE Core test suite SHALL include unit tests for Rule `contrast-text` using fixture HTML
   elements with known foreground and background colors.
2. THE Core test suite SHALL include unit tests for Rule `alt-text` using fixture `<img>` elements
   with absent, empty, and present `alt` attributes.
3. THE Core test suite SHALL include unit tests for Rule `tap-target` using fixture interactive
   elements with known bounding rects below and above 24 × 24 CSS px.
4. THE Core test suite SHALL include unit tests for Rule `label` using fixture `<input>` and
   `<button>` elements with and without accessible names.

---

### Requirement 21: End-to-End Test

**User Story:** As a developer, I want a Playwright E2E test that opens a demo page, activates
Haunt Mode, verifies ghosts appear, then uses Exorcise to reduce the issue count, so that the
critical user path is automatically verified.

#### Acceptance Criteria

1. THE E2E test suite SHALL use Playwright to open one of the Demo_Pages in the Web_App.
2. WHEN the E2E test activates Haunt_Mode, THE test SHALL assert that at least one Ghost element is
   present in the page's Shadow_DOM.
3. WHEN the E2E test triggers Exorcise on the first contrast Issue and confirms the Fix, THE test
   SHALL assert that the Ghost count has decreased by at least one.
4. THE E2E test SHALL run to completion without requiring a human to interact with the browser.

---

### Requirement 22: Self-Audit Compliance

**User Story:** As a team, I want Wraith's own UI to pass its own audit, so that the tool
demonstrates its stated AA standard by example.

#### Acceptance Criteria

1. THE Web_App SHALL pass the Wraith Auditor's checks for Rule `contrast-text` and
   `contrast-non-text` on every page of the Web_App itself, excluding Demo_Pages and Paste HTML
   content, which are intentionally inaccessible.
2. THE Web_App SHALL provide full keyboard navigation to every interactive control, with visible
   focus indicators that meet AA contrast requirements.
3. THE Web_App SHALL include ARIA landmark roles and labels on all major sections and interactive
   components.
4. THE Web_App SHALL honor `prefers-reduced-motion` for every animation it uses.
5. THE Extension popup SHALL pass the same AA-level audit criteria as the Web_App.

---

### Requirement 23: Repository Configuration

**User Story:** As a contributor, I want the repository to follow standard conventions and include
all required documentation, so that judges and collaborators can quickly understand and run the
project.

#### Acceptance Criteria

1. THE Repo SHALL be a pnpm monorepo with TypeScript strict mode enabled in all packages and apps.
2. THE Repo SHALL contain a root `README.md` explaining what Wraith is, a placeholder for the live
   demo URL, setup/run/test/deploy steps, a Mermaid architecture diagram, and a "Third-party code"
   section disclosing axe-core (MPL-2.0), fast-check, Vite, Playwright, and fonts.
3. THE Repo SHALL contain a `KIRO_LOG.md` recording where specs, hooks, steering, and property
   tests helped or caught bugs.
4. THE Repo SHALL keep the MIT `LICENSE` file unchanged.
5. THE Repo SHALL NOT list the `.kiro` folder in `.gitignore`.
6. THE Repo SHALL NOT include any real company names, logos, trademarks, or copyrighted assets.
