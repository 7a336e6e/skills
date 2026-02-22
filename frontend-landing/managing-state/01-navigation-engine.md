# 🎮 01 — Navigation Engine

> The scroll-hijacking system that drives the storybook: wheel, touch, keyboard, and direct-jump navigation.

---

## Core: `goToPage`

All navigation flows through one function:

```typescript
const [activePage, setActivePage] = useState(0)
const isTransitioning = useRef(false)
const totalPages = PAGES.length

const goToPage = useCallback((pageIndex: number) => {
  if (isTransitioning.current) return              // locked during transition
  const clamped = Math.max(0, Math.min(pageIndex, totalPages - 1))
  if (clamped === activePage) return                // already on this page
  isTransitioning.current = true
  setActivePage(clamped)
  setTimeout(() => { isTransitioning.current = false }, 700)  // unlock after animation
}, [activePage, totalPages])
```

**700ms cooldown** matches the longest transition duration. This prevents:
- Double-firing from fast scroll wheels
- Rapid swipe sequences skipping pages
- Keyboard repeat on held arrow keys

---

## Body Scroll Lock

The very first thing: prevent native page scrolling.

```typescript
useEffect(() => {
  document.body.style.overflow = "hidden"
  return () => { document.body.style.overflow = "" }  // restore on unmount
}, [])
```

This is essential. Without it, wheel/touch events will scroll the body AND trigger page changes.

---

## Scrollable Content Detection

Some pages have internal scroll (embed pages on desktop, all content on mobile). Before changing pages, check if the user is actually scrolling content:

```typescript
const isContentScrollable = useCallback(() => {
  // Check for embed pages (FAQ, footer, etc.)
  const embed = contentRef.current?.querySelector("[data-story-embed]") as HTMLElement
  if (embed) {
    const scrollable = embed.scrollHeight > embed.clientHeight + 5
    const atTop = embed.scrollTop <= 5
    const atBottom = embed.scrollTop + embed.clientHeight >= embed.scrollHeight - 5
    return { scrollable, atTop, atBottom }
  }

  // Check for mobile overflow containers
  const scrollEl = contentRef.current?.querySelector(".overflow-y-auto") as HTMLElement
  if (scrollEl) {
    const scrollable = scrollEl.scrollHeight > scrollEl.clientHeight + 5
    const atTop = scrollEl.scrollTop <= 5
    const atBottom = scrollEl.scrollTop + scrollEl.clientHeight >= scrollEl.scrollHeight - 5
    return { scrollable, atTop, atBottom }
  }

  return { scrollable: false, atTop: true, atBottom: true }
}, [])
```

**The 5px tolerance** prevents edge floating-point issues from blocking navigation when the user is "close enough" to the edge.

---

## Wheel Navigation

```typescript
useEffect(() => {
  const handleWheel = (e: WheelEvent) => {
    if (!introComplete) return  // don't navigate during intro

    const { scrollable, atTop, atBottom } = isContentScrollable()
    if (scrollable) {
      if (e.deltaY > 0 && !atBottom) return  // scrolling down, but content not at bottom
      if (e.deltaY < 0 && !atTop) return     // scrolling up, but content not at top
    }

    e.preventDefault()  // prevent native scroll

    // Debounce: ignore rapid subsequent events
    const now = Date.now()
    if (now - lastWheelTime.current < 700) return
    lastWheelTime.current = now

    // Threshold: ignore tiny scroll deltas (trackpad noise)
    if (e.deltaY > 20) goToPage(activePage + 1)
    else if (e.deltaY < -20) goToPage(activePage - 1)
  }

  window.addEventListener("wheel", handleWheel, { passive: false })
  return () => window.removeEventListener("wheel", handleWheel)
}, [activePage, goToPage, isContentScrollable, introComplete])
```

### Why `passive: false`?

We need to call `e.preventDefault()` to stop native scrolling. Passive listeners can't do this. The `{ passive: false }` option is required for wheel events.

### Why 20px threshold?

Trackpads generate tiny `deltaY` values (1-5px) from inertial scrolling. The 20px threshold filters this noise.

### Why 700ms debounce?

Matches the transition cooldown. Without it, a single scroll gesture (which fires 10+ wheel events) would skip multiple pages.

---

## Touch Navigation

```typescript
const touchStartY = useRef(0)

useEffect(() => {
  const onStart = (e: TouchEvent) => {
    touchStartY.current = e.touches[0].clientY
  }

  const onEnd = (e: TouchEvent) => {
    if (!introComplete) return
    const diff = touchStartY.current - e.changedTouches[0].clientY

    // Minimum 50px swipe to count
    if (Math.abs(diff) < 50) return

    // Check content scroll first
    const { scrollable, atTop, atBottom } = isContentScrollable()
    if (scrollable) {
      if (diff > 0 && !atBottom) return  // swiping up but content not at bottom
      if (diff < 0 && !atTop) return     // swiping down but content not at top
    }

    if (diff > 50) goToPage(activePage + 1)   // swipe up = next
    else goToPage(activePage - 1)              // swipe down = previous
  }

  window.addEventListener("touchstart", onStart, { passive: true })
  window.addEventListener("touchend", onEnd, { passive: true })
  return () => {
    window.removeEventListener("touchstart", onStart)
    window.removeEventListener("touchend", onEnd)
  }
}, [activePage, goToPage, isContentScrollable, introComplete])
```

### Why `passive: true`?

Touch listeners don't need `preventDefault()` — we let the browser handle the touch event naturally and only act on the swipe distance in `touchend`.

### Why 50px threshold?

Below 50px, the user probably tapped or did a small scroll gesture, not a deliberate swipe.

---

## Keyboard Navigation

```typescript
useEffect(() => {
  const handleKey = (e: KeyboardEvent) => {
    if (!introComplete) return

    const { scrollable, atTop, atBottom } = isContentScrollable()

    if (e.key === "ArrowDown" || e.key === " ") {
      if (scrollable && !atBottom) return  // let content scroll
      e.preventDefault()
      goToPage(activePage + 1)
    }
    if (e.key === "ArrowUp") {
      if (scrollable && !atTop) return
      e.preventDefault()
      goToPage(activePage - 1)
    }
  }

  window.addEventListener("keydown", handleKey)
  return () => window.removeEventListener("keydown", handleKey)
}, [activePage, goToPage, isContentScrollable, introComplete])
```

Space bar and ArrowDown both go forward. ArrowUp goes back.

---

## Direct Jump (from CTAs)

CTA buttons inside scenes can jump to any page:

```tsx
// In HeroPage:
<Button onClick={() => onJump(9)}>Get Early Access</Button>

// onJump is just goToPage passed as a prop:
case "hero": return <HeroPage onJump={goToPage} />
```

---

## External Navigation (from Navbar)

Expose `goToPage` to parent components via a callback:

```tsx
// In the orchestrator:
export default function Journey({
  onNavigateReady
}: {
  onNavigateReady?: (goTo: (page: number) => void) => void
}) {
  // ...
  useEffect(() => {
    onNavigateReady?.(goToPage)
  }, [goToPage, onNavigateReady])
}

// In the parent page:
const goToRef = useRef(null)
<Journey onNavigateReady={(goTo) => { goToRef.current = goTo }} />
<Navbar onCtaClick={() => goToRef.current?.(9)} />
```

This also supports hash-based navigation (e.g., `/#signup` → jump to signup page on load):

```typescript
const HASH_PAGE_MAP = { "#signup": 9, "#demo": 8, "#faq": 10 }

const handleNavigateReady = (goTo) => {
  goToRef.current = goTo
  const hash = window.location.hash
  if (hash && HASH_PAGE_MAP[hash] !== undefined) {
    setTimeout(() => goTo(HASH_PAGE_MAP[hash]), 100)
  }
}
```

---

## The Full Orchestrator Render

Putting it all together:

```tsx
return (
  <div className="bg-[#0A0F1C] h-screen w-screen overflow-hidden relative">
    <StarField />

    {/* Loading cover */}
    {introState === "loading" && <div className="fixed inset-0 z-[60] bg-[#0A0F1C]" />}

    {/* Intro animation */}
    {introState === "playing" && <IntroSequence onComplete={() => setIntroState("done")} />}

    {/* Scene background glow */}
    <motion.div
      className="fixed inset-0 pointer-events-none z-[1]"
      animate={{ background: currentPage.bgGlow ?? "none" }}
      transition={{ duration: 1.2, ease: "easeOut" }}
    />

    {/* Mascot */}
    <Mascot
      color={currentPage.color}
      position={currentPage.mascotPosition}
      quote={currentPage.mascotQuote}
      visible={currentPage.mascotPosition !== "hidden" && introComplete}
      pageLabel={currentPage.label}
    />

    {/* Mini mascot for embed pages */}
    <MiniMascot
      color={currentPage.color}
      quote={currentPage.mascotQuote}
      pageLabel={currentPage.label}
      visible={isEmbedPage && !!currentPage.mascotQuote && introComplete}
    />

    {/* Page indicator */}
    {introComplete && <PageIndicator pages={PAGES} active={activePage} onDotClick={goToPage} />}

    {/* Content viewport */}
    <div ref={contentRef} className="relative z-[2] h-full w-full">
      {introComplete && (
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
      )}
    </div>
  </div>
)
```

---

## Z-Index Stacking Order

| Layer | Z-Index | Content |
|-------|---------|---------|
| Background (stars, glow) | z-0 / z-[1] | Ambient effects |
| Content viewport | z-[2] | Scene pages |
| Mascot | z-50 | Guide character |
| Page indicator | z-50 | Navigation dots |
| Intro sequence | z-[60] | Covers everything during intro |
| Loading cover | z-[60] | SSR hydration cover |

---

## Next Step

→ [Integrating API](../integrating-api/SKILL.md) — Connect forms, analytics, dynamic data.
