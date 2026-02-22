# Bundling Frontend — Optimisation & Deployment

> Ship a 60 fps storybook experience that loads fast on 3G, scores
> 90+ on Lighthouse, and deploys in one command.

---

## Overview

An animation-heavy, full-screen landing page has specific bundling
concerns that differ from a typical content site:

| Concern                  | Why It Matters                                     |
| ------------------------ | -------------------------------------------------- |
| **JS bundle size**       | Framer Motion + React can push past 150 kB gzipped |
| **CSS payload**          | Hundreds of keyframes and utility classes           |
| **First paint speed**    | Intro animation must start within ~1.5 s            |
| **Hydration cost**       | Full `"use client"` tree hydrates on load           |
| **Asset loading**        | Images, fonts, icons must not block the intro       |

---

## 1 — `"use client"` Boundary Strategy

### The Rule

> Push the `"use client"` boundary **as low as possible**.

```
app/
  layout.tsx          ← Server Component (metadata, fonts, analytics scripts)
  page.tsx            ← Server Component (renders the client journey)
  
components/
  seer/
    seer-journey.tsx  ← "use client" — the single interactive root
```

Only the storybook orchestrator and its children need to be client
components. Everything above it (layout, metadata, font loading) stays
on the server.

### Anti-Pattern

```tsx
// ❌ Don't mark layout.tsx as "use client" just because a child needs it
"use client"; // app/layout.tsx — wastes server rendering
```

### Why This Matters

- Server Components **don't ship JS** to the client
- The layout renders static HTML (fonts, meta tags, body wrapper)
- Only the interactive journey ships as a client bundle

---

## 2 — Code Splitting & Dynamic Imports

### Split Heavy Scenes

If certain scenes use large libraries (e.g., a chart library, 3D
renderer), lazy-load them:

```tsx
import dynamic from "next/dynamic";

const StatsScene = dynamic(() => import("./scenes/stats-scene"), {
  loading: () => <SceneSkeleton />,
  ssr: false, // No SSR for animation-heavy components
});

const DemoScene = dynamic(() => import("./scenes/demo-scene"), {
  loading: () => <SceneSkeleton />,
  ssr: false,
});
```

### When to Split vs Inline

| Scenario                          | Recommendation          |
| --------------------------------- | ----------------------- |
| Scene uses only Tailwind + motion | Keep inline (no split)  |
| Scene imports a chart / 3D lib    | `dynamic()` import      |
| Scene has a heavy embed (iframe)  | `dynamic()` + `ssr: false` |
| Mascot with pupil tracking only   | Keep inline             |

### Framer Motion Tree Shaking

Import only what you use — framer-motion supports deep imports:

```tsx
// ✅ Good — tree-shakeable
import { motion, AnimatePresence } from "framer-motion";

// ❌ Bad — pulls entire library (older pattern)
import * as Motion from "framer-motion";
```

As of framer-motion v11+, the main entry point is already
tree-shakeable, but avoid importing `*`.

---

## 3 — CSS Optimisation

### Tailwind Purging

Tailwind's JIT compiler only includes classes you actually use.
Ensure `content` paths cover all component files:

```ts
// tailwind.config.ts
const config: Config = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  // …
};
```

### CSS-Only Animations Over JS Animations

Prefer CSS `@keyframes` for anything that runs **continuously** or on
**non-interactive elements** (backgrounds, ambient effects):

| Animation Type        | CSS or JS?       | Reason                          |
| --------------------- | ---------------- | ------------------------------- |
| Typewriter text        | CSS `clip-path`  | Zero re-renders                 |
| Floating particles     | CSS `@keyframes` | Offloaded to compositor         |
| Pulse / glow loops     | CSS `@keyframes` | No JS overhead                  |
| Scene transitions      | Framer Motion    | Needs `AnimatePresence` logic   |
| Scroll-linked effects  | Framer Motion    | Needs `useScroll` / `useMotionValue` |
| Mascot eye tracking    | JS (ref-based)   | Needs pointer position          |

### `will-change` for Composited Layers

Apply `will-change` to elements that animate `transform` or `opacity`:

```css
.scene-container {
  will-change: transform, opacity;
}

.mascot-eye {
  will-change: transform;
}
```

> ⚠️ Don't apply `will-change` to everything — it consumes GPU memory.
> Only use it on elements that actually animate.

### Reducing Paint with `contain`

```css
.scene-page {
  contain: layout style paint;
}
```

This tells the browser that the scene is self-contained, reducing
paint calculations when other parts of the DOM change.

---

## 4 — Font Loading Strategy

### Use `next/font` for Zero-FOUT Loading

```tsx
// app/layout.tsx
import { Inter, JetBrains_Mono } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-sans",
});

const mono = JetBrains_Mono({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-mono",
});

export default function RootLayout({ children }) {
  return (
    <html className={`${inter.variable} ${mono.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

- `next/font` **self-hosts** fonts (no external requests)
- `display: "swap"` shows fallback text immediately
- CSS variables let Tailwind reference them:

```ts
// tailwind.config.ts
fontFamily: {
  sans: ["var(--font-sans)", ...fontFamily.sans],
  mono: ["var(--font-mono)", ...fontFamily.mono],
},
```

---

## 5 — Image & Asset Optimisation

### Icons: Lucide React (Tree-Shakeable SVGs)

```tsx
// ✅ Import individual icons
import { Shield, Brain, Eye } from "lucide-react";

// ❌ Don't import the whole set
import * as Icons from "lucide-react";
```

Each icon is ~200 bytes gzipped. Importing only what you need keeps
the bundle small.

### Background Images / Gradients

Prefer CSS gradients over raster images for backgrounds:

```css
/* ✅ Zero network requests, infinitely scalable */
.hero-bg {
  background: radial-gradient(
    ellipse at 30% 50%,
    rgba(0, 212, 170, 0.08) 0%,
    transparent 60%
  );
}

/* ❌ Adds a network request and is fixed resolution */
.hero-bg {
  background-image: url("/hero-bg.png");
}
```

### If You Must Use Images

```tsx
import Image from "next/image";

<Image
  src="/product-preview.png"
  alt="Product dashboard"
  width={800}
  height={500}
  priority={false} // Only priority for above-the-fold
  placeholder="blur"
  blurDataURL="data:image/png;base64,…" // Tiny placeholder
/>
```

---

## 6 — Build Configuration

### next.config.mjs

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,

  // Enable gzip + brotli compression
  compress: true,

  // Optimise package imports
  experimental: {
    optimizePackageImports: [
      "lucide-react",
      "framer-motion",
    ],
  },

  // Security headers
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        ],
      },
    ];
  },
};

export default nextConfig;
```

### TypeScript Strict Mode

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

This catches unused imports and dead code at compile time, preventing
bloat from accumulating.

---

## 7 — Performance Budget

Set targets and measure against them:

| Metric                      | Target         | Tool                    |
| --------------------------- | -------------- | ----------------------- |
| **First Contentful Paint**  | < 1.5 s        | Lighthouse              |
| **Largest Contentful Paint**| < 2.5 s        | Lighthouse              |
| **Total Blocking Time**     | < 200 ms       | Lighthouse              |
| **Cumulative Layout Shift** | < 0.1          | Lighthouse              |
| **JS Bundle (gzipped)**     | < 180 kB       | `next build` output     |
| **CSS (gzipped)**           | < 30 kB        | `next build` output     |
| **Time to Interactive**     | < 3.0 s (3G)   | WebPageTest             |

### Measuring Bundle Size

```bash
# Build and see chunk breakdown
npx next build

# Detailed bundle analysis
ANALYZE=true npx next build
# Requires: npm install @next/bundle-analyzer
```

### next/bundle-analyzer Setup

```js
// next.config.mjs
import withBundleAnalyzer from "@next/bundle-analyzer";

const config = withBundleAnalyzer({
  enabled: process.env.ANALYZE === "true",
})({
  // … your config
});

export default config;
```

---

## 8 — Deployment

### Vercel (Recommended for Next.js)

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy (auto-detects Next.js)
vercel

# Production deploy
vercel --prod
```

Vercel provides:
- Automatic edge caching for static pages
- ISR support out of the box
- Preview deployments for every PR
- Analytics dashboard

### Static Export (Any Host)

If you don't need server-side features:

```js
// next.config.mjs
const nextConfig = {
  output: "export",
  // Images need unoptimized for static export
  images: { unoptimized: true },
};
```

```bash
npx next build
# Output in `out/` — deploy to any static host
```

### Docker

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

Requires `output: "standalone"` in next.config.mjs.

---

## 9 — Pre-Deploy Checklist

- [ ] `npm run build` completes without errors
- [ ] Bundle size is within performance budget
- [ ] Lighthouse score ≥ 90 (Performance, Accessibility)
- [ ] Intro animation starts within 1.5 s on throttled 3G
- [ ] All `console.log` debug statements removed
- [ ] Environment variables set in deployment platform
- [ ] Analytics scripts load with `afterInteractive` strategy
- [ ] Security headers configured
- [ ] OG meta tags and favicon set in layout metadata
- [ ] `robots.txt` and `sitemap.xml` configured
- [ ] 404 page exists and matches the theme

---

## Quick Reference: Build Commands

```bash
# Development
npm run dev           # Start dev server (Turbopack)

# Production build
npm run build         # Build for production
npm run start         # Start production server locally

# Analysis
ANALYZE=true npm run build   # Bundle size analysis

# Linting
npm run lint          # ESLint check
npx tsc --noEmit      # Type check without build
```

---

**Next:** [Testing Frontend →](../testing-frontend/SKILL.md)
