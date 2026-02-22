# 📋 02 — Scene Configuration

> Define the data-driven scene array and transition presets that power the entire storybook.

---

## The StoryPage Interface

Every scene in the storybook is described by a single typed object:

```typescript
type TransitionPreset = "hero" | "slideLeft" | "slideRight" | "split" | "scale" | "rise" | "fade"

interface StoryPage {
  id: string                    // unique slug, used for routing/switch
  label: string                 // short name shown in page indicator
  color: string                 // hex color — drives all accents for this scene
  mascotPosition: "left" | "right" | "center" | "top-left" | "top-right" | "hidden"
  mascotQuote: string           // what the guide character says (empty = no bubble)
  transition: TransitionPreset  // how this scene enters/exits
  bgGlow?: string               // radial-gradient CSS for ambient background color
}
```

---

## The PAGES Array

This is the **single source of truth** for the storybook. The order defines the scroll sequence.

```typescript
const PAGES: StoryPage[] = [
  {
    id: "hero",
    label: "Welcome",
    color: "#00D4AA",
    mascotPosition: "hidden",        // mascot hidden on hero — let the content breathe
    mascotQuote: "",
    transition: "hero",
    bgGlow: "radial-gradient(ellipse 70% 50% at 50% 40%, rgba(0,212,170,0.06) 0%, transparent 70%)",
  },
  {
    id: "problem",
    label: "Problem",
    color: "#FF4D6D",                // red = urgency, danger
    mascotPosition: "left",
    mascotQuote: "Here's what's broken...",
    transition: "slideLeft",
    bgGlow: "radial-gradient(ellipse 60% 50% at 30% 50%, rgba(255,77,109,0.05) 0%, transparent 70%)",
  },
  {
    id: "solution",
    label: "Solution",
    color: "#00D4AA",                // back to green = relief, answer
    mascotPosition: "right",
    mascotQuote: "And here's how we fix it.",
    transition: "slideRight",
    bgGlow: "radial-gradient(ellipse 60% 50% at 70% 50%, rgba(0,212,170,0.05) 0%, transparent 70%)",
  },
  // ... more product pages, stats, etc.
  {
    id: "faq",
    label: "FAQ",
    color: "#00B4D8",
    mascotPosition: "hidden",        // hidden for embed pages
    mascotQuote: "Questions? I have answers.",
    transition: "fade",
    bgGlow: "radial-gradient(ellipse 60% 50% at 30% 50%, rgba(0,180,216,0.04) 0%, transparent 70%)",
  },
  {
    id: "footer",
    label: "Connect",
    color: "#00D4AA",
    mascotPosition: "hidden",
    mascotQuote: "",
    transition: "fade",
    bgGlow: "none",
  },
]
```

### Color Strategy

| Scene Type | Color | Why |
|------------|-------|-----|
| Hero / positive | Teal/green | Trust, growth, action |
| Problem / pain | Red/coral | Urgency, danger |
| Product features | Brand-specific | Each product gets its own identity |
| Process / how-it-works | Purple | Technical sophistication |
| Info / FAQ | Cyan/blue | Calm, informational |

### Background Glow Formula

```
radial-gradient(
  ellipse [width]% [height]% at [x]% [y]%,
  rgba(R, G, B, 0.04-0.06) 0%,
  transparent 70%
)
```

- Position the glow roughly where the content's visual weight is
- Opacity between 0.04 and 0.06 — barely visible but creates atmosphere
- Use `none` for the footer or minimal pages

---

## Transition Variants

Each preset is a Framer Motion `Variants` object with `initial`, `animate`, and `exit` states:

```typescript
const TRANSITION_VARIANTS: Record<TransitionPreset, Variants> = {
  hero: {
    initial: { opacity: 0, scale: 0.92, filter: "blur(12px)" },
    animate: { opacity: 1, scale: 1, filter: "blur(0px)", transition: { duration: 0.8, ease: "easeOut" } },
    exit:    { opacity: 0, scale: 1.04, filter: "blur(6px)", transition: { duration: 0.4 } },
  },
  slideLeft: {
    initial: { opacity: 0, x: -100 },
    animate: { opacity: 1, x: 0, transition: { duration: 0.6, ease: [0.22, 1, 0.36, 1] } },
    exit:    { opacity: 0, x: 100, transition: { duration: 0.35 } },
  },
  slideRight: {
    initial: { opacity: 0, x: 100 },
    animate: { opacity: 1, x: 0, transition: { duration: 0.6, ease: [0.22, 1, 0.36, 1] } },
    exit:    { opacity: 0, x: -100, transition: { duration: 0.35 } },
  },
  split: {
    initial: { opacity: 0, y: 40, scale: 0.96 },
    animate: { opacity: 1, y: 0, scale: 1, transition: { duration: 0.65, ease: [0.22, 1, 0.36, 1] } },
    exit:    { opacity: 0, y: -30, transition: { duration: 0.35 } },
  },
  scale: {
    initial: { opacity: 0, scale: 0.85 },
    animate: { opacity: 1, scale: 1, transition: { duration: 0.6, ease: "easeOut" } },
    exit:    { opacity: 0, scale: 1.08, transition: { duration: 0.35 } },
  },
  rise: {
    initial: { opacity: 0, y: 60 },
    animate: { opacity: 1, y: 0, transition: { duration: 0.55, ease: [0.22, 1, 0.36, 1] } },
    exit:    { opacity: 0, y: -50, transition: { duration: 0.35 } },
  },
  fade: {
    initial: { opacity: 0 },
    animate: { opacity: 1, transition: { duration: 0.6 } },
    exit:    { opacity: 0, transition: { duration: 0.3 } },
  },
}
```

### Choosing Transitions

| Content Direction | Transition | Rationale |
|------------------|------------|-----------|
| Opening scene | `hero` | Zoom + blur-in feels cinematic |
| Problem → Solution | `slideLeft` → `slideRight` | Oppositional movement |
| Product deep-dives | `split` | Feels like drilling into detail |
| Process / steps | `scale` | Growing understanding |
| Stats / impact | `rise` | Rising upward = positive |
| Embed pages (FAQ, etc.) | `fade` | Neutral, doesn't compete with content |

### Custom Easing

`[0.22, 1, 0.36, 1]` is a fast-start, gentle-landing cubic-bezier. It's the signature ease for this pattern — feels swift but not abrupt.

---

## How It's Used

In the main orchestrator, the active page's transition variant is applied:

```tsx
const currentPage = PAGES[activePage]
const transitionVariants = TRANSITION_VARIANTS[currentPage.transition]

<AnimatePresence mode="wait">
  <motion.div
    key={activePage}
    variants={transitionVariants}
    initial="initial"
    animate="animate"
    exit="exit"
    className="absolute inset-0"
  >
    {renderPageContent(activePage)}
  </motion.div>
</AnimatePresence>
```

The `key={activePage}` is critical — it tells AnimatePresence a new component is mounting, triggering the exit animation on the old one and the enter animation on the new one.

---

## Adding a New Page

1. Add an object to the `PAGES[]` array in the desired position
2. Create the scene component function
3. Add a `case` to the `renderPageContent` switch
4. Choose a color, transition, and mascot position

That's it. The navigation engine, indicator, mascot, and background glow all adapt automatically.

---

## Next Step

→ [03 — Intro Sequence](03-intro-sequence.md)
