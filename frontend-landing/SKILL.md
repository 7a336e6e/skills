# 🎭 Immersive Storybook Landing Page — Frontend Skill

> **Build a cinematic, full-screen storybook-style landing page** with scroll-hijacked scene transitions, animated mascot/guide, glassmorphism UI, and dark-theme premium aesthetics. The user tells you what the product/page is about — you craft the experience.

---

## 📋 Skill Metadata

| Key | Value |
|-----|-------|
| **Difficulty** | Advanced |
| **Stack** | Next.js (App Router) · React 19 · TypeScript · Tailwind CSS · Framer Motion |
| **Category** | Frontend · Landing Page · Animation · UX |
| **Time** | 2–4 hours |
| **Prerequisites** | [Scaffolding Frontend](scaffolding-frontend/SKILL.md) |

---

## 🎯 What You Will Build

A **single-page storybook experience** where:

1. The entire viewport is hijacked — each "page" is a full-screen scene
2. Scroll/swipe/keyboard navigates between scenes with cinematic transitions
3. A **mascot or guide character** (positioned at screen edges) reacts to each scene with speech bubbles
4. An animated **intro sequence** plays once, then is skipped on revisit (sessionStorage)
5. A **vertical page indicator** (timeline dots) shows progress and allows direct navigation
6. Content uses **glassmorphism cards**, **gradient text**, **animated counters**, and **mock data visualizations**
7. Existing section components (FAQ, signup, footer) embed seamlessly as scrollable sub-pages
8. Mobile is fully responsive with adapted layouts, hidden mascot, and touch navigation

**This is NOT a template.** The user describes their product, brand, and pages — you use these patterns to create something unique and impressive.

---

## 🏗️ Architecture Overview

```
app/
  page.tsx                    ← Mounts the Journey component
  globals.css                 ← Theme variables, glass/glow utilities, keyframes
components/
  journey/
    journey.tsx               ← THE monolith: config + all scenes + orchestrator
    star-field.tsx            ← Ambient particle/star background (optional)
  landing/
    faq-section.tsx           ← Embeddable sections (scroll inside scene)
    signup-form-section.tsx
    footer-section.tsx
    interactive-demo-section.tsx
  shared/
    navbar.tsx
    logo.tsx
  ui/
    badge.tsx, button.tsx … ← shadcn/ui primitives
```

### Key Architectural Decisions

| Decision | Why |
|----------|-----|
| **Single monolith file** for the journey | All scenes share state (activePage, intro, transitions). Co-location avoids prop drilling and keeps the storybook cohesive. Break out only truly independent pieces (star-field, logo). |
| **Data-driven scene config** | A `PAGES[]` array defines every scene: id, label, color, mascot position, mascot quote, transition preset, background glow. Adding a page = adding an object. |
| **CSS-only typewriter** | Zero React re-renders. Uses `clip-path: inset()` with `steps()` animation. |
| **Framer Motion for orchestration** | `AnimatePresence` + `variants` for scene enter/exit. Spring physics for mascot. Motion values for counters and rings. |
| **sessionStorage intro skip** | Intro plays once per session. On revisit, content mounts instantly. |
| **Body scroll lock** | `document.body.style.overflow = "hidden"` on mount, restored on unmount. |

---

## 📖 Step-by-Step Implementation

### Step 1 — Scaffold the Project

> See [scaffolding-frontend/SKILL.md](scaffolding-frontend/SKILL.md) for full details.

```bash
npx create-next-app@latest my-landing --typescript --tailwind --app --eslint
cd my-landing
npx shadcn@latest init    # dark theme, zinc/slate base
npm install framer-motion lucide-react
```

### Step 2 — Establish the Dark Theme & CSS Utilities

> See [building-components/01-theme-and-css.md](building-components/01-theme-and-css.md)

Set up CSS custom properties, glassmorphism classes, gradient text, glow buttons, keyframe animations. This is the visual foundation everything else builds on.

### Step 3 — Define the Scene Configuration

> See [building-components/02-scene-config.md](building-components/02-scene-config.md)

Create the `PAGES[]` array and transition variants. Each page is a typed object:

```typescript
interface StoryPage {
  id: string
  label: string
  color: string               // brand color for this scene
  mascotPosition: "left" | "right" | "center" | "top-left" | "top-right" | "hidden"
  mascotQuote: string          // what the guide says on this page
  transition: TransitionPreset // "hero" | "slideLeft" | "slideRight" | "split" | "scale" | "rise" | "fade"
  bgGlow?: string              // radial-gradient for ambient scene color
}
```

### Step 4 — Build the Scroll/Navigation Engine

> See [managing-state/01-navigation-engine.md](managing-state/01-navigation-engine.md)

The orchestrator handles:
- **Wheel** (with debounce + content scroll detection)
- **Touch** (swipe threshold ≥ 50px)
- **Keyboard** (ArrowUp/Down, Space)
- **Direct jump** via page indicator dots or CTA buttons
- **Transition lock** (700ms cooldown prevents double-fires)
- **Scrollable sub-page detection** (embeds scroll internally before page-changing)

### Step 5 — Create the Intro Sequence

> See [building-components/03-intro-sequence.md](building-components/03-intro-sequence.md)

A branded intro that:
1. Converges a glowing particle into the center
2. Reveals the brand logo/icon with spring animation
3. Fades away, setting `introComplete = true`
4. Stores completion in `sessionStorage` — skips on revisit

### Step 6 — Build the Mascot/Guide Component

> See [building-components/04-mascot-guide.md](building-components/04-mascot-guide.md)

A fixed-position animated character that:
- Moves between predefined positions with spring physics
- Shows speech bubbles with CSS-only typewriter text
- Blinks periodically (scaleY animation on a timer)
- Tracks mouse cursor (pupil offset with `requestAnimationFrame`)
- Is hidden on mobile (content takes priority on small screens)
- Has a "mini" variant for embed pages (small pulsing dot with hover tooltip)

### Step 7 — Build Each Scene

> See [building-components/05-scene-templates.md](building-components/05-scene-templates.md)

Scene archetypes (mix and match):

| Template | Layout | Best For |
|----------|--------|----------|
| **Hero** | Centered, stacked | Opening page, big headline + CTA |
| **Problem** | 2-column: text + live ticker/data | Pain points, urgency |
| **Solution** | 2-column: text + before/after panel | Value proposition |
| **Product** | 2-column: features + mock terminal | Individual product pages |
| **Process** | 4-column step cards with connectors | How it works |
| **Stats** | 2×2 grid with ring charts + counters | Social proof, metrics |
| **Embed** | Scrollable container for existing components | FAQ, signup, footer |

### Step 8 — Add the Page Indicator

> See [building-components/06-page-indicator.md](building-components/06-page-indicator.md)

A vertical timeline of dots:
- Active dot is larger + colored + glowing
- Past dots are medium + muted
- Future dots are small + barely visible
- Labels appear on hover (desktop only)
- Dots are clickable for direct navigation

### Step 9 — Mobile Responsiveness

> See [building-components/07-mobile-responsive.md](building-components/07-mobile-responsive.md)

Critical mobile adaptations:
- Mascot: `hidden md:block` (too much screen real estate on mobile)
- Page indicator: compact dots, no labels
- Text: scale down (text-3xl → text-2xl, etc.)
- Grids: collapse to single/double column
- Content overflow: `overflow-y-auto md:overflow-visible` with scroll detection
- Touch: swipe navigation with edge-of-scroll detection

### Step 10 — Performance & Polish

> See [bundling-frontend/SKILL.md](bundling-frontend/SKILL.md) and [testing-frontend/SKILL.md](testing-frontend/SKILL.md)

- `"use client"` only on the journey component
- Memoize heavy computations with `useMemo`
- Use `useCallback` for event handlers passed as props
- CSS-only animations where possible (typewriter, blink, float)
- `passive: true` on touch/scroll listeners
- `will-change: transform` on animated elements (use sparingly)
- Test all transitions at 3× CPU throttle

---

## 🧩 Sub-Skills

| Skill | Description |
|-------|-------------|
| [Scaffolding Frontend](scaffolding-frontend/SKILL.md) | Project setup, dependencies, folder structure |
| [Building Components](building-components/SKILL.md) | All visual components: theme, scenes, mascot, indicator |
| [Managing State](managing-state/SKILL.md) | Navigation engine, scroll hijack, intro persistence |
| [Integrating API](integrating-api/SKILL.md) | Connecting forms, analytics, dynamic data |
| [Bundling Frontend](bundling-frontend/SKILL.md) | Build optimization, code splitting, deployment |
| [Testing Frontend](testing-frontend/SKILL.md) | Visual regression, interaction testing, accessibility |

---

## 🎨 Design Principles

### 1. Restraint Over Excess
- Maximum 3 brand colors + 1 neutral palette
- Animations serve comprehension, never distract
- Blur/glow at 4-8% opacity — barely there, deeply felt
- No continuous CPU-heavy animations (use CSS, not JS timers)

### 2. Dark Theme Done Right
- Background: deep navy/charcoal (#0A0F1C), NOT pure black
- Cards: semi-transparent with subtle borders, NOT opaque
- Glows: color-matched to the scene's brand color
- Text: white for headings, gray-400/500 for body, brand color for accents

### 3. Motion With Purpose
- Scene transitions match content direction (slide left for "problem → solution")
- Staggered delays (0.1–0.15s per item) create reading rhythm
- Spring physics feel organic; linear easing feels mechanical
- Exit animations are faster than enter (0.3s exit vs 0.6s enter)

### 4. Mobile First, Desktop Enhanced
- Content must be fully usable without the mascot, animations, or hover effects
- Touch targets ≥ 44px
- No horizontal scroll, ever
- Test on 375px width (iPhone SE) as minimum

---

## 📐 Pattern Library

### Glassmorphism Card
```css
.glass-card {
  background: rgba(17, 24, 39, 0.8);
  backdrop-filter: blur(16px);
  border: 1px solid rgba(YOUR_COLOR, 0.1);
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.2),
              inset 0 1px 0 rgba(255, 255, 255, 0.03);
}
```

### CSS-Only Typewriter
```css
@keyframes typewriter-clip {
  from { clip-path: inset(0 100% 0 0); }
  to   { clip-path: inset(0 0 0 0); }
}
/* Usage: animation: typewriter-clip ${charCount * 28}ms steps(${charCount}, end) forwards; */
```

### Scene Transition Variant (Framer Motion)
```typescript
const slideLeft: Variants = {
  initial: { opacity: 0, x: -100 },
  animate: { opacity: 1, x: 0, transition: { duration: 0.6, ease: [0.22, 1, 0.36, 1] } },
  exit:    { opacity: 0, x: 100, transition: { duration: 0.35 } },
}
```

### Animated SVG Ring Chart
```tsx
<motion.circle r={36} strokeDasharray={2*Math.PI*36}
  initial={{ strokeDashoffset: 2*Math.PI*36 }}
  animate={{ strokeDashoffset: circumference - (percent/100)*circumference }}
  transition={{ duration: 1.5, ease: "easeOut" }}
/>
```

### Spring-Animated Mascot
```tsx
<motion.div
  animate={{ left: pos.x, top: pos.y }}
  transition={{ type: "spring", stiffness: 50, damping: 16, mass: 1.2 }}
/>
```

---

## ⚠️ Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Hydration mismatch** on intro state | Initialize state to `"loading"`, decide in `useEffect` |
| **Double page transitions** from fast scrolling | Use `isTransitioning` ref with 700ms cooldown |
| **Content hidden behind mascot** on mobile | Hide mascot on mobile, add padding on desktop |
| **AnimatePresence doesn't animate** | Ensure `key` changes and component conditionally unmounts |
| **Scroll events fire on embedded content** | Detect scroll position of embed container before allowing page change |
| **CSS animations restart on re-render** | Use `key={text}` to force restart, keep animation in CSS not React state |
| **Performance jank on low-end devices** | Prefer CSS transforms over layout properties, use `will-change` sparingly |
| **Touch events conflict with native scroll** | Check embed scroll position in touchend before page-changing |

---

## 🚀 Getting Started

Tell the AI agent:

> "Build me a storybook landing page for [YOUR PRODUCT]. It's a [description]. The pages should cover: [list your sections]. Use [your brand colors]. The mascot/guide should be [describe or say 'skip the mascot']."

The agent will use these skills to create a complete, production-ready landing page.
