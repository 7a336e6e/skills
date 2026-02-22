# Integrating API — Data, Forms & Analytics

> Wire up dynamic data, capture leads, and track engagement without
> slowing down the immersive experience.

---

## Overview

A storybook landing page is mostly **static visual theatre**, but three
API touch-points turn it into a business tool:

| Touch-Point           | Purpose                            | Timing              |
| --------------------- | ---------------------------------- | ------------------- |
| **Form submission**   | Capture leads / sign-ups           | User-initiated      |
| **Analytics events**  | Track scroll depth, scene views    | Passive / automatic |
| **Dynamic content**   | Pricing, testimonials, blog posts  | On mount or lazy    |

Each must be integrated so it **never blocks rendering** or interferes
with the 60 fps animation pipeline.

---

## 1 — Form Submission (Contact / Sign-Up / Early Access)

### Architecture

```
┌──────────────────────────┐
│  StoryEmbedPage          │  ← scrollable wrapper scene
│  ┌────────────────────┐  │
│  │  SignUpFormSection  │  │  ← standalone component
│  │  ┌──────────────┐  │  │
│  │  │  <form>       │  │  │
│  │  │  state + UX   │  │  │
│  │  └──────────────┘  │  │
│  └────────────────────┘  │
└──────────────────────────┘
```

### Pattern: React Server Action (App Router)

```tsx
// app/actions/signup.ts
"use server";

import { z } from "zod";

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
  company: z.string().optional(),
});

export async function submitSignup(formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { ok: false, errors: parsed.error.flatten().fieldErrors };
  }

  // Forward to your backend / CRM / Supabase / etc.
  await fetch(process.env.API_URL + "/leads", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(parsed.data),
  });

  return { ok: true };
}
```

### Pattern: Client Form with Server Action

```tsx
"use client";

import { useActionState } from "react";
import { submitSignup } from "@/app/actions/signup";

export function SignUpForm() {
  const [state, action, pending] = useActionState(submitSignup, null);

  return (
    <form action={action} className="space-y-4">
      <input
        name="email"
        type="email"
        required
        className="glass-card w-full px-4 py-3 bg-white/5 …"
        disabled={pending}
      />
      {state?.errors?.email && (
        <p className="text-red-400 text-sm">{state.errors.email}</p>
      )}

      <button
        type="submit"
        disabled={pending}
        className="btn-glow w-full …"
      >
        {pending ? "Submitting…" : "Get Early Access"}
      </button>

      {state?.ok && (
        <motion.div
          initial={{ opacity: 0, y: 10 }}
          animate={{ opacity: 1, y: 0 }}
          className="text-brand-teal text-center"
        >
          ✓ You're on the list!
        </motion.div>
      )}
    </form>
  );
}
```

### Pattern: Client-Only Fetch (No Server Action)

For pages deployed as static exports or when using an external API
directly:

```tsx
const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  setLoading(true);

  try {
    const res = await fetch("/api/signup", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, name }),
    });

    if (!res.ok) throw new Error("Submission failed");
    setSuccess(true);
  } catch (err) {
    setError(err instanceof Error ? err.message : "Something went wrong");
  } finally {
    setLoading(false);
  }
};
```

### UX Rules for Forms Inside Storybook Pages

| Rule | Why |
|------|-----|
| Wrap in a **scrollable embed scene** | Keyboard opening on mobile shifts viewport |
| Disable page navigation while form is focused | Prevent accidental swipe-away mid-typing |
| Show inline validation, never `alert()` | Matches premium feel |
| Animate success state with `motion.div` | Consistent with rest of transitions |
| Auto-focus first field when scene is active | Reduce friction |

```tsx
// Disable wheel navigation while form is focused
const formRef = useRef<HTMLFormElement>(null);

useEffect(() => {
  const form = formRef.current;
  if (!form) return;

  const stop = (e: WheelEvent) => e.stopPropagation();
  form.addEventListener("wheel", stop, { passive: false });
  return () => form.removeEventListener("wheel", stop);
}, []);
```

---

## 2 — Analytics & Event Tracking

### Scene View Tracking

Fire analytics events as users navigate through scenes:

```tsx
const goToPage = useCallback((index: number) => {
  // … transition logic …

  // Track scene view
  if (typeof window !== "undefined" && window.gtag) {
    window.gtag("event", "scene_view", {
      scene_name: PAGES[index].id,
      scene_index: index,
      event_category: "engagement",
    });
  }
}, []);
```

### Scroll Depth / Furthest Scene Reached

```tsx
const furthestScene = useRef(0);

const goToPage = useCallback((index: number) => {
  if (index > furthestScene.current) {
    furthestScene.current = index;

    // Fire only when user reaches a new deepest point
    trackEvent("furthest_scene", {
      scene_index: index,
      scene_name: PAGES[index].id,
      total_scenes: PAGES.length,
      percent: Math.round((index / (PAGES.length - 1)) * 100),
    });
  }
}, []);
```

### CTA Click Tracking

```tsx
function TrackedButton({
  event,
  children,
  ...props
}: ButtonProps & { event: string }) {
  return (
    <button
      {...props}
      onClick={(e) => {
        trackEvent("cta_click", { button: event });
        props.onClick?.(e);
      }}
    >
      {children}
    </button>
  );
}
```

### Generic `trackEvent` Helper

```tsx
// lib/analytics.ts
export function trackEvent(
  name: string,
  params?: Record<string, string | number>
) {
  // Google Analytics 4
  if (typeof window !== "undefined" && window.gtag) {
    window.gtag("event", name, params);
  }

  // PostHog
  if (typeof window !== "undefined" && window.posthog) {
    window.posthog.capture(name, params);
  }

  // Plausible
  if (typeof window !== "undefined" && window.plausible) {
    window.plausible(name, { props: params });
  }
}
```

---

## 3 — Dynamic Content Fetching

### When to Fetch

| Data Type       | Strategy                      | Example              |
| --------------- | ----------------------------- | -------------------- |
| **Pricing**     | ISR (revalidate every 1 hr)   | `fetch(url, { next: { revalidate: 3600 } })` |
| **Testimonials**| Static at build + ISR         | CMS integration      |
| **Stats**       | Client fetch + animated count | Live user count      |
| **Blog posts**  | RSC (Server Component)        | Latest 3 posts       |

### Pattern: ISR for Semi-Dynamic Data

```tsx
// app/api/stats/route.ts
export const revalidate = 3600; // 1 hour

export async function GET() {
  const stats = await fetchStatsFromDB();
  return Response.json(stats);
}
```

### Pattern: Client Fetch with Fallback

For data that should never block the initial paint:

```tsx
function StatsDisplay({ fallback }: { fallback: Stats }) {
  const [stats, setStats] = useState<Stats>(fallback);

  useEffect(() => {
    fetch("/api/stats")
      .then((r) => r.json())
      .then(setStats)
      .catch(() => {/* keep fallback */});
  }, []);

  return (
    <AnimatedCounter value={stats.users} suffix="+" />
  );
}
```

### Pattern: Suspense for Server Data

```tsx
// In a Server Component scene wrapper
import { Suspense } from "react";
import { StatsSkeleton } from "./skeletons";

export default function StatsSceneWrapper() {
  return (
    <Suspense fallback={<StatsSkeleton />}>
      <StatsScene />
    </Suspense>
  );
}

async function StatsScene() {
  const data = await fetch("https://api.example.com/stats", {
    next: { revalidate: 3600 },
  });
  const stats = await data.json();

  return <StatsPage stats={stats} />;
}
```

---

## 4 — Environment Variables & Config

```env
# .env.local (never committed)
API_URL=https://api.example.com
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX
NEXT_PUBLIC_POSTHOG_KEY=phc_xxxx

# .env.example (committed — shows required vars)
API_URL=
NEXT_PUBLIC_GA_ID=
NEXT_PUBLIC_POSTHOG_KEY=
```

| Prefix           | Accessible Where | Use For               |
| ---------------- | ---------------- | --------------------- |
| `NEXT_PUBLIC_`   | Client + Server  | Analytics IDs, public keys |
| _(no prefix)_    | Server only      | API secrets, DB URLs  |

### Loading Analytics Scripts

```tsx
// app/layout.tsx
import Script from "next/script";

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        {process.env.NEXT_PUBLIC_GA_ID && (
          <>
            <Script
              src={`https://www.googletagmanager.com/gtag/js?id=${process.env.NEXT_PUBLIC_GA_ID}`}
              strategy="afterInteractive"
            />
            <Script id="ga-init" strategy="afterInteractive">
              {`
                window.dataLayer = window.dataLayer || [];
                function gtag(){dataLayer.push(arguments);}
                gtag('js', new Date());
                gtag('config', '${process.env.NEXT_PUBLIC_GA_ID}');
              `}
            </Script>
          </>
        )}
      </body>
    </html>
  );
}
```

---

## 5 — External Navigation & Deep Linking

Pass a navigation callback from the page orchestrator so external
components (navbar, footer, CTAs) can jump to specific scenes:

```tsx
// In the main orchestrator
const goToPage = useCallback((index: number) => { /* … */ }, []);

// Expose to navbar, footer, etc.
<Navbar onNavigate={goToPage} />
<Footer onNavigate={goToPage} />
```

### Hash-Based Deep Links

```tsx
// app/page.tsx
"use client";

import { useSearchParams } from "next/navigation";

const HASH_MAP: Record<string, number> = {
  "#signup": 9,
  "#demo": 8,
  "#faq": 10,
  "#pricing": 7,
};

export default function HomePage() {
  const hash = typeof window !== "undefined" ? window.location.hash : "";
  const initialPage = HASH_MAP[hash] ?? 0;

  return <StoryJourney initialPage={initialPage} />;
}
```

### Programmatic Navigation from Anywhere

```tsx
// Use window.location.hash for simple page jumps
function CTAButton({ targetScene }: { targetScene: string }) {
  return (
    <button
      onClick={() => {
        window.location.hash = targetScene;
        window.location.reload(); // or use router
      }}
      className="btn-glow"
    >
      Learn More
    </button>
  );
}
```

---

## Integration Checklist

- [ ] Forms are inside scrollable embed scenes
- [ ] Form focus disables page-level scroll/swipe navigation
- [ ] Success/error states animate with framer-motion
- [ ] Analytics fire on scene transitions, not on every render
- [ ] Furthest-scene metric tracks engagement funnel
- [ ] CTA clicks are tracked with descriptive event names
- [ ] Dynamic data has static fallbacks so page never shows empty
- [ ] Environment variables use correct `NEXT_PUBLIC_` prefix
- [ ] Analytics scripts load with `strategy="afterInteractive"`
- [ ] Deep links map URL hashes to scene indices

---

**Next:** [Bundling Frontend →](../bundling-frontend/SKILL.md)
