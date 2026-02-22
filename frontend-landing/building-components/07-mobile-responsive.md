# 📱 07 — Mobile Responsiveness

> Make the storybook experience feel native on every screen size.

---

## Breakpoint Strategy

| Breakpoint | Width | Target |
|------------|-------|--------|
| Base | < 640px | Phones (iPhone SE to iPhone 15) |
| `sm` | ≥ 640px | Large phones, small tablets |
| `md` | ≥ 768px | Tablets, small laptops |
| `lg` | ≥ 1024px | Laptops, desktops |

The critical split is at `md` (768px). Below `md`, the experience is a vertically-scrolling content viewer. Above `md`, it's the full storybook with mascot and effects.

---

## Element Visibility Matrix

| Element | Mobile (< md) | Desktop (≥ md) |
|---------|--------------|----------------|
| Mascot (full) | **Hidden** | Visible |
| Mini mascot dot | **Hidden** | Visible |
| Speech bubbles | **Hidden** | Visible |
| Page indicator dots | Visible (compact) | Visible (full + labels) |
| Page indicator labels | **Hidden** | Show on hover |
| Scene content | Scrollable overflow | Fixed viewport |
| Trust badges | Visible (smaller) | Visible |
| Star field / background | Visible | Visible |
| Intro sequence | Visible | Visible |

---

## Text Scale Strategy

Use Tailwind responsive prefixes for all text:

| Element | Mobile | sm | md+ |
|---------|--------|-----|-----|
| Hero headline | `text-3xl` | `text-5xl` | `text-7xl` |
| Scene headline | `text-2xl` | `text-4xl` | `text-5xl` |
| Body text | `text-sm` | — | `text-base` |
| Badge/label | `text-xs` | — | `text-xs` |
| Stat number | `text-2xl` | — | `text-4xl` |
| Caption/metric | `text-[10px]` | — | `text-[11px]` |

---

## Layout Collapse Pattern

Grids collapse on mobile:

```tsx
// 2-column → full width
className="grid md:grid-cols-[1.1fr_0.9fr] gap-6 md:gap-10"

// 4-column → 2×2
className="grid grid-cols-2 md:grid-cols-4 gap-3 md:gap-4"

// 2×2 → stacked
className="grid grid-cols-1 sm:grid-cols-2 gap-5"
```

---

## Content Overflow Handling

On mobile, scenes can be taller than the viewport. Use this pattern:

```tsx
<div className="flex items-start md:items-center justify-center h-full
                px-4 md:px-6 pt-20 md:pt-0 pb-32 md:pb-48
                overflow-y-auto md:overflow-visible">
```

Key parts:
- `items-start md:items-center` — content starts at top on mobile (scrollable), centered on desktop
- `pt-20 md:pt-0` — top padding prevents content from touching the top edge on mobile
- `pb-32 md:pb-48` — bottom padding: less on mobile (no mascot), more on desktop
- `overflow-y-auto md:overflow-visible` — mobile scrolls within the scene, desktop doesn't

### Scroll Detection

The navigation engine must detect when an embed page or mobile overflow container is mid-scroll, to avoid changing pages while the user is scrolling content:

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

**The rule:** Only allow page change when the content is at the top (scrolling up) or bottom (scrolling down) of its internal scroll.

---

## Touch Navigation

```typescript
const touchStartY = useRef(0)

// Record start position
const onStart = (e: TouchEvent) => { touchStartY.current = e.touches[0].clientY }

// Calculate swipe distance
const onEnd = (e: TouchEvent) => {
  const diff = touchStartY.current - e.changedTouches[0].clientY
  if (Math.abs(diff) < 50) return  // threshold: 50px minimum

  const { scrollable, atTop, atBottom } = isContentScrollable()
  if (scrollable) {
    if (diff > 0 && !atBottom) return  // swiping up but not at bottom → let content scroll
    if (diff < 0 && !atTop) return     // swiping down but not at top → let content scroll
  }

  if (diff > 50) goToPage(activePage + 1)   // swipe up → next page
  else goToPage(activePage - 1)              // swipe down → previous page
}
```

---

## Card & Spacing Compression

```tsx
// Padding
className="p-3 md:p-5"

// Gaps
className="gap-2 md:gap-3"
className="gap-6 md:gap-10 lg:gap-16"

// Margin
className="mt-5 md:mt-8"
className="mb-6 md:mb-12"

// Icon sizes
className="w-4 h-4 md:w-5 md:h-5"
className="w-9 h-9 md:w-11 md:h-11"
className="w-12 h-12 md:w-16 md:h-16"
```

---

## Elements to Show/Hide

```tsx
// Hidden on mobile, visible on desktop
className="hidden md:block"
className="hidden md:flex"
className="hidden md:inline"

// Hidden on mobile but keep in DOM (for layout)
className="invisible md:visible"

// Visible on mobile, hidden on desktop (rare)
className="md:hidden"

// Visible everywhere but different sizes
className="hidden sm:block"     // hide on very small, show on small+
```

---

## Testing Checklist

- [ ] iPhone SE (375px) — nothing overflows horizontally
- [ ] iPhone 15 Pro (393px) — all text readable
- [ ] iPad Mini (768px) — `md` breakpoint triggers correctly
- [ ] iPad (820px) — mascot appears, layout switches
- [ ] Landscape phone — content doesn't break
- [ ] Touch targets ≥ 44px (dots, buttons)
- [ ] Swipe up/down navigates between pages
- [ ] Content scroll works before page navigation fires
- [ ] No horizontal scroll on any page

---

## Next Step

→ [Managing State](../managing-state/SKILL.md) — The navigation engine in detail.
