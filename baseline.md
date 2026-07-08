# Performance Baseline — Course Project

**Site:** AP News — [https://apnews.com/](https://apnews.com/)
**Primary page tested:** [https://apnews.com/](https://apnews.com/) (Homepage)
**Conditions:** **Mobile** — Lighthouse 13.4.0, emulated **Moto G Power**, **Slow 4G network throttling** with 4x CPU slowdown (lab data), compared against Desktop (no throttling); Chrome DevTools Network tab, no throttling (networking data). All metrics below are labeled Mobile or Desktop explicitly.

---

## Metrics Summary

### PageSpeed Insights — Category Scores (Mobile vs. Desktop)

| Category | Mobile | Desktop |
|---|---|---|
| Performance | 35 🔴 | 26 🔴 |
| Accessibility | 76 🟠 | 79 🟠 |
| Best Practices | 77 🟠 | **35 🔴** |
| SEO | 85 🟠 | 85 🟠 |

### Core Web Vitals / Rendering — Lab Data (Mobile: Slow 4G + 4x CPU throttling vs. Desktop: untethered)

| Metric | Mobile | Desktop |
|---|---|---|
| First Contentful Paint (FCP) | 7.0 s 🔴 | 2.2 s 🔴 |
| Largest Contentful Paint (LCP) | 41.6 s 🔴 | 15.6 s 🔴 |
| Total Blocking Time (TBT) | 3,050 ms 🔴 | 5,350 ms 🔴 |
| Cumulative Layout Shift (CLS) | 0.033 🟢 | 0.064 🟢 |
| Speed Index | 20.4 s 🔴 | 13.2 s 🔴 |

### Core Web Vitals — Field Data (Mobile vs. Desktop, real users, 28-day period)

| Metric | Mobile — This URL | Mobile — Origin | Desktop — This URL | Desktop — Origin |
|---|---|---|---|---|
| LCP | 2.9 s 🟠 | 3.0 s 🟠 | 3.7 s 🔴 | 2.7 s 🟠 |
| INP | 185 ms 🟢 | 239 ms 🟠 | 160 ms 🟢 | 161 ms 🟢 |
| CLS | 0.05 🟢 | 0.06 🟢 | 0.02 🟢 | 0.06 🟢 |
| FCP | 2.1 s 🟠 | 2.4 s 🟠 | 2.1 s 🟠 | 2.1 s 🟠 |
| TTFB | 0.2 s 🟢 | 0.4 s 🟢 | 0.2 s 🟢 | 0.2 s 🟢 |
| **Overall** | **Failed** 🔴 | **Failed** 🔴 | **Failed** 🔴 | **Failed** 🔴 |

*Note the large lab-vs-field gap: lab LCP is 41.6s (mobile) while field LCP is only 2.9s. Field data is a real-user aggregate and may reflect abandoned loads, ad/content blockers, or cached states the CrUX sample handles differently than a full, unthrottled Lighthouse run. Both signals are valid — the lab run exposes a worst-case scenario the field aggregate smooths over.*

### Networking (captured with DevTools set to "No throttling")

> Note: these captures were taken without network/CPU throttling enabled, so they aren't directly comparable to the Mobile lab profile above. Re-run with Network throttling set to **Slow 4G** and CPU set to **4x slowdown** (or device emulation set to Moto G Power) in DevTools if you want networking numbers that match the PSI mobile conditions.

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

## Prioritization System

This project uses a different scoring model than the personal project's SRC/E score — the **Weighted Priority Index (WPI)**, an additive weighted-average model rather than a multiplicative ratio.

Each corrective finding is rated on four dimensions, each on a **1–10 scale**, then combined using fixed percentage weights:

| Dimension | Question it answers | Weight |
|---|---|---|
| **User Impact (UI)** | How directly does this hurt the end-user's ability to use the page (frustration, abandonment, inability to complete a task)? | 35% |
| **Metric Severity (MS)** | How far from "good" is the raw measured number (e.g. how deep into the red zone)? | 25% |
| **Traffic Reach (TR)** | What share of page views/visits does this affect? | 20% |
| **Ease of Implementation (EI)** | How easy is the fix likely to be? (10 = trivial config change, 1 = major structural/vendor overhaul) — scored as *ease*, not effort, so higher is always better. | 20% |

**Formula:**

```
WPI = (0.35 × User Impact) + (0.25 × Metric Severity) + (0.20 × Traffic Reach) + (0.20 × Ease of Implementation)
```

Result is on a 1–10 scale; higher = higher priority. Unlike the personal project's SRC/E score (which multiplies Severity × Reach × Confidence and divides by Effort — a ratio that can swing dramatically based on the divisor), WPI is an additive weighted average: every dimension contributes proportionally according to its fixed weight, so no single low score can collapse the whole result to near-zero the way dividing by a high Effort value can. This tends to keep findings with strong user-impact evidence ranked highly even when they're also hard to fix, rather than letting implementation difficulty dominate the ranking.

---

## Findings

Each finding below represents one distinct, independently-observable root cause. Metrics that share a root cause (e.g. FCP, LCP, and Speed Index all delayed by the same render-blocking consent/ad infrastructure) are grouped as a single finding rather than listed separately.

### Corrective Finding 1: Render-blocking consent/ad infrastructure delays first paint, largest paint, and visual completeness

**Metric(s) affected:** First Contentful Paint (7.0s mobile / 2.2s desktop), Largest Contentful Paint (41.6s mobile / 15.6s desktop), Speed Index (20.4s mobile / 13.2s desktop)

**How it affects users:** Users see a blank screen for up to 7 seconds, then wait as long as 41.6 seconds before the actual page content appears — and the page stays visually incomplete well beyond that. This isn't "slow," it reads as broken, and the vast majority of users will abandon the page long before it's usable.

**Cause (likely):** The captured page state shows a cookie/privacy consent overlay still displayed at capture time. Large news publishers typically run heavy third-party Consent Management Platforms (CMPs) with their own JS/iframes, which can block or delay the real content from becoming the measured paint target and take a long time to resolve themselves.

**Solution (likely):** Load the CMP/consent banner asynchronously without blocking the render of the underlying page content; inline critical above-the-fold CSS; defer non-critical JS/stylesheets so the real content — not the overlay — is prioritized for early paint.

**Priority Rating:** User Impact 10 · Metric Severity 10 · Traffic Reach 10 · Ease of Implementation 3 → **WPI = 0.35(10)+0.25(10)+0.20(10)+0.20(3) = 8.6**

---

### Corrective Finding 2: Heavy main-thread JS execution blocks interaction

**Metric(s) affected:** Total Blocking Time (3,050 ms mobile / 5,350 ms desktop)

**How it affects users:** For 3–5+ seconds, the page is completely unresponsive to clicks or taps — a user trying to dismiss the consent banner or click a headline sees no reaction at all.

**Cause (likely):** A large volume of third-party scripts (ad tech, analytics, tracking pixels, the CMP itself) executing synchronously on the main thread. Desktop TBT being *worse* than mobile is notable — a network/device constraint wouldn't explain that, but a high volume of CPU-heavy script execution regardless of device would.

**Solution (likely):** Defer/async-load third-party scripts so they don't block the main thread during initial load; audit the ad-tech and analytics stack for redundant scripts; break up long tasks so input isn't blocked for seconds at a time.

**Priority Rating:** User Impact 8 · Metric Severity 9 · Traffic Reach 10 · Ease of Implementation 4 → **WPI = 0.35(8)+0.25(9)+0.20(10)+0.20(4) = 7.85**

---

### Corrective Finding 3: High request count driven by ad/tracking sprawl

**Metric(s) affected:** Number of Requests (214–217)

**How it affects users:** Over 200 separate network round-trips per load compounds badly on any connection with real latency, adding delay on top of the already severe LCP/TBT problems.

**Cause (likely):** A heavy programmatic-advertising and tag-management stack — each ad vendor, tracker, and consent script firing its own independent requests rather than being consolidated.

**Solution (likely):** Audit and consolidate the ad/tracking vendor list, lazy-load below-the-fold ads, and batch or debounce tracking/analytics calls instead of firing them individually.

**Priority Rating:** User Impact 6 · Metric Severity 6 · Traffic Reach 10 · Ease of Implementation 4 → **WPI = 0.35(6)+0.25(6)+0.20(10)+0.20(4) = 6.4**

---

### Corrective Finding 4: JavaScript is over half the page's total weight (~13.2 MB of ~24.4 MB)

**Metric(s) affected:** Total Byte Weight (JS), ~54% of total resource size

**How it affects users:** This much JS must be downloaded, parsed, and executed before the page is usable, directly compounding the paint and interactivity delays already measured.

**Cause (likely):** Unlike a single oversized first-party bundle, the request chain (`content.js` → `dom.js`/`js.js` → further scripts) suggests many separate third-party scripts stacking on top of each other, each ad/analytics vendor loading its own dependencies.

**Solution (likely):** Aggressively audit and reduce the number of third-party scripts and ad vendors; lazy-load anything not required for the initial view; code-split first-party JS so only what's needed loads immediately.

**Priority Rating:** User Impact 8 · Metric Severity 8 · Traffic Reach 10 · Ease of Implementation 3 → **WPI = 0.35(8)+0.25(8)+0.20(10)+0.20(3) = 7.4**

---

### Corrective Finding 5: Caching reduces bytes but not request count

**Metric(s) affected:** Request count on repeat visits (214 → 217, effectively unchanged despite a 75% byte reduction)

**How it affects users:** Returning visitors still pay the full round-trip/latency cost of ~215 requests every visit, even though most bytes are cached — meaning caching alone won't meaningfully speed up repeat visits without also reducing request count.

**Cause (likely):** Many requests are inherently non-cacheable — personalized ad creative, tracking pixels with unique per-load IDs, and real-time "trending content" fetches that must query fresh data on every load.

**Solution (likely):** Reduce reliance on real-time/personalized fetches for above-the-fold content, consolidate multiple tracking calls into fewer batched requests, and ensure only genuinely time-sensitive data is fetched immediately.

**Priority Rating:** User Impact 5 · Metric Severity 5 · Traffic Reach 8 · Ease of Implementation 5 → **WPI = 0.35(5)+0.25(5)+0.20(8)+0.20(5) = 5.6**

---

### Corrective Finding 6: Images are only partially cached, unlike CSS

**Metric(s) affected:** Cached Transfer Size for Images (~1.74 MB re-transferred out of ~6.9 MB total on repeat visit)

**How it affects users:** Repeat visitors re-download a meaningful chunk of image weight that could otherwise be served instantly from cache, adding unnecessary bytes and time to what should be a fast repeat visit.

**Cause (likely):** Many images are served through a dynamic image-resizing/proxy service with query parameters that may vary by placement or crop, and ad creative changes per impression — both patterns defeat standard browser caching even when the underlying image is unchanged.

**Solution (likely):** Standardize image proxy URLs/query parameters so identical images share one cache key, apply long `Cache-Control` headers on the resizing service's static outputs, and lazy-load below-the-fold images to reduce total image requests.

**Priority Rating:** User Impact 4 · Metric Severity 4 · Traffic Reach 7 · Ease of Implementation 6 → **WPI = 0.35(4)+0.25(4)+0.20(7)+0.20(6) = 5.0**

---

### Corrective Finding 7: Best Practices score anomaly — 35 (Desktop) vs. 77 (Mobile)

**Metric(s) affected:** Best Practices score (Desktop 35 🔴 vs. Mobile 77 🟠 — a 42-point gap)

**How it affects users:** A large, device-specific gap in Best Practices typically signals something concrete going wrong only in one rendering path — such as console errors, deprecated API usage, insecure requests, or image-serving issues — that could carry real risk (security, deprecated-API breakage) even if it isn't directly a speed issue.

**Cause (likely):** Not yet confirmed — the specific audit items behind this desktop-only drop haven't been captured in this baseline. Likely candidates include desktop-specific third-party scripts using deprecated APIs, mixed-content/insecure requests, or image aspect-ratio issues that only manifest at desktop viewport sizes.

**Solution (likely):** Open the detailed Best Practices audit list on the Desktop PSI report and identify the specific failing checks — this is a case where the "likely cause" genuinely needs the diagnostic detail before a fix can be scoped.

**Priority Rating:** User Impact 3 · Metric Severity 7 · Traffic Reach 6 · Ease of Implementation 7 → **WPI = 0.35(3)+0.25(7)+0.20(6)+0.20(7) = 5.4**

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

## Mobile-Specific Findings

These findings are specifically about the *gap* between mobile and desktop, not just mobile numbers in isolation.

### Corrective Finding 8: Network + CPU throttling amplify the consent/ad-blocking delay far more on mobile than desktop

**Metric(s) affected:** Largest Contentful Paint (41.6s mobile vs. 15.6s desktop — mobile is ~2.7x worse)

**How it affects users:** While both platforms fail LCP badly, mobile users face a dramatically worse version of the same underlying problem (Corrective Finding 1) — the gap isn't small, it's the difference between "very slow" and "essentially non-functional."

**Cause (likely):** The same consent-banner/ad-loading payload is served to both platforms, but mobile testing applies Slow 4G network throttling and 4x CPU slowdown together — meaning the identical fetch-and-execute work for the CMP and ad scripts takes dramatically longer on constrained mobile conditions, compounding rather than just adding to the desktop delay.

**Solution (likely):** Beyond the general fixes in Finding 1, consider serving a lighter-weight consent/ad experience specifically to mobile or low-bandwidth connections (e.g. via `navigator.connection`) — smaller ad creative sizes, fewer simultaneous third-party requests, and more aggressive deferral of non-critical scripts on constrained networks.

**Priority Rating:** User Impact 9 · Metric Severity 9 · Traffic Reach 9 · Ease of Implementation 3 → **WPI = 0.35(9)+0.25(9)+0.20(9)+0.20(3) = 7.8**

### Finding 9: Total Blocking Time is unexpectedly *better* on mobile than desktop

**Metric(s) affected:** Total Blocking Time (3,050 ms mobile vs. 5,350 ms desktop)

**How it affects users:** This is a genuinely unusual pattern — normally mobile (CPU-throttled) performs worse than desktop on execution-bound metrics. Here it's reversed, suggesting the desktop test run may have encountered a heavier or different set of third-party scripts/ads than the mobile run (ad servers often serve different creative/scripts by viewport size), or timing variance between the two test runs.

**Why this matters as a separate finding:** It's tempting to assume mobile is uniformly worse across every metric, but this shows that's not true here — treating "mobile-specific" fixes as a blanket assumption could mean under-prioritizing a desktop-specific TBT problem. Worth re-running both tests to confirm this isn't a one-off measurement, since if confirmed, it points to a desktop-specific ad/script configuration issue distinct from the general JS-weight problem in Finding 4.

---

## Bundle Analysis

**JavaScript:** First-party code shows webpack-style hashed chunk names, indicating some route/vendor code-splitting exists for the app's own code. However, the majority of JS on the page comes from **33 separate, independently-loaded third-party vendors**, which isn't a coordinated bundling strategy so much as an uncoordinated collection of scripts from different origins. PSI estimates **2,650 KiB of unused JavaScript** total — notably, even the first-party bundle (`All.min...gz.js`, 234.7 KiB) is **70% unused (164.9 KiB)** on this page. On source maps: third-party vendors like Viafoura and OneSignal expose original source paths (`.../avatar.vue`, `.../SessionManager.ts`) in their unused-code reports, meaning they ship source maps — but the first-party AP News bundle shows no such attribution, suggesting it ships without source maps in production.

**CSS:** First-party styles are a **single monolithic file** (`All.min...gz.css`, 104.3 KiB) with **no route or component splitting** evident. On this page, **90% of it (93.4 KiB) is unused** — a stark number for a site-wide, one-size-fits-all stylesheet. Total unused CSS across the page is 148 KiB, including third-party widget styles (Optanon consent banner, JW Player controls, Riverdrop quiz, Viafoura comments).

**Images:** Confirmed from the networking pass that first-party images are served through a dynamic resize/proxy service (multiple sizes, modern formats) — this part is handled well. One notable exception: a third-party quiz widget (riverdrop.com) serves a **600 KiB JPEG directly**, appearing to bypass that optimization pipeline entirely.

**Third-Party Resources:** 33 distinct vendors are loaded — spanning ad tech (11+ vendors), analytics, consent management, personalization/CDP, social, and notifications. Combined, these vendors account for **~4.36 seconds of cumulative main-thread time**, which almost fully explains the TBT problem measured earlier in this baseline. Top individual offenders: Google Tag Manager (645ms), Quantcast (465ms), Google APIs/IMA video ads (363ms), Viafoura commenting widget (330ms), Optanon consent (310ms). Notably, `dianomi`'s `contextfeed.js` loads **twice** — an exact duplicate request.

---

### Corrective Finding 10: First-party JS bundle ships significant unused code (~70% waste)

**Metric(s) affected:** Unused JavaScript (164.9 KiB of 234.7 KiB transferred, first-party bundle)

**How it affects users:** Even setting aside the third-party sprawl, the site's own bundle is delivering roughly 2.3x more code than this specific page actually uses — extra parse/compile time and bytes downloaded for no benefit to this page's users.

**Cause (likely):** The bundle name (`All.min...`) suggests a shared, not-fully-scoped bundle serving multiple page types rather than being split per route or component.

**Solution (likely):** Introduce route-based code-splitting and tree-shaking for the first-party bundle so the homepage only loads the JS it actually needs, rather than a shared "all pages" bundle.

**Priority Rating:** User Impact 7 · Metric Severity 6 · Traffic Reach 10 · Ease of Implementation 6 → **WPI = 0.35(7)+0.25(6)+0.20(10)+0.20(6) = 7.15**

---

### Corrective Finding 11: First-party CSS is a single monolithic bundle, 90% unused on this page

**Metric(s) affected:** Unused CSS (93.4 KiB of 104.3 KiB transferred, first-party stylesheet)

**How it affects users:** Nearly the entire first-party stylesheet goes unused on this specific page — meaning users are downloading and having the browser parse a large stylesheet built for the whole site, not just what's needed here.

**Cause (likely):** CSS is bundled as a single site-wide file (`All.min...css`) with no route or component-level splitting.

**Solution (likely):** Split CSS by route/component, or introduce a critical-CSS extraction step so only the styles needed for the current page's above-the-fold content load immediately, with the rest deferred or split off entirely.

**Priority Rating:** User Impact 6 · Metric Severity 7 · Traffic Reach 10 · Ease of Implementation 7 → **WPI = 0.35(6)+0.25(7)+0.20(10)+0.20(7) = 7.25**

---

### Corrective Finding 12: Third-party script sprawl accounts for ~4.36s of cumulative main-thread time

**Metric(s) affected:** Third-party main-thread blocking time (~4,361 ms combined across 33 vendors) — directly explaining the previously-measured Total Blocking Time (3,050–5,350 ms)

**How it affects users:** This is the most directly quantified cause in the entire baseline: virtually all of the measured main-thread blocking traces back to third-party scripts, not first-party code. Users experience an unresponsive page almost entirely because of ad tech, analytics, and tag-management infrastructure stacked on top of each other.

**Cause (likely):** 33 separate vendors are loaded on a single page load, with heavy redundancy — at least 11 different ad-related vendors alone (GTM, GPT/Doubleclick, pub.network/Prebid, Wunderkind, dianomi, Connatix, Nativo, Amazon Ads, IAS, Quantcast, LongTail/JWPlayer ads) likely serving overlapping functions. One exact duplicate was also found: `dianomi`'s `contextfeed.js` loads twice.

**Solution (likely):** Conduct a vendor audit to consolidate or remove redundant ad/analytics tools (especially overlapping ad-tech vendors), fix the confirmed duplicate script load, and defer/lazy-load lower-priority vendors (e.g. social SDKs, non-critical personalization scripts) until after the initial page is interactive.

**Priority Rating:** User Impact 9 · Metric Severity 9 · Traffic Reach 10 · Ease of Implementation 4 → **WPI = 0.35(9)+0.25(9)+0.20(10)+0.20(4) = 8.2**

---

### Corrective Finding 13: First-party production bundles ship without source maps

**Metric(s) affected:** Debuggability/maintainability (not a direct performance metric, but a notable engineering-practice gap)

**How it affects users:** Not a direct user-facing performance issue, but it affects the team's ability to diagnose and fix the very problems in this report — without source maps, a minified production JS error is far harder to trace back to its original source.

**Cause (likely):** Source map generation may be disabled in the production build config, or maps are generated but not hosted/uploaded anywhere accessible to the team's error-tracking tooling.

**Solution (likely):** Enable source map generation in the build pipeline and upload maps privately to an error-tracking service (e.g. Sentry) rather than serving them publicly — this improves debugging without adding to the public payload size.

**Priority Rating:** User Impact 3 · Metric Severity 4 · Traffic Reach 10 · Ease of Implementation 8 → **WPI = 0.35(3)+0.25(4)+0.20(10)+0.20(8) = 5.65**

---

### Corrective Finding 14: A third-party embedded image bypasses the site's optimization pipeline

**Metric(s) affected:** Transfer Size for a single asset (600 KiB JPEG from the Riverdrop interactive quiz widget)

**How it affects users:** Users who load a page containing this embedded quiz widget download a single unoptimized 600 KiB image — disproportionately large compared to the resized/modern-format images used elsewhere on the site.

**Cause (likely):** The quiz widget is served by a third-party vendor (riverdrop.com) that delivers its own assets directly, bypassing AP News's own image-resizing/optimization proxy used for first-party content.

**Solution (likely):** Work with the vendor to confirm responsive/optimized image delivery is enabled on their end, or proxy their assets through AP's own image-optimization pipeline where contractually possible.

**Priority Rating:** User Impact 5 · Metric Severity 5 · Traffic Reach 4 · Ease of Implementation 7 → **WPI = 0.35(5)+0.25(5)+0.20(4)+0.20(7) = 5.2**



---

## Final Priority Ranking

Corrective findings ranked by Weighted Priority Index (highest = fix first):

| Rank | Finding | User Impact | Metric Severity | Traffic Reach | Ease | WPI |
|---|---|---|---|---|---|---|
| 1 | Finding 1 — Render-blocking consent/ad delays paint | 10 | 10 | 10 | 3 | **8.6** |
| 2 | Finding 12 — Third-party sprawl = ~4.36s main-thread time | 9 | 9 | 10 | 4 | **8.2** |
| 3 | Finding 2 — Main-thread JS execution blocks interaction | 8 | 9 | 10 | 4 | **7.85** |
| 4 | Finding 8 — Throttling amplifies mobile LCP gap | 9 | 9 | 9 | 3 | **7.8** |
| 5 | Finding 4 — JS is over half the page's weight | 8 | 8 | 10 | 3 | **7.4** |
| 6 | Finding 11 — CSS monolith, 90% unused on this page | 6 | 7 | 10 | 7 | **7.25** |
| 7 | Finding 10 — First-party JS bundle, 70% unused | 7 | 6 | 10 | 6 | **7.15** |
| 8 | Finding 3 — High request count from ad/tracking sprawl | 6 | 6 | 10 | 4 | **6.4** |
| 9 | Finding 13 — First-party bundles ship without source maps | 3 | 4 | 10 | 8 | **5.65** |
| 10 | Finding 5 — Caching reduces bytes but not request count | 5 | 5 | 8 | 5 | **5.6** |
| 11 | Finding 7 — Best Practices anomaly (mobile vs desktop) | 3 | 7 | 6 | 7 | **5.4** |
| 12 | Finding 14 — Third-party image bypasses optimization | 5 | 5 | 4 | 7 | **5.2** |
| 13 | Finding 6 — Images only partially cached | 4 | 4 | 7 | 6 | **5.0** |

**Takeaway:** the bundle-level analysis surfaced the single most concretely-evidenced finding in this entire baseline — Finding 12 (third-party script sprawl, ~4.36s of measured main-thread time) — which now ranks #2 overall, just behind the render-blocking consent/ad finding it's closely related to. Together, Findings 1, 12, 2, and 8 form a tightly connected top-4 cluster that all trace back to the same root problem: an enormous, uncoordinated third-party ad/tracking stack. The CSS and JS bundle-waste findings (10, 11) land solidly in the upper-middle of the ranking — real, fixable problems, but secondary to the third-party governance issue that dominates this baseline.

---

*Next steps: Findings 1, 12, 2, and 8 should be treated as a single connected initiative (third-party/ad-tech governance and render-blocking behavior) rather than four separate projects, since fixing the vendor stack will likely move all four simultaneously. Finding 7 remains flagged as needing further diagnostic investigation before it can be scored with full confidence.*
