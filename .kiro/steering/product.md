# Wraith — Product Overview

Wraith is a haunted accessibility auditor built for the Kiroween 2026 hackathon (Costume Contest category). It audits web pages for WCAG 2.2 AA violations and renders failing elements as **ghosts**: each ghost's opacity, blur, and spectral glow encode how inaccessible that element is. The tagline is "the spooky effect IS the information."

## Core Concept

The severity of an accessibility violation maps directly to a visual ghost property:
- More inaccessible → more transparent, more blurred, brighter glow
- Ghost opacity = `1 - 0.85 * severity` (range [0.15, 1])
- Haunting Score = integer [0, 100]; higher is more accessible

## Three Modes

- **Haunt Mode** — runs the auditor and renders ghost overlays on every failing element
- **Possess Mode** — applies full-page vision-simulation filters (protanopia, deuteranopia, tritanopia, low-vision, grayscale)
- **Exorcise** — suggests and applies the nearest OKLCH-space color fix that passes contrast threshold

## Delivery

- `apps/web` — hosted Vite + Vercel demo web app (primary deliverable; judges can use it without login)
- `apps/extension` — Chrome MV3 extension (stretch goal)

## Five Audited Rules

| Rule | WCAG | What it checks |
|---|---|---|
| `contrast-text` | 1.4.3 | Text foreground/background contrast (4.5:1 normal, 3:1 large) |
| `contrast-non-text` | 1.4.11 | Form control border vs adjacent background (3:1) |
| `alt-text` | 1.1.1 | Missing or empty non-decorative `alt` on `<img>` |
| `tap-target` | 2.5.8 | Interactive elements smaller than 24×24 CSS px |
| `label` | 4.1.2 | Inputs and buttons without accessible names |
