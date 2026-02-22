# 📍 06 — Page Indicator

> A vertical dot timeline fixed to the right edge, showing progress and enabling direct navigation.

---

## Design

```
           ●  Awaken     ← active: large dot + glow + label
           │
           ◉  Threat     ← past: medium dot, muted color
           │
           ◉  Solution
           │
           ○  Product    ← future: tiny dot, barely visible
           │
           ○  Stats
           │
           ○  FAQ
           │
           ○  Footer
```

---

## Implementation

```tsx
function PageIndicator({
  pages,
  active,
  onDotClick,
}: {
  pages: StoryPage[]
  active: number
  onDotClick: (i: number) => void
}) {
  return (
    <div className="fixed right-2 md:right-5 top-1/2 -translate-y-1/2 z-50
                    flex flex-col items-end gap-0">
      {pages.map((page, i) => {
        const isActive = i === active
        const isPast = i < active
        return (
          <button
            key={page.id}
            onClick={() => onDotClick(i)}
            className="group relative flex items-center gap-3 py-[4px] md:py-[6px] pr-1"
            aria-label={`Go to ${page.label}`}
          >
            {/* Label — visible on hover (desktop only) */}
            <span
              className={`hidden md:inline text-[10px] font-medium uppercase tracking-wider
                         whitespace-nowrap select-none transition-all duration-300 ${
                isActive
                  ? "opacity-100 translate-x-0"
                  : "opacity-0 translate-x-2 group-hover:opacity-70 group-hover:translate-x-0"
              }`}
              style={{ color: isActive ? page.color : "rgba(255,255,255,0.4)" }}
            >
              {page.label}
            </span>

            {/* Dot + connector */}
            <div className="relative flex flex-col items-center">
              <motion.div
                className="rounded-full transition-all duration-300
                           group-hover:shadow-[0_0_10px_rgba(255,255,255,0.15)]"
                animate={{
                  width: isActive ? 8 : isPast ? 5 : 4,
                  height: isActive ? 8 : isPast ? 5 : 4,
                  backgroundColor: isActive
                    ? page.color
                    : isPast
                    ? `${page.color}60`
                    : "rgba(255,255,255,0.15)",
                  boxShadow: isActive ? `0 0 14px ${page.color}60` : "none",
                }}
                transition={{ duration: 0.3 }}
              />
              {/* Connecting line between dots */}
              {i < pages.length - 1 && (
                <div
                  className="w-px h-[10px] mt-[2px] transition-colors duration-300"
                  style={{
                    background: i < active
                      ? `${pages[i].color}40`
                      : "rgba(255,255,255,0.08)"
                  }}
                />
              )}
            </div>
          </button>
        )
      })}
    </div>
  )
}
```

---

## Sizing Strategy

| State | Dot Size | Color | Shadow |
|-------|----------|-------|--------|
| **Active** | 8px | Scene's brand color | `0 0 14px color60` |
| **Past** | 5px | Scene color at 60% opacity | None |
| **Future** | 4px | `rgba(255,255,255,0.15)` | None |

### Mobile Adjustments

- Position: `right-2` instead of `right-5`
- Labels: `hidden md:inline` (hidden on mobile)
- Padding: `py-[4px]` instead of `py-[6px]`
- Dots remain visible — they're the only navigation cue on mobile besides swiping

---

## Mounting

Only render after intro completes:

```tsx
{introComplete && (
  <PageIndicator pages={PAGES} active={activePage} onDotClick={goToPage} />
)}
```

---

## Accessibility

- Each dot is a `<button>` with `aria-label`
- Labels provide context on hover
- Keyboard-focusable and clickable
- Color contrast: the glow + size difference makes active dot identifiable without color alone

---

## Next Step

→ [07 — Mobile Responsive](07-mobile-responsive.md)
