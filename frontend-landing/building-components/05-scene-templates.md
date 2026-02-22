# 🎬 05 — Scene Templates

> Reusable scene archetypes. Mix, match, and customize for any product story.

---

## Scene Structure Rules

Every scene component follows these constraints:

1. **Full viewport** — `h-full w-full` fills the parent (which is `absolute inset-0`)
2. **Centered content** — flex/grid centered vertically and horizontally
3. **Max width** — `max-w-5xl` or `max-w-6xl` prevents ultra-wide stretch
4. **Stagger animations** — each element uses `initial` + `animate` with incremental delays
5. **Mobile overflow** — `overflow-y-auto md:overflow-visible` allows scrolling when content overflows on small screens
6. **Bottom padding** — `pb-32 md:pb-48` keeps content above the mascot zone (desktop)

```tsx
// Standard scene wrapper
<div className="flex items-start md:items-center justify-center h-full
                px-4 md:px-6 pt-20 md:pt-0 pb-32 md:pb-48
                overflow-y-auto md:overflow-visible">
  <div className="max-w-6xl w-full">
    {/* Scene content */}
  </div>
</div>
```

---

## Template 1: Hero

**Purpose:** Opening page. Big headline, subtext, CTAs, trust badges.

**Layout:** Single column, centered text.

```
┌─────────────────────────────────┐
│           [Logo/Icon]           │
│           [Badge]               │
│     Big Headline Text           │
│     Gradient Subline            │
│     Description paragraph       │
│     [Primary CTA] [Secondary]   │
│     🔒 Trust  ⚡ Speed  🛡 Cert │
│     ↓ Scroll to begin           │
└─────────────────────────────────┘
```

**Key patterns:**
- Logo with spring scale animation (`stiffness: 120, damping: 14`)
- Badge slides in (`opacity: 0, y: 10` → animate)
- Headline with 0.4s delay, title → gradient span on second line
- CTAs at 1.0s delay
- Trust badges at 1.3s delay
- "Scroll to begin" + bouncing chevron at 2.0s delay, positioned `absolute bottom-4`

```tsx
function HeroPage({ onJump }: { onJump: (idx: number) => void }) {
  return (
    <div className="relative flex flex-col items-center justify-center text-center px-6 h-full">
      <motion.div
        initial={{ opacity: 0, scale: 0.8 }}
        animate={{ opacity: 1, scale: 1 }}
        transition={{ delay: 0.1, type: "spring", stiffness: 120, damping: 14 }}
      >
        <YourLogo size={56} />
      </motion.div>

      <motion.h1
        initial={{ opacity: 0, y: 30 }}
        animate={{ opacity: 1, y: 0 }}
        transition={{ delay: 0.4, duration: 0.8 }}
        className="text-3xl sm:text-5xl md:text-7xl font-bold mt-6 leading-tight"
      >
        Your Headline Here
        <span className="gradient-text block mt-2">Gradient Subline</span>
      </motion.h1>

      {/* ... CTAs, trust badges, scroll indicator */}
    </div>
  )
}
```

---

## Template 2: Problem / Pain Point

**Purpose:** Create urgency. Show the user's pain in data.

**Layout:** 2-column — text left, live data visualization right.

```
┌──────────────────┬───────────────┐
│ [Badge: Problem]  │  ┌──────────┐ │
│ Headline with     │  │ Live     │ │
│ red accent        │  │ Data     │ │
│ Description       │  │ Feed     │ │
│                   │  │ Ticker   │ │
│ ┌────┐ ┌────┐   │  │          │ │
│ │3.4K│ │4.2h│   │  │          │ │
│ │stat│ │stat│   │  └──────────┘ │
│ ┌────┐ ┌────┐   │               │
│ │73% │ │89% │   │               │
│ │stat│ │stat│   │               │
└──────────────────┴───────────────┘
```

**Key patterns:**
- Color scheme: red/coral for danger
- 2×2 stat grid with staggered entry (`delay: 0.12 * i + 0.3`)
- Right panel: auto-scrolling data feed (CSS `motion.div` with `animate y`)
- Fade edges on the ticker (gradient overlays top/bottom)

**Stat pill pattern:**
```tsx
<motion.div
  initial={{ opacity: 0, y: 15 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ delay: 0.12 * i + 0.3, duration: 0.5 }}
  className="p-4 rounded-xl bg-red-500/[0.04] border border-red-500/10"
>
  <div className="text-2xl font-bold text-red-400">{stat}</div>
  <div className="text-[11px] uppercase tracking-wider text-red-400/50">{label}</div>
  <p className="text-sm text-gray-400 mt-2">{description}</p>
</motion.div>
```

---

## Template 3: Solution / Value Prop

**Purpose:** Relief. Show how your product fixes the pain.

**Layout:** 2-column — feature list left, before/after panel right.

```
┌──────────────────┬────────────────┐
│ [Badge: Solution] │  Workflow       │
│ Headline with     │  Transform     │
│ green accent      │  ┌──┬──┬──┐    │
│ Description       │  │ B │→│ A │    │
│                   │  │ B │→│ A │    │
│ ✓ Feature 1      │  │ B │→│ A │    │
│ ✓ Feature 2      │  │ B │→│ A │    │
│ ✓ Feature 3      │  └──┴──┴──┘    │
│ ✓ Feature 4      │  ~12hrs → ~4min │
└──────────────────┴────────────────┘
```

**Before/After pattern:**
```tsx
{workflow.map((step, i) => (
  <div className="flex items-center gap-4">
    <span className="text-[10px] font-bold uppercase tracking-wider">{step.label}</span>
    <div className="flex-1 px-3 py-2 rounded-lg bg-red-500/[0.06] border border-red-500/10">
      <span className="text-xs text-red-400/60 line-through">{step.from}</span>
    </div>
    <motion.div animate={{ x: [0, 4, 0] }} transition={{ duration: 1.5, repeat: Infinity }}>
      <ArrowRight className="w-4 h-4 text-green-400/50" />
    </motion.div>
    <div className="flex-1 px-3 py-2 rounded-lg bg-green-500/[0.06] border border-green-400/15">
      <span className="text-xs text-green-400/90 font-medium">{step.to}</span>
    </div>
  </div>
))}
```

---

## Template 4: Product Feature

**Purpose:** Showcase an individual product/feature.

**Layout:** 2-column — description + features left, mock terminal/dashboard right.

```
┌──────────────────┬────────────────┐
│ VISION I          │  $ command     │
│ [Icon]            │  ────────────  │
│ Product Name      │  label: 847   │
│ Tagline           │  label: 2,341 │
│ Description       │  label: 23    │
│                   │  label: 1,205 │
│ ✓ Feature 1      │               │
│ ✓ Feature 2      │  [Progress ══] │
│ [Learn More →]    │               │
└──────────────────┴────────────────┘
```

**Mock terminal pattern:**
```tsx
<div className="rounded-xl border overflow-hidden"
     style={{ borderColor: `${color}15`, background: `${color}03` }}>
  {/* Terminal header with dots */}
  <div className="flex items-center gap-2 px-4 py-2.5 border-b"
       style={{ borderColor: `${color}10`, background: `${color}05` }}>
    <div className="flex gap-1.5">
      <div className="w-2.5 h-2.5 rounded-full bg-white/10" />
      <div className="w-2.5 h-2.5 rounded-full bg-white/10" />
      <div className="w-2.5 h-2.5 rounded-full bg-white/10" />
    </div>
    <span className="text-[11px] font-mono text-white/30 ml-2">{command}</span>
  </div>
  {/* Data rows with stagger */}
  <div className="p-4 space-y-3">
    {lines.map((line, i) => (
      <motion.div key={i}
        initial={{ opacity: 0, x: 10 }}
        animate={{ opacity: 1, x: 0 }}
        transition={{ delay: 0.1 * i + 0.5 }}
        className="flex items-center justify-between"
      >
        <span className="text-xs text-gray-500 font-mono">{line.label}</span>
        <span className="text-sm font-mono font-medium"
              style={line.highlight ? { color } : {}}>{line.value}</span>
      </motion.div>
    ))}
  </div>
  {/* Animated progress bar */}
  <motion.div className="h-1.5 rounded-full" style={{ background: color }}
    initial={{ width: "0%" }}
    animate={{ width: "100%" }}
    transition={{ duration: 2, delay: 0.8, ease: "easeOut" }}
  />
</div>
```

---

## Template 5: Process / Steps

**Purpose:** Show a multi-step workflow.

**Layout:** 4-column card grid with connecting line.

```
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ 01     │→ │ 02     │→ │ 03     │→ │ 04     │
│ [Icon] │  │ [Icon] │  │ [Icon] │  │ [Icon] │
│ Title  │  │ Title  │  │ Title  │  │ Title  │
│ Desc   │  │ Desc   │  │ Desc   │  │ Desc   │
│ ────── │  │ ────── │  │ ────── │  │ ────── │
│ Metric │  │ Metric │  │ Metric │  │ Metric │
└────────┘  └────────┘  └────────┘  └────────┘
```

**Key patterns:**
- `grid-cols-2 md:grid-cols-4` (collapses to 2×2 on mobile)
- Connecting line: `absolute top-[3.5rem] left-[12.5%] right-[12.5%] h-px` with gradient
- Arrow circles between cards (`hidden md:flex`)
- Spring stagger: `delay: 0.12 * i`
- Bottom metrics with `border-t border-white/5`

---

## Template 6: Stats / Impact

**Purpose:** Social proof with animated numbers.

**Layout:** 2×2 grid, each card has ring chart + animated counter.

**Animated number pattern:**
```tsx
function AnimatedNumber({ value, suffix, duration = 1.5, delay = 0 }) {
  const [displayed, setDisplayed] = useState(0)
  const [started, setStarted] = useState(false)

  useEffect(() => {
    const t = setTimeout(() => setStarted(true), delay * 1000)
    return () => clearTimeout(t)
  }, [delay])

  useEffect(() => {
    if (!started) return
    const steps = 40
    const increment = value / steps
    let current = 0
    const interval = setInterval(() => {
      current += increment
      if (current >= value) { setDisplayed(value); clearInterval(interval) }
      else { setDisplayed(Math.floor(current)) }
    }, (duration * 1000) / steps)
    return () => clearInterval(interval)
  }, [started, value, duration])

  return <>{displayed.toLocaleString()}{suffix}</>
}
```

**SVG ring chart pattern:**
```tsx
function RingChart({ percent, color, delay = 0 }) {
  const r = 36, c = 2 * Math.PI * r
  return (
    <svg width="92" height="92" viewBox="0 0 92 92">
      <circle cx="46" cy="46" r={r} fill="none" stroke="rgba(255,255,255,0.05)" strokeWidth="5" />
      <motion.circle cx="46" cy="46" r={r} fill="none" stroke={color} strokeWidth="5"
        strokeLinecap="round" strokeDasharray={c}
        initial={{ strokeDashoffset: c }}
        animate={{ strokeDashoffset: c - (percent/100) * c }}
        transition={{ duration: 1.5, delay, ease: "easeOut" }}
        style={{ transform: "rotate(-90deg)", transformOrigin: "center" }}
      />
    </svg>
  )
}
```

---

## Template 7: Embed (FAQ, Signup, Footer)

**Purpose:** Wrap existing standalone section components inside the storybook.

**Layout:** Full-width scrollable container.

```tsx
function StoryEmbedPage({ children }: { children: React.ReactNode }) {
  return (
    <div className="h-full w-full flex flex-col">
      {/* Top spacer for navbar clearance */}
      <div className="h-16 shrink-0" />
      {/* Scrollable content area */}
      <div
        data-story-embed
        className="flex-1 overflow-y-auto overscroll-contain
                   scrollbar-thin scrollbar-thumb-white/10 pb-8"
      >
        {children}
      </div>
    </div>
  )
}
```

The `data-story-embed` attribute is detected by the scroll engine to know this page has its own internal scrolling.

---

## Stagger Timing Formula

For any list of N items:
```
delay = BASE_DELAY + (INDEX * STAGGER)
```

| Context | BASE_DELAY | STAGGER |
|---------|-----------|---------|
| Feature list | 0.3s | 0.12s |
| Stat cards | 0.0s | 0.12s |
| Data rows | 0.5s | 0.1s |
| Process steps | 0.0s | 0.12s |

This creates a reading rhythm — items appear one-by-one, fast enough not to bore.

---

## Next Step

→ [06 — Page Indicator](06-page-indicator.md)
