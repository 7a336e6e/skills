# 🧠 Managing State

> Navigation engine, scroll hijacking, and state management for the storybook.

---

## Sub-Skills

| # | File | What It Covers |
|---|------|----------------|
| 01 | [Navigation Engine](01-navigation-engine.md) | Wheel, touch, keyboard handlers + scroll detection + transition lock |

---

## State Overview

The entire storybook has surprisingly little state:

| State | Type | Purpose |
|-------|------|---------|
| `introState` | `"loading" \| "playing" \| "done"` | Controls intro sequence visibility |
| `activePage` | `number` | Current scene index (0-based) |
| `isTransitioning` | `ref<boolean>` | Lock to prevent double-navigation |
| `lastWheelTime` | `ref<number>` | Debounce wheel events |
| `touchStartY` | `ref<number>` | Touch swipe start position |

No context providers. No state management libraries. Everything lives in the orchestrator component's local state.

### Why Refs for Transition/Wheel?

- `isTransitioning` changes rapidly but doesn't need to trigger re-renders
- `lastWheelTime` is a timestamp checked synchronously — a state update would be too slow
- `touchStartY` is saved on touchstart, read on touchend — no render needed between

---

## Next Step

Start with → [01 — Navigation Engine](01-navigation-engine.md)
