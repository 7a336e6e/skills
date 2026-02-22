# ✨ 03 — Intro Sequence

> A branded animation that plays once per session, establishing mood before the content loads.

---

## Purpose

The intro serves three functions:
1. **Brand imprint** — the user sees your logo/icon before anything else
2. **Loading mask** — hides the initial render while React hydrates
3. **Tone setting** — communicates "this is a premium, crafted experience"

---

## Architecture

```
State machine: "loading" → "playing" → "done"
                   ↓                      ↓
          (server match)        (sessionStorage check)
```

### Hydration Safety

The intro state must start as `"loading"` (not `"playing"` or `"done"`) to avoid hydration mismatches. The decision happens in `useEffect`:

```typescript
const [introState, setIntroState] = useState<"loading" | "playing" | "done">("loading")
const introComplete = introState === "done"

useEffect(() => {
  try {
    if (sessionStorage.getItem("intro-seen") === "1") {
      setIntroState("done")   // skip on revisit
    } else {
      setIntroState("playing")
    }
  } catch {
    setIntroState("playing")  // private mode / SSR fallback
  }
}, [])
```

During `"loading"`, render a solid-color cover (`<div className="fixed inset-0 z-[60] bg-[#0A0F1C]" />`) that matches the background. This prevents a flash of unstyled content.

---

## The Intro Component

A 3-phase animation:

| Phase | Duration | What Happens |
|-------|----------|--------------|
| `converge` | 0 → 800ms | A soft glow particle scales from 3× to 1×, gathering energy |
| `eye/logo` | 800ms → 2400ms | Your brand icon springs in from scale 0, text label fades in |
| `done` | 2400ms+ | Entire overlay fades to opacity 0, then unmounts |

```tsx
function IntroSequence({ onComplete }: { onComplete: () => void }) {
  const [phase, setPhase] = useState<"converge" | "reveal" | "done">("converge")

  useEffect(() => {
    const t1 = setTimeout(() => setPhase("reveal"), 800)
    const t2 = setTimeout(() => {
      setPhase("done")
      try { sessionStorage.setItem("intro-seen", "1") } catch {}
      onComplete()
    }, 2400)
    return () => { clearTimeout(t1); clearTimeout(t2) }
  }, [onComplete])

  return (
    <motion.div
      className="fixed inset-0 z-[60] flex items-center justify-center bg-[#0A0F1C]"
      animate={phase === "done" ? { opacity: 0 } : { opacity: 1 }}
      transition={{ duration: 0.6 }}
    >
      {/* Gathering glow */}
      <motion.div
        className="absolute w-40 h-40 rounded-full"
        initial={{ scale: 3, opacity: 0 }}
        animate={phase === "converge"
          ? { scale: 1, opacity: 0.4 }
          : { scale: 0.6, opacity: 0 }
        }
        transition={{ duration: 0.8, ease: "easeInOut" }}
        style={{ background: "radial-gradient(circle, rgba(YOUR_COLOR, 0.25) 0%, transparent 70%)" }}
      />

      {/* Brand icon / logo */}
      <motion.div
        initial={{ scale: 0, opacity: 0 }}
        animate={phase === "reveal" || phase === "done"
          ? { scale: 1, opacity: phase === "done" ? 0 : 1 }
          : {}
        }
        transition={{ type: "spring", stiffness: 120, damping: 14 }}
      >
        {/* Your logo or icon component here */}
        <YourLogo size={64} />
      </motion.div>

      {/* Brand name text */}
      <motion.span
        className="absolute mt-24 text-sm tracking-[0.3em] uppercase font-medium"
        style={{ color: "rgba(YOUR_COLOR, 0.6)" }}
        initial={{ opacity: 0, y: 10 }}
        animate={phase === "reveal" ? { opacity: 1, y: 0 } : { opacity: 0 }}
        transition={{ delay: 0.3, duration: 0.5 }}
      >
        Your Brand Name
      </motion.span>
    </motion.div>
  )
}
```

---

## Usage in the Orchestrator

```tsx
{/* Loading cover — matches SSR, prevents flash */}
{introState === "loading" && (
  <div className="fixed inset-0 z-[60] bg-[#0A0F1C]" />
)}

{/* Intro animation */}
{introState === "playing" && (
  <IntroSequence onComplete={() => setIntroState("done")} />
)}

{/* Content mounts ONLY after intro completes */}
{introComplete && (
  <AnimatePresence mode="wait">
    <motion.div key={activePage} ...>
      {renderPageContent(activePage)}
    </motion.div>
  </AnimatePresence>
)}
```

### Why Gate Content on `introComplete`?

If content renders simultaneously with the intro (hidden behind z-index), `AnimatePresence` won't fire the enter animation because the component was already mounted. By gating on `introComplete`, the hero page **mounts fresh** and its `initial → animate` transition plays correctly.

---

## Customization Ideas

| Style | Modification |
|-------|-------------|
| **Minimal** | Skip converge phase, just spring the logo in for 1.5s |
| **Dramatic** | Add particle system converging to center, longer duration |
| **Text-only** | Skip logo, type out the brand name letter by letter |
| **Skip entirely** | Set `introState` to `"done"` immediately in the useEffect |

---

## Timing Guidelines

- Total intro: **2–3 seconds max.** Users shouldn't wait.
- Converge phase: 600–800ms
- Reveal phase: 1000–1600ms
- Fade out: 400–600ms
- If skipping (sessionStorage), content appears **instantly** — no flicker.

---

## Next Step

→ [04 — Mascot / Guide](04-mascot-guide.md)
