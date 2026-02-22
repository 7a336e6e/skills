# 👁️ 04 — Mascot / Guide Component

> An animated character fixed to the viewport edge, commenting on each scene with speech bubbles and personality.

---

## Two Variants

| Variant | When Used | Behavior |
|---------|-----------|----------|
| **Full Mascot** | Story pages (hero, problem, solution, product, stats) | Moves between positions, speech bubble with typewriter, blink, pupil tracking |
| **Mini Mascot** | Embed pages (FAQ, signup, footer) | Small pulsing dot in corner, tooltip on hover (desktop only) |

Both are completely **hidden on mobile** (`hidden md:block`). Mobile screens are too small for a fixed-position character overlaying content.

---

## Position System

Pre-define named positions as CSS calc values:

```typescript
const MASCOT_POSITIONS: Record<string, { x: string; y: string }> = {
  center:     { x: "calc(50% - 24px)",            y: "calc(100% - 120px)" },
  left:       { x: "2%",                          y: "calc(100% - 100px)" },
  right:      { x: "calc(100% - 2% - 48px)",      y: "calc(100% - 100px)" },
  "top-left": { x: "2%",                          y: "calc(100% - 100px)" },
  "top-right":{ x: "calc(100% - 2% - 48px)",      y: "calc(100% - 100px)" },
  hidden:     { x: "50%",                         y: "110%" },  // off-screen
}
```

The mascot animates between positions using spring physics:

```tsx
<motion.div
  className="fixed z-50 hidden md:block"
  animate={{ left: pos.x, top: pos.y }}
  transition={{ type: "spring", stiffness: 50, damping: 16, mass: 1.2 }}
>
```

Low stiffness + high damping = smooth, weighted movement (feels alive, not robotic).

---

## Pupil Tracking

A hook that tracks the mouse and returns a small XY offset for the mascot's eye pupil:

```typescript
function usePupilTracking(ref: React.RefObject<HTMLDivElement | null>) {
  const [offset, setOffset] = useState({ x: 0, y: 0 })

  useEffect(() => {
    let raf = 0
    let mx = 0, my = 0
    const MAX_OFFSET = 3 // px

    const onMove = (e: MouseEvent) => { mx = e.clientX; my = e.clientY }
    window.addEventListener("mousemove", onMove, { passive: true })

    const tick = () => {
      if (ref.current) {
        const rect = ref.current.getBoundingClientRect()
        const cx = rect.left + rect.width / 2
        const cy = rect.top + rect.height / 2
        const dx = mx - cx
        const dy = my - cy
        const dist = Math.sqrt(dx * dx + dy * dy)
        if (dist > 10) {
          const scale = Math.min(MAX_OFFSET / dist, 1)
          setOffset({ x: dx * scale, y: dy * scale })
        }
      }
      raf = requestAnimationFrame(tick)
    }
    raf = requestAnimationFrame(tick)

    return () => {
      window.removeEventListener("mousemove", onMove)
      cancelAnimationFrame(raf)
    }
  }, [ref])

  return offset
}
```

Apply to the pupil element:
```tsx
style={{ transform: `translate(calc(-50% + ${offset.x}px), calc(-50% + ${offset.y}px))` }}
```

---

## Periodic Blink

Every 4–8 seconds, the eye squishes vertically:

```typescript
const [isBlinking, setIsBlinking] = useState(false)

useEffect(() => {
  if (!visible) return
  const scheduleBlink = () => {
    const delay = 4000 + Math.random() * 4000
    return setTimeout(() => {
      setIsBlinking(true)
      setTimeout(() => setIsBlinking(false), 180)
      timerRef = scheduleBlink()
    }, delay)
  }
  let timerRef = scheduleBlink()
  return () => clearTimeout(timerRef)
}, [visible])
```

Apply via inline style on the eye core:
```tsx
style={isBlinking
  ? { transform: "scaleY(0.08)", transition: "transform 0.08s ease-in" }
  : { transition: "transform 0.2s ease-out" }
}
```

Blink closes fast (0.08s) and opens slow (0.2s) — mimics a real blink.

---

## Speech Bubble with Typewriter

Position the bubble relative to the mascot:
- If mascot is in the bottom half: bubble appears **above** (`bottom-[calc(100%+1.5rem)]`)
- If mascot is at the top: bubble appears **below** (`top-[calc(100%+1.5rem)]`)
- Horizontal alignment matches mascot side

```tsx
<AnimatePresence mode="wait">
  {quote && (
    <motion.div
      key={quote}  // re-animates when quote changes
      initial={{ opacity: 0, y: isTop ? -10 : 10, scale: 0.9 }}
      animate={{ opacity: 1, y: 0, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95, filter: "blur(4px)" }}
      transition={{ duration: 0.45, delay: 0.5 }}
    >
      <div
        className="relative px-5 py-3.5 rounded-2xl text-[13px] leading-relaxed
                   backdrop-blur-xl border w-max max-w-[360px]"
        style={{
          background: `linear-gradient(135deg, ${color}12, ${color}08)`,
          borderColor: `${color}25`,
          color: color,
        }}
      >
        {/* Scene label badge */}
        <span className="absolute -top-2.5 left-4 px-2 py-0.5 rounded-full text-[10px]
                        font-semibold uppercase tracking-wider"
              style={{ background: `${color}20`, color, border: `1px solid ${color}30` }}>
          {pageLabel}
        </span>

        <TypewriterText text={quote} />
      </div>
    </motion.div>
  )}
</AnimatePresence>
```

---

## CSS-Only Typewriter Component

Zero React re-renders. The animation is pure CSS:

```tsx
function TypewriterText({ text }: { text: string }) {
  const charCount = text.length
  const durationMs = charCount * 28 // ~28ms per character

  return (
    <span className="mt-1 block relative">
      {/* Invisible text holds layout width */}
      <span className="invisible">{text}</span>

      {/* Visible text revealed by clip-path */}
      <span
        key={text}
        className="absolute inset-0"
        style={{
          clipPath: "inset(0 100% 0 0)",
          animation: `typewriter-clip ${durationMs}ms steps(${charCount}, end) forwards`,
        }}
      >
        {text}
      </span>

      {/* Blinking cursor, hides after typing finishes */}
      <span
        key={text + "-cursor"}
        className="inline-block w-[2px] h-[1em] bg-current ml-0.5 align-text-bottom"
        style={{
          animation: `typewriter-blink 0.5s step-end infinite,
                      typewriter-cursor-hide 0s ${durationMs}ms forwards`,
        }}
      />
    </span>
  )
}
```

The `key={text}` ensures the animation restarts when the quote changes (new page).

---

## Mini Mascot (Embed Pages)

For pages where the full mascot is hidden (FAQ, signup, footer), a small pulsing dot appears in the corner:

```tsx
function MiniMascot({ color, quote, pageLabel, visible }) {
  const [hovered, setHovered] = useState(false)

  return (
    <AnimatePresence>
      {visible && quote && (
        <motion.div
          className="fixed z-50 bottom-8 left-6 hidden md:block"
          onMouseEnter={() => setHovered(true)}
          onMouseLeave={() => setHovered(false)}
        >
          {/* Tooltip on hover */}
          <AnimatePresence>
            {hovered && (
              <motion.div
                initial={{ opacity: 0, x: -8, scale: 0.9 }}
                animate={{ opacity: 1, x: 0, scale: 1 }}
                exit={{ opacity: 0, x: -8, scale: 0.9 }}
                className="absolute bottom-[calc(100%+0.75rem)] left-0"
              >
                <div className="px-4 py-2.5 rounded-xl text-[12px] backdrop-blur-xl border w-max max-w-[300px]" ...>
                  {quote}
                </div>
              </motion.div>
            )}
          </AnimatePresence>

          {/* Pulsing dot */}
          <motion.div
            className="w-3.5 h-3.5 rounded-full"
            style={{ background: color }}
            animate={{ boxShadow: [`0 0 8px ${color}50`, `0 0 18px ${color}70`, `0 0 8px ${color}50`] }}
            transition={{ duration: 2.5, repeat: Infinity, ease: "easeInOut" }}
          />
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

---

## Mascot Visual Styles

The mascot's visual design is up to you. Common approaches:

| Style | Implementation |
|-------|---------------|
| **Eye / orb** | SVG circles + CSS glow rings (used in ThreatSeer) |
| **Character** | CSS art: hood, face, cloak created with divs + border-radius |
| **Emoji / icon** | Simple — just a large emoji or Lucide icon in a glowing circle |
| **3D model** | React Three Fiber — heavyweight, use only if brand requires it |
| **None** | Skip the mascot entirely, use only the speech bubble |

The CSS character approach (divs with `border-radius`, gradients, and shadows) is the best balance of visual impact and performance.

---

## Next Step

→ [05 — Scene Templates](05-scene-templates.md)
