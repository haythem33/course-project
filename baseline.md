# Performance Baseline — Course Project

**Site:** AP News — [https://apnews.com/](https://apnews.com/)
**Primary page tested:** [https://apnews.com/](https://apnews.com/) (Homepage)
**Conditions:** Lighthouse 13.4.0 — Mobile (emulated Moto G Power, Slow 4G throttling) and Desktop (lab data); Chrome DevTools Network tab, no throttling (networking data)

---

## Metrics Summary

### PageSpeed Insights — Category Scores

| Category | Mobile | Desktop |
|---|---|---|
| Performance | 35 🔴 | 26 🔴 |
| Accessibility | 76 🟠 | 79 🟠 |
| Best Practices | 77 🟠 | **35 🔴** |
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

*Note the large lab-vs-field gap: lab LCP is 41.6s (mobile) while field LCP is only 2.9s. Field data is a real-user aggregate and may reflect abandoned loads, ad/content blockers, or cached states the CrUX sample handles differently than a full, unthrottled Lighthouse run. Both signals are valid — the lab run exposes a worst-case scenario the field aggregate smooths over.*

### Networking

| | Cold Load (cache disabled) | Soft Refresh (cache enabled) |
|---|---|---|
| Requests | 214 | 217 |
| Data Transferred | 10.6 MB | 2.6 MB |
| Total Resource Size | 22.9 MB | 24.4 MB |
| Load Time | 11.79 s | 10.97 s |

- **Compression reduction (cold load):** 22.9 MB → 10.6 MB ≈ **53.7%**
- **Caching reduction:** 10.6 MB → 2.6 MB ≈ **75.5%** in bytes, but request count did not drop (214 → 217).

**Breakdown by resource type (approximate, filtered views):**

| Type | Requests | Resource Size | Share of Total |
|---|---|---|---|
| JS | 87 | ~13.2 MB | ~54% |
| Images | 59 | ~6.9 MB | ~28% |
| CSS | 7 | ~1.1 MB | ~4% |
| Other (fonts/XHR/ads/tracking) | ~70 | ~3.2 MB | ~13% |

**Additional relevant metric — Third-Party Request Share:** the majority of both JS weight and total request count trace back to third-party ad/tag-management infrastructure (`executor.js`, `dom.js`, GPT ad calls, `collect?v=2` tracking pixels) rather than first-party site code — worth tracking separately since it points to a vendor-management fix rather than a pure engineering one.

---

## Findings

Each finding below represents one distinct, independently-observable root cause. Metrics that share a root cause (e.g. FCP, LCP, and Speed Index all delayed by the same render-blocking consent/ad infrastructure) are grouped as a single finding rather than listed separately.

### Corrective Finding 1: Render-blocking consent/ad infrastructure delays first paint, largest paint, and visual completeness

**Metric(s) affected:** First Contentful Paint (7.0s mobile / 2.2s desktop), Largest Contentful Paint (41.6s mobile / 15.6s desktop), Speed Index (20.4s mobile / 13.2s desktop)

**How it affects users:** Users see a blank screen for up to 7 seconds, then wait as long as 41.6 seconds before the actual page content appears — and the page stays visually incomplete well beyond that. This isn't "slow," it reads as broken, and the vast majority of users will abandon the page long before it's usable.

**Cause (likely):** The captured page state shows a cookie/privacy consent overlay still displayed at capture time. Large news publishers typically run heavy third-party Consent Management Platforms (CMPs) with their own JS/iframes, which can block or delay the real content from becoming the measured paint target and take a long time to resolve themselves.

**Solution (likely):** Load the CMP/consent banner asynchronously without blocking the render of the underlying page content; inline critical above-the-fold CSS; defer non-critical JS/stylesheets so the real content — not the overlay — is prioritized for early paint.

---

### Corrective Finding 2: Heavy main-thread JS execution blocks interaction

**Metric(s) affected:** Total Blocking Time (3,050 ms mobile / 5,350 ms desktop)

**How it affects users:** For 3–5+ seconds, the page is completely unresponsive to clicks or taps — a user trying to dismiss the consent banner or click a headline sees no reaction at all.

**Cause (likely):** A large volume of third-party scripts (ad tech, analytics, tracking pixels, the CMP itself) executing synchronously on the main thread. Desktop TBT being *worse* than mobile is notable — a network/device constraint wouldn't explain that, but a high volume of CPU-heavy script execution regardless of device would.

**Solution (likely):** Defer/async-load third-party scripts so they don't block the main thread during initial load; audit the ad-tech and analytics stack for redundant scripts; break up long tasks so input isn't blocked for seconds at a time.

---

### Corrective Finding 3: High request count driven by ad/tracking sprawl

**Metric(s) affected:** Number of Requests (214–217)

**How it affects users:** Over 200 separate network round-trips per load compounds badly on any connection with real latency, adding delay on top of the already severe LCP/TBT problems.

**Cause (likely):** A heavy programmatic-advertising and tag-management stack — each ad vendor, tracker, and consent script firing its own independent requests rather than being consolidated.

**Solution (likely):** Audit and consolidate the ad/tracking vendor list, lazy-load below-the-fold ads, and batch or debounce tracking/analytics calls instead of firing them individually.

---

### Corrective Finding 4: JavaScript is over half the page's total weight (~13.2 MB of ~24.4 MB)

**Metric(s) affected:** Total Byte Weight (JS), ~54% of total resource size

**How it affects users:** This much JS must be downloaded, parsed, and executed before the page is usable, directly compounding the paint and interactivity delays already measured.

**Cause (likely):** Unlike a single oversized first-party bundle, the request chain (`content.js` → `dom.js`/`js.js` → further scripts) suggests many separate third-party scripts stacking on top of each other, each ad/analytics vendor loading its own dependencies.

**Solution (likely):** Aggressively audit and reduce the number of third-party scripts and ad vendors; lazy-load anything not required for the initial view; code-split first-party JS so only what's needed loads immediately.

---

### Corrective Finding 5: Caching reduces bytes but not request count

**Metric(s) affected:** Request count on repeat visits (214 → 217, effectively unchanged despite a 75% byte reduction)

**How it affects users:** Returning visitors still pay the full round-trip/latency cost of ~215 requests every visit, even though most bytes are cached — meaning caching alone won't meaningfully speed up repeat visits without also reducing request count.

**Cause (likely):** Many requests are inherently non-cacheable — personalized ad creative, tracking pixels with unique per-load IDs, and real-time "trending content" fetches that must query fresh data on every load.

**Solution (likely):** Reduce reliance on real-time/personalized fetches for above-the-fold content, consolidate multiple tracking calls into fewer batched requests, and ensure only genuinely time-sensitive data is fetched immediately.

---

### Corrective Finding 6: Images are only partially cached, unlike CSS

**Metric(s) affected:** Cached Transfer Size for Images (~1.74 MB re-transferred out of ~6.9 MB total on repeat visit)

**How it affects users:** Repeat visitors re-download a meaningful chunk of image weight that could otherwise be served instantly from cache, adding unnecessary bytes and time to what should be a fast repeat visit.

**Cause (likely):** Many images are served through a dynamic image-resizing/proxy service with query parameters that may vary by placement or crop, and ad creative changes per impression — both patterns defeat standard browser caching even when the underlying image is unchanged.

**Solution (likely):** Standardize image proxy URLs/query parameters so identical images share one cache key, apply long `Cache-Control` headers on the resizing service's static outputs, and lazy-load below-the-fold images to reduce total image requests.

---

### Corrective Finding 7: Best Practices score anomaly — 35 (Desktop) vs. 77 (Mobile)

**Metric(s) affected:** Best Practices score (Desktop 35 🔴 vs. Mobile 77 🟠 — a 42-point gap)

**How it affects users:** A large, device-specific gap in Best Practices typically signals something concrete going wrong only in one rendering path — such as console errors, deprecated API usage, insecure requests, or image-serving issues — that could carry real risk (security, deprecated-API breakage) even if it isn't directly a speed issue.

**Cause (likely):** Not yet confirmed — the specific audit items behind this desktop-only drop haven't been captured in this baseline. Likely candidates include desktop-specific third-party scripts using deprecated APIs, mixed-content/insecure requests, or image aspect-ratio issues that only manifest at desktop viewport sizes.

**Solution (likely):** Open the detailed Best Practices audit list on the Desktop PSI report and identify the specific failing checks — this is a case where the "likely cause" genuinely needs the diagnostic detail before a fix can be scoped.

---

### Good Finding 1: Cumulative Layout Shift is well-controlled on both platforms

**Metric(s) affected:** CLS — Mobile 0.033 🟢, Desktop 0.064 🟢 (both "good," despite every other rendering metric failing)

**Why this is good:** Even with a heavy ad/tracking stack that could easily push content around as it loads, the page maintains visual stability. This suggests the team has already handled image/ad-slot sizing reasonably well — a foundation worth preserving while fixing the other issues.

---

### Good Finding 2: CSS caches perfectly on repeat visits

**Metric(s) affected:** Cached Transfer Size for CSS (0 kB re-transferred out of ~3.6 MB total on repeat visit)

**Why this is good:** Unlike JS and images, CSS is served entirely from cache on a soft refresh — meaning the site's caching strategy works well for this asset type. This is a solid, working example to model the JS/image caching fixes (Corrective Findings 5–6) after.

---

### Good Finding 3: Real-user interactivity (INP) is largely passing despite catastrophic lab TBT

**Metric(s) affected:** Interaction to Next Paint — Mobile 185 ms 🟢 (This URL), Desktop 160–161 ms 🟢 (both platforms)

**Why this is good:** Despite a lab-measured Total Blocking Time as high as 5,350ms, real users' actual interaction responsiveness (INP) is measured as good on both platforms. This suggests that whatever's blocking the main thread during load isn't necessarily blocking users' first meaningful interactions once they do engage — a meaningfully less severe real-world picture than the lab TBT number alone would suggest, even though the overall Core Web Vitals assessment still fails due to LCP.

---

*Next steps: Corrective Findings 1 and 2 remain the highest-impact fixes (they touch the most metrics and represent the most severe measured degradation), with Finding 7 flagged as needing further diagnostic investigation before it can be scoped.*
