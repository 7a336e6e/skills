# Testing Frontend — Verification & Quality Assurance

> Ensure the storybook experience works flawlessly across devices,
> browsers, and interaction modes before every deploy.

---

## Overview

A full-screen, animation-heavy landing page has **unique testing
challenges** that standard unit tests don't cover:

| Challenge                    | Why Standard Tests Fail                  |
| ---------------------------- | ---------------------------------------- |
| Scene transitions            | Depend on wheel/touch/keyboard events    |
| CSS-only animations          | Not observable via React Testing Library |
| 60 fps performance           | Requires real browser profiling          |
| Mobile vs desktop layouts    | Visibility classes, not conditional JSX  |
| Intro sequence + skip logic  | sessionStorage + hydration timing        |

This guide covers the **four testing layers** needed:

```
Layer 4:  Performance & Lighthouse CI        (automated budget)
Layer 3:  Visual Regression Testing          (screenshot diffs)
Layer 2:  Integration / E2E Testing          (real browser interactions)
Layer 1:  Unit & Component Tests             (logic + render checks)
```

---

## 1 — Unit & Component Tests

### Setup

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "."),
    },
  },
});
```

```ts
// tests/setup.ts
import "@testing-library/jest-dom/vitest";

// Mock framer-motion to avoid animation timing issues
vi.mock("framer-motion", async () => {
  const actual = await vi.importActual("framer-motion");
  return {
    ...actual,
    AnimatePresence: ({ children }: any) => children,
    motion: new Proxy(
      {},
      {
        get: (_target, prop) => {
          // Return a simple component that passes through props
          return ({ children, ...props }: any) => {
            const Component = prop as string;
            return <Component {...props}>{children}</Component>;
          };
        },
      }
    ),
  };
});
```

### What to Unit Test

| Component / Logic         | What to Assert                           |
| ------------------------- | ---------------------------------------- |
| Scene config array        | All scenes have required fields          |
| TypewriterText            | Renders correct text content             |
| Mascot speech bubbles     | Correct text per scene index             |
| Form validation           | Email format, required fields            |
| Hash-to-page mapping      | Known hashes → correct page indices      |
| sessionStorage intro skip | Returns correct skip state               |

### Example: Scene Config Validation

```tsx
// tests/scene-config.test.ts
import { PAGES } from "@/components/seer/config";

describe("Scene Configuration", () => {
  it("every scene has a unique id", () => {
    const ids = PAGES.map((p) => p.id);
    expect(new Set(ids).size).toBe(ids.length);
  });

  it("every scene has a title and subtitle", () => {
    PAGES.forEach((page, i) => {
      expect(page.title, `Scene ${i} missing title`).toBeTruthy();
      expect(page.subtitle, `Scene ${i} missing subtitle`).toBeTruthy();
    });
  });

  it("mascot has a position for every scene", () => {
    PAGES.forEach((page, i) => {
      expect(page.mascotPosition, `Scene ${i} missing mascotPosition`).toBeDefined();
    });
  });

  it("all gradient values are valid CSS", () => {
    PAGES.forEach((page) => {
      if (page.gradient) {
        expect(page.gradient).toMatch(/^(linear|radial|conic)-gradient/);
      }
    });
  });
});
```

### Example: Hash Navigation Mapping

```tsx
// tests/hash-navigation.test.ts
const HASH_MAP: Record<string, number> = {
  "#signup": 9,
  "#demo": 8,
  "#faq": 10,
};

describe("Hash Navigation", () => {
  Object.entries(HASH_MAP).forEach(([hash, expected]) => {
    it(`${hash} → scene ${expected}`, () => {
      expect(HASH_MAP[hash]).toBe(expected);
    });
  });

  it("unknown hash returns undefined (falls back to 0)", () => {
    expect(HASH_MAP["#nonexistent"]).toBeUndefined();
  });
});
```

### Example: Intro Skip Logic

```tsx
// tests/intro-skip.test.ts
describe("Intro Sequence Skip", () => {
  beforeEach(() => {
    sessionStorage.clear();
  });

  it("first visit does not skip intro", () => {
    const seen = sessionStorage.getItem("introSeen");
    expect(seen).toBeNull();
  });

  it("sets flag after intro completes", () => {
    sessionStorage.setItem("introSeen", "true");
    expect(sessionStorage.getItem("introSeen")).toBe("true");
  });

  it("subsequent visits skip intro", () => {
    sessionStorage.setItem("introSeen", "true");
    const shouldSkip = sessionStorage.getItem("introSeen") === "true";
    expect(shouldSkip).toBe(true);
  });
});
```

---

## 2 — Integration / E2E Tests (Playwright)

### Setup

```bash
npm install -D @playwright/test
npx playwright install
```

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  retries: 1,
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
    video: "on-first-retry",
  },
  projects: [
    { name: "Desktop Chrome", use: { ...devices["Desktop Chrome"] } },
    { name: "Desktop Firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "Mobile Safari", use: { ...devices["iPhone 14"] } },
    { name: "Mobile Chrome", use: { ...devices["Pixel 7"] } },
  ],
  webServer: {
    command: "npm run dev",
    port: 3000,
    reuseExistingServer: true,
  },
});
```

### Test: Scene Navigation via Scroll

```ts
// tests/e2e/navigation.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Scene Navigation", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/");
    // Wait for intro to complete or skip it
    await page.waitForTimeout(4000); // Intro duration
  });

  test("wheel down advances to next scene", async ({ page }) => {
    // Start at scene 0 (Hero)
    const hero = page.locator("[data-scene='hero']");
    await expect(hero).toBeVisible();

    // Scroll down
    await page.mouse.wheel(0, 200);
    await page.waitForTimeout(1500); // Transition duration

    // Should now be on scene 1
    const scene1 = page.locator("[data-scene='threat']");
    await expect(scene1).toBeVisible();
  });

  test("keyboard ArrowDown advances scene", async ({ page }) => {
    await page.keyboard.press("ArrowDown");
    await page.waitForTimeout(1500);

    const scene1 = page.locator("[data-scene='threat']");
    await expect(scene1).toBeVisible();
  });

  test("page indicator dots are clickable", async ({ page }) => {
    // Click the 3rd dot
    const dot = page.locator("[data-page-dot='2']");
    await dot.click();
    await page.waitForTimeout(1500);

    // Scene 2 should be visible
    const scene2 = page.locator("[data-scene='solution']");
    await expect(scene2).toBeVisible();
  });
});
```

### Test: Touch Swipe (Mobile)

```ts
// tests/e2e/touch-navigation.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Touch Navigation", () => {
  test.use({ ...devices["iPhone 14"] });

  test("swipe up advances to next scene", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4000);

    // Simulate swipe up
    await page.touchscreen.tap(200, 600);
    await page.mouse.move(200, 600);
    await page.mouse.down();
    await page.mouse.move(200, 200, { steps: 10 }); // Swipe up
    await page.mouse.up();

    await page.waitForTimeout(1500);

    const scene1 = page.locator("[data-scene='threat']");
    await expect(scene1).toBeVisible();
  });
});
```

### Test: Intro Sequence

```ts
// tests/e2e/intro.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Intro Sequence", () => {
  test("plays full intro on first visit", async ({ page, context }) => {
    // Clear session storage
    await context.clearCookies();

    await page.goto("/");

    // Intro overlay should be visible
    const intro = page.locator("[data-testid='intro-sequence']");
    await expect(intro).toBeVisible();

    // Wait for intro to complete
    await page.waitForTimeout(4500);

    // Hero scene should now be visible
    const hero = page.locator("[data-scene='hero']");
    await expect(hero).toBeVisible();
  });

  test("skips intro on subsequent visits", async ({ page }) => {
    // First visit
    await page.goto("/");
    await page.waitForTimeout(5000);

    // sessionStorage should now have the flag
    const flag = await page.evaluate(() =>
      sessionStorage.getItem("introSeen")
    );
    expect(flag).toBe("true");

    // Reload — intro should be skipped
    await page.reload();
    await page.waitForTimeout(500);

    const hero = page.locator("[data-scene='hero']");
    await expect(hero).toBeVisible();
  });
});
```

### Test: Form Submission

```ts
// tests/e2e/signup-form.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Sign-Up Form", () => {
  test("submits valid form and shows success", async ({ page }) => {
    await page.goto("/#signup");
    await page.waitForTimeout(2000);

    await page.fill("[name='email']", "test@example.com");
    await page.fill("[name='name']", "Test User");
    await page.click("[type='submit']");

    // Should show success state
    const success = page.locator("[data-testid='signup-success']");
    await expect(success).toBeVisible({ timeout: 5000 });
  });

  test("shows validation error for invalid email", async ({ page }) => {
    await page.goto("/#signup");
    await page.waitForTimeout(2000);

    await page.fill("[name='email']", "not-an-email");
    await page.click("[type='submit']");

    const error = page.locator("[data-testid='email-error']");
    await expect(error).toBeVisible();
  });
});
```

---

## 3 — Visual Regression Testing

### Why Screenshots Matter

CSS-only animations, glassmorphism effects, and gradient overlays
can break silently — no test runner catches a misaligned gradient.

### Playwright Visual Comparisons

```ts
// tests/e2e/visual.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Visual Regression", () => {
  const scenes = ["hero", "threat", "solution", "product", "process", "stats"];

  for (const scene of scenes) {
    test(`scene "${scene}" matches snapshot`, async ({ page }) => {
      await page.goto("/");
      await page.waitForTimeout(4500); // Wait for intro

      // Navigate to the target scene
      await page.evaluate((s) => {
        // Use the exposed goToPage function
        (window as any).__goToPage?.(s);
      }, scene);

      await page.waitForTimeout(2000); // Wait for transition

      // Take screenshot and compare
      await expect(page).toHaveScreenshot(`${scene}-desktop.png`, {
        maxDiffPixelRatio: 0.02, // Allow 2% pixel difference
        animations: "disabled",   // Freeze CSS animations
      });
    });
  }
});
```

### Mobile Visual Regression

```ts
test.describe("Mobile Visual Regression", () => {
  test.use({ ...devices["iPhone 14"] });

  test("hero scene mobile layout", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4500);

    await expect(page).toHaveScreenshot("hero-mobile.png", {
      maxDiffPixelRatio: 0.02,
      animations: "disabled",
    });
  });

  test("mascot is hidden on mobile", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4500);

    const mascot = page.locator("[data-testid='mascot']");
    await expect(mascot).not.toBeVisible();
  });
});
```

### Updating Snapshots

```bash
# Update all snapshots after intentional visual changes
npx playwright test --update-snapshots

# Update specific test snapshots
npx playwright test visual.spec.ts --update-snapshots
```

---

## 4 — Performance Testing

### Lighthouse CI

```bash
npm install -D @lhci/cli
```

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:3000"],
      "startServerCommand": "npm run start",
      "numberOfRuns": 3,
      "settings": {
        "preset": "desktop"
      }
    },
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "categories:accessibility": ["error", { "minScore": 0.9 }],
        "categories:best-practices": ["warn", { "minScore": 0.85 }],
        "first-contentful-paint": ["error", { "maxNumericValue": 1500 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "total-blocking-time": ["error", { "maxNumericValue": 200 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }]
      }
    }
  }
}
```

```bash
# Run Lighthouse CI
npx lhci autorun
```

### Manual Performance Profiling

For animation-specific performance that automated tools miss:

```
1. Open Chrome DevTools → Performance tab
2. Enable "CPU 4x slowdown" (simulates low-end device)
3. Start recording
4. Navigate through all scenes (wheel / click dots)
5. Stop recording
6. Check for:
   - Frame drops below 60 fps during transitions
   - Long tasks (> 50ms) blocking the main thread
   - Excessive layout recalculations (reflows)
   - Memory leaks (heap growing without release)
```

### Runtime Animation Frame Budget

```tsx
// Debug-only: monitor frame rate during development
if (process.env.NODE_ENV === "development") {
  let lastTime = performance.now();
  let frames = 0;

  const measureFPS = () => {
    frames++;
    const now = performance.now();
    if (now - lastTime >= 1000) {
      console.log(`FPS: ${frames}`);
      frames = 0;
      lastTime = now;
    }
    requestAnimationFrame(measureFPS);
  };

  requestAnimationFrame(measureFPS);
}
```

---

## 5 — Accessibility Testing

### Automated A11y Checks

```bash
npm install -D @axe-core/playwright
```

```ts
// tests/e2e/accessibility.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test.describe("Accessibility", () => {
  test("hero scene has no critical a11y violations", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4500);

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa"])
      .analyze();

    expect(results.violations.filter((v) => v.impact === "critical")).toEqual([]);
  });

  test("keyboard navigation works through all scenes", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4500);

    // Tab should reach interactive elements
    await page.keyboard.press("Tab");
    const focused = await page.evaluate(() =>
      document.activeElement?.tagName
    );
    expect(focused).toBeTruthy();
  });

  test("all images have alt text", async ({ page }) => {
    await page.goto("/");
    await page.waitForTimeout(4500);

    const imagesWithoutAlt = await page.$$eval(
      "img:not([alt])",
      (imgs) => imgs.length
    );
    expect(imagesWithoutAlt).toBe(0);
  });
});
```

### Manual A11y Checklist

- [ ] All interactive elements reachable via keyboard (Tab/Shift+Tab)
- [ ] ArrowUp/ArrowDown navigate scenes (plus Escape for special actions)
- [ ] Focus indicators visible on all interactive elements
- [ ] Color contrast ratio ≥ 4.5:1 for body text, ≥ 3:1 for large text
- [ ] `prefers-reduced-motion` disables non-essential animations
- [ ] Screen reader announces scene changes (aria-live region)
- [ ] Form fields have associated labels

### Reduced Motion Support

```tsx
// Detect and respect user preference
const prefersReducedMotion =
  typeof window !== "undefined" &&
  window.matchMedia("(prefers-reduced-motion: reduce)").matches;

// Use for framer-motion variants
const transition = prefersReducedMotion
  ? { duration: 0 }
  : { type: "spring", stiffness: 120, damping: 20 };

// CSS fallback
// @media (prefers-reduced-motion: reduce) {
//   *, *::before, *::after {
//     animation-duration: 0.01ms !important;
//     transition-duration: 0.01ms !important;
//   }
// }
```

---

## 6 — Cross-Browser Test Matrix

| Browser         | Desktop | Mobile | Priority |
| --------------- | ------- | ------ | -------- |
| Chrome          | ✅      | ✅     | P0       |
| Safari          | ✅      | ✅     | P0       |
| Firefox         | ✅      | —      | P1       |
| Edge            | ✅      | —      | P2       |
| Samsung Internet| —       | ✅     | P2       |

### Known Cross-Browser Issues

| Issue                          | Browser  | Workaround                      |
| ------------------------------ | -------- | ------------------------------- |
| `backdrop-filter` not working  | Firefox  | Provide solid fallback bg       |
| Scroll event passive default   | Chrome   | `{ passive: false }` explicitly |
| `will-change` memory usage     | Safari   | Limit to actively animating     |
| Touch event vs pointer event   | Safari   | Use both `touch*` and `pointer*`|

---

## 7 — CI Pipeline Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Test & Quality

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx vitest run

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npx lhci autorun
```

---

## Test Commands Quick Reference

```bash
# Unit tests
npx vitest              # Watch mode
npx vitest run          # Single run
npx vitest run --coverage  # With coverage

# E2E tests
npx playwright test                    # All browsers
npx playwright test --project="Desktop Chrome"  # One browser
npx playwright test --ui               # Interactive UI mode
npx playwright test --debug            # Step-through debugger

# Visual regression
npx playwright test visual.spec.ts
npx playwright test --update-snapshots # After intentional changes

# Lighthouse
npx lhci autorun

# Type checking
npx tsc --noEmit

# Lint
npm run lint
```

---

## Testing Checklist

- [ ] Scene config validation (unit test)
- [ ] Hash navigation mapping (unit test)
- [ ] Intro skip logic (unit test)
- [ ] Wheel navigation advances scenes (E2E)
- [ ] Keyboard navigation works (E2E)
- [ ] Touch swipe navigation works (E2E, mobile project)
- [ ] Form submit + validation (E2E)
- [ ] Visual snapshots for each scene (visual regression)
- [ ] Mobile layout snapshots (visual regression)
- [ ] Mascot hidden on mobile (E2E assertion)
- [ ] Lighthouse scores within budget (performance)
- [ ] No critical a11y violations (accessibility)
- [ ] Works in Chrome, Safari, Firefox (cross-browser)
- [ ] No console errors during full navigation (E2E)
- [ ] CI pipeline runs all test layers on PR

---

**← Back to:** [Master SKILL.md](../SKILL.md)
