# AP News Performance Audit — Implementer Report

**Site audited:** https://apnews.com/ (homepage)
**Audience:** Engineers implementing fixes

This document covers mechanism, reproduction steps, and effort for every finding. For business framing (cost/risk/priority for funding decisions), see the companion stakeholder report — same findings, different lens. Neither document is a summary of the other.

---

## How to Reproduce Everything in This Report

**Lab data (PageSpeed Insights):**
- Tool: [pagespeed.web.dev](https://pagespeed.web.dev), Lighthouse 13.4.0
- URL tested: `https://apnews.com/`
- Mobile profile: emulated **Moto G Power**, **Slow 4G** network throttling, **4x CPU slowdown**
- Desktop profile: no throttling
- Captured: Jul 7, 2026

**Field data (Core Web Vitals):**
- Source: Chrome UX Report (CrUX), pulled automatically by PageSpeed Insights, 28-day rolling window
- Both "This URL" (homepage specifically) and "Origin" (site-wide aggregate) shown where available

**Networking data:**
- Tool: Chrome DevTools → Network tab
- Captured with "No throttling" (see Finding 8 note below — this is a known gap, re-test with Slow 4G + 4x CPU if you need networking numbers directly comparable to the mobile PSI profile)
- Cold load: "Disable cache" checked, hard refresh (Ctrl+Shift+R)
- Warm load: "Disable cache" unchecked, normal refresh (Ctrl+R)

**Bundle/coverage data:**
- Chrome DevTools → More tools → Coverage, and PSI's "Reduce unused JavaScript" / "Reduce unused CSS" / "Reduce the impact of third-party code" diagnostic audits
- Rendering strategy verified via View Page Source (raw HTML, pre-JS) on the homepage only

**What we did NOT check** (flagging explicitly so it doesn't read as an oversight):
- **Offline/service worker support** — not evaluated. A news homepage doesn't have an obvious offline-use-case the way, say, a documentation site or PWA might; if there's a product reason to want offline reading, that's a separate scoping conversation, not something this audit found a problem with.
- **Article pages, live-blog pages, and other page types** — this entire audit is scoped to the homepage only. Rendering strategy, bundle contents, and third-party load specifically were confirmed via source inspection on the homepage; we have not verified whether article/live-blog pages share the same patterns. Given the site-wide nature of the CSS/JS bundles found, it's likely similar, but this is an assumption, not a verified finding.
- **Detailed accessibility audit** — we have the PSI accessibility category score (76 mobile / 79 desktop) but did not expand the individual accessibility audit items. Worth a follow-up pass if accessibility becomes a priority.
- **Performance flame charts, layer/compositing analysis, and animation jank** — these require live DevTools recording sessions on a real browser and were not completed as part of this baseline. Flagging this as an open item for a follow-up pass, not a "no findings" conclusion.

---

## Findings

Each finding includes: **Mechanism** (what's actually happening and why), **Reproduce** (how to see it yourself), **Fix**, **Effort**, and whether it's **Structural** (architectural/vendor-level, needs broader buy-in) or **Local** (contained code change, one team can ship it).

---

### Finding 1 — Render-blocking consent/ad infrastructure delays first and largest paint

**Metrics:** FCP 7.0s (mobile) / 2.2s (desktop), LCP 41.6s (mobile) / 15.6s (desktop), Speed Index 20.4s (mobile) / 13.2s (desktop)

**Mechanism:** At the moment Lighthouse captures the page, a cookie/consent overlay (OneTrust/Optanon) is still on screen. Consent Management Platforms typically inject their own JS/iframe stack early in the load sequence and can hold up the browser's ability to paint the real page content, either by direct render-blocking or by becoming the element competing for "largest contentful" status.

**Reproduce:** Run PSI mobile on `https://apnews.com/`, expand the "Metrics" section, and look at the screenshot thumbnail next to the LCP value — it shows the consent overlay still visible at capture time.

**Fix:** Ensure the CMP loads asynchronously and doesn't block the main document parse. Work with OneTrust/Optanon configuration (most CMPs have an async-load mode) rather than the default synchronous embed snippet.

**Effort:** Medium — configuration change, but likely needs coordination with whoever manages the CMP account (legal/privacy team dependency).

**Structural or Local:** Structural (cross-team dependency, vendor configuration).

---

### Finding 2 — Heavy main-thread JS execution blocks interaction

**Metrics:** TBT 3,050ms (mobile) / 5,350ms (desktop)

**Mechanism:** Total Blocking Time measures how long the main thread is occupied by long tasks (>50ms) after First Contentful Paint. See Finding 12 below — this is now well-explained by cumulative third-party script execution (~4.36s across 33 vendors), not first-party code.

**Reproduce:** PSI mobile/desktop → Metrics section → Total Blocking Time. For a breakdown by script, use DevTools Performance tab: record a page load, look at the "Bottom-Up" tab sorted by "Total Time," grouped by URL.

**Fix:** See Finding 12 (same root cause — this metric is the symptom, Finding 12 is the mechanism).

**Effort:** See Finding 12.

**Structural or Local:** Structural.

---

### Finding 3 — High request count driven by ad/tracking sprawl

**Metrics:** 214 requests (cold) / 217 (warm)

**Mechanism:** Each third-party vendor fires its own independent requests (tags, pixels, sub-resources) rather than sharing a consolidated delivery path. At ~200+ requests for a single page, connection/latency overhead compounds even before considering payload size.

**Reproduce:** DevTools Network tab, hard refresh with cache disabled, read the total request count in the bottom status bar.

**Fix:** See Finding 12 — vendor consolidation is the actual fix; this metric just quantifies one symptom of it.

**Effort:** See Finding 12.

**Structural or Local:** Structural.

---

### Finding 4 — JavaScript is over half the page's total weight (~13.2 MB of ~24.4 MB)

**Metrics:** ~54% of total resource size

**Mechanism:** Request-chain inspection (`content.js` → `dom.js`/`js.js` → further scripts) shows many separate third-party scripts stacking dependencies on top of each other, rather than one oversized first-party bundle. First-party code itself does show some webpack-style code-splitting (see Finding 10), but it's a minority contributor to total JS weight.

**Reproduce:** DevTools Network tab, filter by **JS**, read total transferred/resource size at the bottom.

**Fix:** Primarily a third-party reduction problem (Finding 12); secondarily, first-party bundle splitting (Finding 10).

**Effort:** High for the third-party portion, Medium for the first-party portion.

**Structural or Local:** Mixed — Structural (third-party) + Local (first-party splitting).

---

### Finding 5 — Caching reduces bytes but not request count

**Metrics:** Requests unchanged (214 → 217) despite 75.5% byte reduction on warm load

**Mechanism:** Many requests are inherently non-cacheable — personalized ad creative, tracking pixels with unique per-load IDs, and real-time content fetches that must query fresh data every load. Standard `Cache-Control` headers can't help these by design.

**Reproduce:** DevTools Network, compare cold vs. warm reload request counts and transfer sizes side by side.

**Fix:** Reduce reliance on real-time/personalized fetches for above-the-fold content; batch tracking calls; audit which "trending content" widgets truly need live data vs. periodic refresh.

**Effort:** Medium-High — requires rethinking which content is genuinely real-time vs. cacheable-with-short-TTL.

**Structural or Local:** Structural (architectural decision about data freshness).

---

### Finding 6 — Images only partially cached, unlike CSS

**Metrics:** ~1.74 MB of images re-transferred out of ~6.9 MB total on warm reload; CSS fully cached (0 kB)

**Mechanism:** Images are served through a dynamic resize/proxy service (`dims4/default/...`) where query parameters may vary by placement/crop, and ad creative changes per impression — both defeat standard browser caching even when the underlying image is unchanged.

**Reproduce:** DevTools Network, filter by **Img**, compare cold vs. warm reload transfer size for the image group specifically.

**Fix:** Standardize proxy URL query parameters so identical images share one cache key; apply long `Cache-Control`/immutable headers on the resize service's static outputs.

**Effort:** Low-Medium — mostly a CDN/proxy configuration change.

**Structural or Local:** Local.

---

### Finding 7 — Best Practices score anomaly: 35 (Desktop) vs. 77 (Mobile)

**Metrics:** Best Practices category score

**Mechanism:** Unknown — flagging honestly. This is a 42-point gap between rendering paths that we have not yet root-caused.

**Reproduce:** Run PSI on both Mobile and Desktop, compare the Best Practices score circle. Then expand the Best Practices audit list on the Desktop run specifically to see which individual checks are failing (we did not capture this detail in this baseline).

**Fix:** Cannot be scoped yet — needs the diagnostic step above first. Likely candidates based on common causes of this pattern: desktop-specific third-party script behavior, deprecated API usage that only triggers at desktop viewport, or an image aspect-ratio/console-error issue.

**Effort:** Unknown until diagnosed (the diagnosis itself is ~30 minutes of work).

**Structural or Local:** Unknown — depends on diagnosis.

---

### Finding 8 — Total Blocking Time is unexpectedly *better* on mobile than desktop

**Metrics:** TBT 3,050ms (mobile) vs. 5,350ms (desktop)

**Mechanism:** Unconfirmed — this is a genuine anomaly worth flagging rather than smoothing over. Normally CPU-throttled mobile testing produces *worse* execution-bound metrics than desktop. One plausible explanation: ad servers frequently serve different creative/scripts by viewport size, so the desktop run may have encountered a heavier ad payload than the mobile run purely by chance of ad auction outcome, not a systematic device difference.

**Reproduce:** Re-run PSI mobile and desktop back-to-back multiple times and compare TBT — if the mobile-better pattern holds consistently, it's systematic; if it flips between runs, it's likely just ad-auction variance.

**Fix:** N/A until confirmed as a real, repeatable pattern rather than test-run noise.

**Effort:** Diagnostic only for now (~15 minutes of repeated testing).

**Structural or Local:** Unknown.

---

### Finding 9 — CPU + network throttling amplify the consent/ad-blocking delay far more on mobile

**Metrics:** LCP 41.6s (mobile) vs. 15.6s (desktop) — ~2.7x worse on mobile

**Mechanism:** The same consent-banner/ad-loading payload is served to both platforms, but mobile PSI testing applies Slow 4G network throttling *and* 4x CPU slowdown together. The identical fetch-and-execute work for CMP/ad scripts takes proportionally much longer under those combined constraints — this compounds Finding 1's underlying problem rather than being a separate root cause.

**Reproduce:** Compare the LCP screenshots/values between the Mobile and Desktop PSI runs for the same URL.

**Fix:** Same as Finding 1, plus consider serving a lighter-weight consent/ad experience specifically on constrained connections (e.g. via the `navigator.connection` API) — smaller ad creative, fewer simultaneous third-party requests on slow networks.

**Effort:** Medium (builds on Finding 1's fix, adds connection-aware logic).

**Structural or Local:** Structural.

---

### Finding 10 — First-party JS bundle ships ~70% unused code

**Metrics:** 164.9 KiB of 234.7 KiB transferred is unused (per-page)

**Mechanism:** The bundle filename (`All.min...gz.js`) and its all-pages-shared naming pattern indicate it's built as one shared bundle rather than split per route. This page (the homepage) doesn't need all the code in it.

**Reproduce:** DevTools → More tools → Coverage → reload page → look at the JS entry for the first-party domain, showing used vs. unused bytes. Or expand "Reduce unused JavaScript" in PSI diagnostics.

**Fix:** Introduce route-based code-splitting (e.g. dynamic `import()` per route/page-type) and tree-shaking so the homepage bundle only includes what the homepage needs.

**Effort:** Medium — standard modern-bundler feature, but requires restructuring how the app currently imports/bundles code.

**Structural or Local:** Local (contained to build/bundling configuration, no vendor dependency).

---

### Finding 11 — First-party CSS is a single monolithic bundle, 90% unused on this page

**Metrics:** 93.4 KiB of 104.3 KiB transferred is unused

**Mechanism:** All first-party CSS ships as one file (`All.min...css`) regardless of page type. Notably (see Finding 19), this file *is* loaded asynchronously via a preload+swap technique, so it's not render-blocking — but it's still the full, unscoped bundle that gets applied.

**Reproduce:** DevTools Coverage tab (same as Finding 10, filtered to CSS), or PSI's "Reduce unused CSS" diagnostic.

**Fix:** Extract critical, page-specific CSS and inline it; split the remainder by route/component rather than shipping one file everywhere.

**Effort:** Medium — critical-CSS extraction tooling is mature (e.g. `critical`, `criticalCSS`), but requires build-pipeline integration.

**Structural or Local:** Local.

---

### Finding 12 — Third-party script sprawl accounts for ~4.36s of cumulative main-thread time

**Metrics:** ~4,361ms combined main-thread time across 33 vendors — directly explains Finding 2's TBT numbers

**Mechanism:** PSI's third-party impact table breaks this down by vendor. Top offenders: Google Tag Manager (645ms), Quantcast (465ms), Google APIs/IMA video ads (363ms), Viafoura commenting widget (330ms), Optanon consent (310ms), pub.network/Prebid (299ms), and 27 more below that. At least one confirmed exact duplicate: `dianomi`'s `contextfeed.js` loads twice in the same page load.

**Reproduce:** PSI → expand "Reduce the impact of third-party code" — shows the full vendor table with transfer size and main-thread blocking time per vendor.

**Fix:** (1) Fix the confirmed dianomi duplicate immediately — trivial, check the tag-manager container for a duplicate trigger. (2) Conduct a full vendor audit: count how many of the 11+ ad-tech vendors actually serve non-overlapping purposes, and cut/consolidate. (3) Defer non-critical vendors (social SDKs, secondary personalization tools) to load after the page is interactive rather than up front.

**Effort:** The dianomi duplicate fix: trivial (hours). The full vendor audit: High — this is the most expensive item in the whole report, and requires stakeholder/vendor-contract involvement, not just code changes.

**Structural or Local:** Structural (vendor governance), with one trivial Local fix embedded in it (the duplicate).

---

### Finding 13 — First-party production bundles ship without source maps

**Metrics:** N/A (not a performance metric — an engineering-practice observation)

**Mechanism:** Several third-party vendors (Viafoura, OneSignal) expose real source file paths (`.../avatar.vue`, `.../SessionManager.ts`) in their unused-code reports, meaning they ship source maps. The first-party AP bundle shows no equivalent source attribution in the same reports.

**Reproduce:** Open PSI's "Reduce unused JavaScript" audit, expand a first-party entry vs. a Viafoura/OneSignal entry, and compare whether original file paths are shown.

**Fix:** Enable source map generation in the build config; upload maps privately to your error-tracking tool (Sentry, etc.) rather than serving them publicly, so debugging improves without adding to the public payload.

**Effort:** Low — most bundlers support this with a config flag; the main work is wiring up private upload to your error tracker.

**Structural or Local:** Local.

---

### Finding 14 — A third-party embedded image bypasses the site's optimization pipeline

**Metrics:** 600 KiB JPEG (Riverdrop quiz widget), no resizing/format optimization applied

**Mechanism:** First-party images go through a resize/format proxy (`dims4/...`); this specific vendor's asset does not, and is served as a large, direct JPEG.

**Reproduce:** DevTools Network tab, look for `riverdrop.com` requests, check the image request's size and response headers (no `content-encoding` or resize params visible).

**Fix:** Ask the vendor if responsive/optimized delivery can be enabled; alternatively, proxy their asset through AP's own image pipeline if licensing/contract terms allow.

**Effort:** Low-Medium — mostly a vendor conversation, not engineering-heavy once agreed.

**Structural or Local:** Local (contained to one vendor integration).

---

### Finding 15 (Rendering Strategy) — A third-party A/B-testing script is given top loading priority over the page's own SSR-ready content

**Metrics:** Contributes directly to FCP (7.0s mobile) and LCP (41.6s mobile) — see Finding 1

**Mechanism:** Confirmed via raw HTML inspection: the homepage IS server-side rendered — real headline text and links are present in the initial HTML response, before any JS runs. However, the very first `<script>` tag in `<head>` is a Kameleoon (A/B-testing) script explicitly marked `fetchpriority="high"`, telling the browser to prioritize fetching it above the page's own critical resources — even though the actual content is already sitting in the HTML, ready to paint.

**Reproduce:** View Page Source on `https://apnews.com/`, search for `fetchpriority` — it appears on the Kameleoon script tag near the top of `<head>`.

**Fix:** Remove `fetchpriority="high"` from the Kameleoon script tag, or move it later in the document without a priority hint, so the browser paints the already-ready SSR content first.

**Effort:** Trivial — one attribute removal, testable immediately.

**Structural or Local:** Local. **This is the single best effort-to-impact ratio finding in the entire report.**

---

### Finding 16 (Rendering Strategy) — No images use native lazy-loading

**Metrics:** Contributes to LCP (41.6s mobile / 15.6s desktop)

**Mechanism:** All 83 `<img>` tags in the homepage's raw HTML load eagerly — `loading="lazy"` appears zero times in the source. Every image, including ones far below the fold, competes for the same connection pool as the actual LCP-candidate hero image, even though that image's markup (with a proper `srcset`) is ready immediately via SSR.

**Reproduce:** View Page Source, search for `<img`, then search for `loading="lazy"` — count the occurrences of each (83 vs. 0).

**Fix:** Add `loading="lazy"` to all images below the first viewport in the page template.

**Effort:** Low — template-level change, no architectural rework.

**Structural or Local:** Local.

---

### Finding 17 (Bundle/Loading) — CSS is loaded asynchronously but never actually scoped to critical content

**Metrics:** Ties to Finding 11's unused-CSS number (93.4 KiB)

**Mechanism:** The raw HTML reveals a proper non-blocking CSS-loading technique already in place: `<link rel="preload">` followed by JS that swaps it to `rel="stylesheet"` on load (the standard "loadCSS" pattern). This correctly prevents CSS from blocking initial render. However, what gets swapped in is still the entire monolithic bundle (Finding 11), not a page-scoped critical subset — so the loading *mechanism* is good, but the bundle *scoping* problem remains unsolved by it.

**Reproduce:** View Page Source, search for "Preload stylesheet" (an HTML comment marking this pattern) and the accompanying `<link>`/JS swap logic just after it.

**Fix:** Layer critical-CSS extraction on top of the existing async-loading infrastructure — inline a small, page-specific critical subset directly in `<head>`, and let the existing async mechanism continue loading the fuller stylesheet for everything else.

**Effort:** Medium — the async-loading infrastructure already exists and doesn't need to be rebuilt; this is additive.

**Structural or Local:** Local.

---

## Good Findings (Don't Touch These)

**CLS is well-controlled** (Mobile 0.033, Desktop 0.064 — both "good") despite the ad-heavy environment. Whatever image/ad-slot sizing discipline produced this should be preserved and extended, not disrupted while fixing the items above.

**CSS caches perfectly on repeat visits** (0 KB re-transferred on warm reload). This is a working example — use the same caching configuration approach when fixing Finding 6 (image caching).

**Field-measured INP passes** (185ms mobile / 160–161ms desktop, both "good") despite the alarming lab TBT numbers — once a user does get a working page, it responds fine. Don't let a scary lab number drive over-engineering here; the actual interaction responsiveness problem is smaller in the real world than TBT alone suggests.

**The SSR foundation is correct** — real content is server-rendered, not a client-side app shell. Every fix above should preserve this; none of them require touching the rendering architecture itself.
