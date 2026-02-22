# 🎨 01 — Theme & CSS Foundation

> Establish the dark theme, glassmorphism utilities, gradient effects, and keyframe animations that every component builds on.

---

## CSS Custom Properties

Define all theme tokens in `globals.css` so Tailwind + custom classes share the same source of truth.

```css
@layer base {
  :root {
    /* Core background — deep navy, NOT pure black */
    --color-bg-dark: #0A0F1C;
    --color-bg-card: #111827;
    --color-bg-elevated: #1F2937;

    /* Text hierarchy */
    --color-text-primary: #F9FAFB;
    --color-text-secondary: #9CA3AF;
    --color-text-muted: #6B7280;

    /* Brand colors — CUSTOMIZE THESE for each project */
    --color-primary: #00D4AA;       /* teal/green — main CTA, success */
    --color-primary-light: #00F5C4;
    --color-primary-dark: #00B894;
    --color-accent: #6C5CE7;        /* purple — secondary actions */
    --color-accent-light: #A29BFE;
    --color-info: #00B4D8;          /* cyan — informational */
    --color-danger: #FF4D6D;        /* red — problems, urgency */

    /* Borders */
    --color-border: rgba(0, 212, 170, 0.15);

    /* shadcn/ui HSL variables */
    --background: 220 20% 7%;
    --foreground: 210 40% 98%;
    --card: 220 20% 10%;
    --muted: 220 20% 18%;
    --border: 220 20% 18%;
    --ring: 174 100% 42%;
    --radius: 0.75rem;
    /* ... (rest of shadcn HSL palette) */
  }
}
```

### Why Not Pure Black?
`#0A0F1C` (dark navy) gives depth. Pure `#000000` looks flat and makes glows appear harsh. The slight blue tint creates a sense of space.

---

## Glassmorphism Components

```css
@layer components {
  /* Base glass card */
  .glass-card {
    background: rgba(17, 24, 39, 0.8);
    backdrop-filter: blur(16px);
    -webkit-backdrop-filter: blur(16px);
    border: 1px solid rgba(0, 212, 170, 0.1);
    box-shadow:
      0 4px 24px rgba(0, 0, 0, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.03);
  }

  /* Glass card with hover lift */
  .glass-card-hover {
    @apply glass-card;
    transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  }
  .glass-card-hover:hover {
    border-color: rgba(0, 212, 170, 0.3);
    box-shadow:
      0 8px 32px rgba(0, 0, 0, 0.3),
      0 0 0 1px rgba(0, 212, 170, 0.1),
      inset 0 1px 0 rgba(255, 255, 255, 0.05);
    transform: translateY(-2px);
  }
}
```

### Adapt border color to the scene
In React, use inline styles for the border color, pulling from the scene's `color` property:

```tsx
style={{ borderColor: `${sceneColor}15` }}  // 15 = ~9% opacity hex
```

---

## Gradient Text

```css
.gradient-text {
  background: linear-gradient(135deg, #00D4AA 0%, #00B4D8 50%, #6C5CE7 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
```

Use on hero headlines or key phrases. Don't overuse — one instance per viewport max.

---

## Glow Button

```css
.btn-glow {
  @apply relative overflow-hidden;
  background: linear-gradient(to right, var(--color-primary), var(--color-primary-dark));
  box-shadow: 0 4px 20px rgba(0, 212, 170, 0.25);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
.btn-glow:hover {
  box-shadow: 0 6px 30px rgba(0, 212, 170, 0.4);
  transform: translateY(-1px);
}
/* Shine sweep on hover */
.btn-glow::before {
  content: '';
  position: absolute;
  top: 0; left: -100%;
  width: 100%; height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.2), transparent);
  transition: left 0.6s ease;
}
.btn-glow:hover::before { left: 100%; }
```

---

## Keyframe Animations

```css
/* Typewriter reveal — CSS-only, no React re-renders */
@keyframes typewriter-clip {
  from { clip-path: inset(0 100% 0 0); }
  to   { clip-path: inset(0 0 0 0); }
}

/* Cursor blink */
@keyframes typewriter-blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

/* Hide cursor after typing finishes */
@keyframes typewriter-cursor-hide {
  to { opacity: 0; }
}

/* Subtle float */
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-6px); }
}

/* Pulsing glow */
@keyframes pulse-glow {
  0%, 100% { box-shadow: 0 0 15px rgba(0, 212, 170, 0.3); }
  50% { box-shadow: 0 0 25px rgba(0, 212, 170, 0.5); }
}

/* Animated gradient background (use sparingly) */
@keyframes gradient-shift {
  0%, 100% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
}

/* Shimmer loading */
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

---

## Scrollbar Styling

```css
::-webkit-scrollbar { width: 8px; height: 8px; }
::-webkit-scrollbar-track { background: #0A0F1C; }
::-webkit-scrollbar-thumb { background: #1F2937; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: var(--color-primary); }
```

---

## Selection Color

```css
::selection {
  background: rgba(0, 212, 170, 0.3);
  color: white;
}
```

---

## Focus States

```css
:focus-visible {
  outline: none;
  box-shadow: 0 0 0 2px var(--color-primary), 0 0 0 4px var(--color-bg-dark);
}
```

---

## Premium Effects (Optional)

Use these sparingly for special sections:

```css
/* Cyber grid overlay */
.cyber-grid::after {
  content: '';
  position: absolute; inset: 0;
  background-image:
    linear-gradient(rgba(0, 212, 170, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 212, 170, 0.03) 1px, transparent 1px);
  background-size: 40px 40px;
  pointer-events: none;
  mask-image: radial-gradient(ellipse 60% 60% at 50% 50%, black 20%, transparent 70%);
}

/* Noise texture for depth */
.noise-overlay::after {
  content: '';
  position: absolute; inset: 0;
  background-image: url("data:image/svg+xml,..."); /* fractalNoise SVG */
  opacity: 0.03;
  pointer-events: none;
}

/* Holographic card shine — hover only */
.holographic-card::before {
  background: linear-gradient(105deg, transparent 20%, rgba(YOUR_COLOR, 0.04) 30%, ...);
  background-size: 200% 100%;
  transition: background-position 0.8s ease;
}
.holographic-card:hover::before { background-position: -200% 0; }
```

---

## Key Rules

1. **All opacity values ≤ 10%** for backgrounds/borders. Glassmorphism breaks if too opaque.
2. **No continuous JS-driven animations** for visual effects. Use CSS `@keyframes` or Framer Motion's declarative `animate`.
3. **Match border/glow colors** to the scene's brand color via inline styles.
4. **Test on a dark monitor** — what looks good on a bright display may be invisible on a dim one.

---

## Next Step

→ [02 — Scene Config](02-scene-config.md)
