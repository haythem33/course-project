# Performance Baseline — Course Project

**Site:** AP News — [https://apnews.com/](https://apnews.com/)
**Primary page tested:** [https://apnews.com/](https://apnews.com/) (Homepage)
**Conditions:** Lighthouse 13.4.0 — Mobile (emulated Moto G Power, Slow 4G throttling) and Desktop

---

## Metrics Summary

### PageSpeed Insights — Category Scores

| Category | Mobile | Desktop |
|---|---|---|
| Performance | 35 🔴 | 26 🔴 |
| Accessibility | 76 🟠 | 79 🟠 |
| Best Practices | 77 🟠 | 35 🔴 |
| SEO | 85 🟠 | 85 🟠 |

### Core Web Vitals / Rendering — Lab Data

| Metric | Mobile | Desktop |
|---|---|---|
| First Contentful Paint (FCP) | 7.0 s 🔴 | 2.2 s 🔴 |
| Largest Contentful Paint (LCP) | 41.6 s 🔴 | 15.6 s 🔴 |
| Total Blocking Time (TBT) | 3,050 ms 🔴 | 5,350 ms 🔴 |
| Cumulative Layout Shift (CLS) | 0.033 🟢 | 0.064 🟢 |
| Speed Index | 20.4 s 🔴 | 13.2 s 🔴 |

### Core Web Vitals — Field Data (real users, 28-day period)

| Metric | Mobile — This URL | Mobile — Origin | Desktop — This URL | Desktop — Origin |
|---|---|---|---|---|
| LCP | 2.9 s 🟠 | 3.0 s 🟠 | 3.7 s 🔴 | 2.7 s 🟠 |
| INP | 185 ms 🟢 | 239 ms 🟠 | 160 ms 🟢 | 161 ms 🟢 |
| CLS | 0.05 🟢 | 0.06 🟢 | 0.02 🟢 | 0.06 🟢 |
| FCP | 2.1 s 🟠 | 2.4 s 🟠 | 2.1 s 🟠 | 2.1 s 🟠 |
| TTFB | 0.2 s 🟢 | 0.4 s 🟢 | 0.2 s 🟢 | 0.2 s 🟢 |
| **Overall** | **Failed** 🔴 | **Failed** 🔴 | **Failed** 🔴 | **Failed** 🔴 |

*Note the large gap between lab and field data here: lab LCP is 41.6s (mobile) while field LCP is only 2.9s. This is common — field data is an aggregate of real users who may bail out, have ad/content blockers, or land on a cached/partially-loaded state that the CrUX sample doesn't fully capture the same way a full unthrottled Lighthouse run does. Both are valid signals, but the lab run exposes a worst-case scenario the aggregate field data smooths over.*

---

## Findings

### Finding 1: Largest Contentful Paint is catastrophic — 41.6s mobile / 15.6s desktop

**Metric(s) affected:** LCP — by a huge margin the worst number in this baseline.

**How it affects users:** Users would abandon the page well before the main content ever appears. 41.6 seconds isn't "slow" — it's a broken experience for anyone on a real mobile connection.

**Cause (likely):** The captured page state (visible in the mobile screenshot) shows a cookie/privacy consent overlay ("Manage Your Privacy...") still displayed at capture time. Large news publishers typically run heavy third-party Consent Management Platforms (CMPs) with their own JS/iframes for GDPR/CCPA compliance — these often block or delay the actual page content from becoming the measured LCP element, and can themselves take a very long time to fully resolve.

**Solution (likely):** Audit the CMP/consent banner implementation — load it asynchronously without blocking the render of the actual page content, use a lighter-weight consent solution, and ensure the real content (not the overlay) is prioritized for early paint.

---

### Finding 2: Total Blocking Time is severe on both, and worse on Desktop — 5,350ms

**Metric(s) affected:** TBT (both mobile and desktop), and by extension real-world responsiveness (INP).

**How it affects users:** For over 5 seconds on desktop (3 seconds on mobile), the page is completely unresponsive to clicks or taps — a user trying to scroll past the consent banner or click a headline would see no reaction at all.

**Cause (likely):** A large number of third-party scripts (ad tech, analytics, tracking pixels, the CMP itself) executing synchronously on the main thread. The fact that desktop TBT is worse than mobile TBT is unusual and suggests these scripts are CPU-heavy regardless of device — a network/device constraint wouldn't explain a desktop-worse result, but heavy JS execution volume would.

**Solution (likely):** Defer/async-load third-party scripts so they don't block the main thread during initial load; audit the ad-tech and analytics stack for redundant or unnecessary scripts; break up long tasks so input isn't blocked for seconds at a time.

---

### Finding 3: Speed Index is very high — 20.4s mobile / 13.2s desktop

**Metric(s) affected:** Speed Index (how quickly the page visually fills in, distinct from first/largest paint).

**How it affects users:** The page isn't just slow to show something — it stays visually incomplete for a very long stretch, meaning users see a broken or half-loaded layout for many seconds even after initial content appears.

**Cause (likely):** Same root contributors as Findings 1 and 2 — render-blocking consent/ad infrastructure and heavy JS execution delay visual completeness throughout the load, not just at one single point.

**Solution (likely):** Prioritize critical above-the-fold content and defer everything else (images below the fold, non-critical ads, secondary widgets) so the visible page fills in quickly even while background scripts are still working.

---

### Finding 4: First Contentful Paint is slow — 7.0s mobile / 2.2s desktop

**Metric(s) affected:** FCP.

**How it affects users:** Users see a completely blank screen for 7 seconds on mobile before anything appears at all — a long wait with zero feedback that the page is even working.

**Cause (likely):** Render-blocking CSS/JS — likely including the consent banner's own stylesheet/script — loading before the browser can paint anything.

**Solution (likely):** Inline critical CSS for above-the-fold content, defer non-critical JS and stylesheets, and ensure the consent banner doesn't block the render path for the rest of the page.
