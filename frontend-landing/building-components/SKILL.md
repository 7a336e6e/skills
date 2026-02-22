# 🧱 Building Components

> All visual components for the immersive storybook landing page.

---

## Sub-Skills

| # | File | What It Covers |
|---|------|----------------|
| 01 | [Theme & CSS](01-theme-and-css.md) | CSS custom properties, glassmorphism, gradients, glow effects, keyframes |
| 02 | [Scene Config](02-scene-config.md) | The PAGES[] data array, transition variants, TypeScript interfaces |
| 03 | [Intro Sequence](03-intro-sequence.md) | Branded loading animation with sessionStorage persistence |
| 04 | [Mascot / Guide](04-mascot-guide.md) | Animated character with speech bubbles, pupil tracking, blink, typewriter |
| 05 | [Scene Templates](05-scene-templates.md) | Hero, Problem, Solution, Product, Process, Stats, Embed archetypes |
| 06 | [Page Indicator](06-page-indicator.md) | Vertical dot timeline with labels and direct navigation |
| 07 | [Mobile Responsive](07-mobile-responsive.md) | Breakpoint strategy, touch targets, hidden elements, overflow handling |

---

## Design Token Strategy

All colors flow from the scene config. Each page has a `color` property that drives:

- Background glow (`radial-gradient` with the page color at 4-6% opacity)
- Mascot speech bubble border and text
- Badge and accent colors in content
- Page indicator active dot

This means you define colors **once** in the `PAGES[]` config, and every component adapts.

---

## Component Colocation

The journey is a **single file** containing:
- Type definitions
- Scene config array
- Transition variants
- Intro sequence component
- Mascot component
- All scene components
- Page indicator
- Main orchestrator (default export)

This is intentional. These components are tightly coupled to the storybook flow and share state. Extracting them into separate files would create prop-drilling or require a context provider — unnecessary complexity for a single-page experience.

**Extract only:**
- Background effects (star-field, particles) — purely visual, no shared state
- Logo component — reused across the site
- Embeddable sections (FAQ, signup) — exist independently of the journey

---

## Next Step

Start with → [01 — Theme & CSS](01-theme-and-css.md)
