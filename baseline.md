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

---
---

## Networking Baseline

**Tool:** Chrome DevTools → Network tab

### Overall Numbers

| | Cold Load (cache disabled) | Soft Refresh (cache enabled) |
|---|---|---|
| Requests | 214 | 217 |
| Data Transferred | 10.6 MB | 2.6 MB |
| Total Resource Size | 22.9 MB | 24.4 MB |
| Load time | 11.79 s | 10.97 s |

- **Compression reduction (cold load):** 22.9 MB → 10.6 MB ≈ **53.7%**
- **Caching reduction:** 10.6 MB → 2.6 MB ≈ **75.5%** in bytes, but request count did not drop (214 → 217) — nearly every request still fires on repeat visits.

### Breakdown by Resource Type (approximate, filtered views)

| Type | Requests | Resource Size | Share of Total |
|---|---|---|---|
| JS | 87 | ~13.2 MB | ~54% |
| Images | 59 | ~6.9 MB | ~28% |
| CSS | 7 | ~1.1 MB | ~4% |
| Other (fonts/XHR/ads/tracking) | ~70 | ~3.2 MB | ~13% |

JS alone is over half the page's total weight. Request names reveal a heavy programmatic-advertising and tag-management stack (`executor.js`, `dom.js`, GPT ad calls, `collect?v=2` tracking pixels, GDPR/consent scripts) contributing significantly to both JS weight and total request count — not the site's own core content.

---

### Finding 5: Extremely high request count (214–217) driven by ad/tracking sprawl

**Metric(s) affected:** Number of Requests — a major contributor to the already-poor load times given per-request latency overhead.

**How it affects users:** Over 200 separate network round-trips for a single page load compounds badly on any connection with real latency, adding delay on top of the already catastrophic LCP/TBT findings from the Core Web Vitals baseline.

**Cause (likely):** Request names show a heavy programmatic-advertising and tag-management stack (`executor.js`, `dom.js`, GPT ad calls, `collect?v=2` tracking pixels, GDPR/consent scripts) — a large news site typically runs dozens of ad vendors and analytics tools simultaneously, each firing its own requests independently.

**Solution (likely):** Audit and consolidate the ad/tracking vendor list, lazy-load ads that are below the fold, and batch or debounce tracking/analytics calls instead of firing them individually.

---

### Finding 6: JavaScript is over half the page's total weight (~13.2 MB of ~24.4 MB)

**Metric(s) affected:** Total Byte Weight (JS) — directly tying into the catastrophic LCP (41.6s) and TBT (up to 5,350ms) from the Core Web Vitals baseline.

**How it affects users:** This much JS must be downloaded, parsed, and executed before the page is usable — explaining why users see a blank or unresponsive page for so long.

**Cause (likely):** Unlike a typical single-bundle problem, the request chain here (`content.js` → `dom.js`/`js.js` → further scripts) suggests many separate third-party scripts stacking on top of each other rather than one oversized first-party bundle — each ad/analytics vendor loading its own dependencies.

**Solution (likely):** Aggressively audit and reduce the number of third-party scripts and ad vendors; lazy-load anything not required for the initial view; code-split first-party JS so only what's needed up front loads immediately.

---

### Finding 7: Caching reduces bytes but not request count — many requests are inherently non-cacheable

**Metric(s) affected:** Request count on repeat visits (214 → 217, effectively unchanged despite a 75% byte reduction).

**How it affects users:** Even returning visitors still pay the full round-trip/latency cost of ~215 requests every single visit, even though the bytes themselves are mostly cached — meaning caching alone won't meaningfully speed up repeat visits without also reducing request count.

**Cause (likely):** Many requests are inherently non-cacheable by nature — personalized ad creative, tracking pixels with unique per-load IDs, and "trending content" fetches that must query fresh data each time — so they can't benefit from standard browser caching regardless of headers.

**Solution (likely):** Reduce reliance on real-time/personalized fetches for above-the-fold content, consolidate multiple tracking calls into fewer batched requests, and ensure only genuinely time-sensitive data is fetched immediately while everything else loads lazily.

---

### Finding 8: Images are only partially cached, unlike CSS (fully cached)

**Metric(s) affected:** Cached Transfer Size for Images (~1.74 MB re-transferred out of ~6.9 MB total, vs. CSS which showed 0 kB re-transferred).

**How it affects users:** Repeat visitors still re-download a meaningful chunk of image weight that could otherwise be served instantly from cache, adding unnecessary bytes and time to what should be a fast repeat visit.

**Cause (likely):** Many images are served through a dynamic image-resizing/proxy service (URLs like `.../90/?url=https://assets.apnews.com/...`) where query parameters may vary by placement or crop, and ad creative images change per impression — both patterns defeat standard browser caching even when the underlying image is unchanged.

**Solution (likely):** Standardize image proxy URLs/query parameters so identical images share one cache key, apply long `Cache-Control` headers on the resizing service's static outputs, and lazy-load below-the-fold images to cut down on total image requests in the first place.
