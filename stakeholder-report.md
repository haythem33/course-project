# AP News Performance Audit — Executive Summary

**Site audited:** apnews.com (homepage)
**Prepared for:** Product/Engineering leadership
**Bottom line:** The homepage fails Google's Core Web Vitals for real users today. The overwhelming majority of the problem traces back to one thing — an unmanaged stack of 33 third-party ad and tracking scripts — not a fundamentally broken site. Most of the highest-impact fixes are cheap. A few are structural and expensive, and we say so plainly below.

---

## TL;DR — For the 30-Second Read

| | |
|---|---|
| **Current state** | Fails Google's Core Web Vitals assessment on both mobile and desktop, for real users, every day |
| **Root cause** | One dominant problem: 33 third-party vendors (ads, trackers, consent, personalization) fighting for the same page load |
| **Is the foundation sound?** | Yes — the site is built the right way (server-rendered content, stable layout, working cache strategy for some assets). This is a cleanup job, not a rebuild. |
| **Cheapest high-impact fix** | Two one-line code changes, ~1 day of engineering time, that could measurably move the needle before any bigger project starts |
| **Most expensive fix** | Consolidating/renegotiating the third-party ad & tracking vendor stack — this is an organizational project, not just an engineering one |
| **You can verify this yourself** | Paste `apnews.com` into [pagespeed.web.dev](https://pagespeed.web.dev) right now — no engineering background required |

---

## What This Is Costing the Business

- **Search ranking risk:** Google uses Core Web Vitals as a ranking signal. Our real-user data (not a lab estimate — this is Google's own aggregate of actual visitor traffic) shows the homepage **failing** this assessment on both mobile and desktop. That's a standing tax on organic search visibility, every day it goes unaddressed.
- **Reader abandonment risk:** In our slowest measured test conditions, the page took over 40 seconds to show its main content. Even accounting for real-world conditions typically being better than that worst case, our real-user data still shows meaningfully slow load experiences for a share of visitors — and slow news pages lose readers to competitors during exactly the moments (breaking news, high-traffic spikes) when speed matters most.
- **Reputation/trust risk:** A wire-service brand's core product *is* being fast and reliable with information. A sluggish, ad-heavy experience undercuts that positioning regardless of how good the journalism is.

---

## The Evidence, In Terms Anyone Can Check

We used **Google PageSpeed Insights** (pagespeed.web.dev) — the same public tool Google itself recommends — and Chrome's built-in developer tools. Anyone can re-run this in under a minute:

1. Go to pagespeed.web.dev
2. Paste `https://apnews.com/`
3. Click Analyze

**What it shows today:**

| | Mobile | Desktop |
|---|---|---|
| Overall Performance Score (0–100) | 35 | 26 |
| Real-user Core Web Vitals | **Failed** | **Failed** |

A score in the 20s–30s sits solidly in the tool's own "poor" range (its scale runs red/orange/green, and this is red). This isn't our interpretation — it's Google's own categorization of Google's own measurement.

---

## What's Already Working

This report is easy to misread as "everything is broken." It isn't. Specifically:

- **The site's core architecture is sound.** Content is server-rendered — meaning search engines and users get real content immediately, without needing to wait on JavaScript. That's the right foundation, and it's not something we need to rebuild.
- **Visual stability is good.** Content doesn't jump around as the page loads (a common, frustrating problem on ad-heavy sites) — this one's handled well already.
- **Some caching is working correctly.** Stylesheets are served from cache efficiently on repeat visits, which is the correct behavior and gives us a working model to extend to other asset types.
- **Real-world responsiveness (tapping, clicking) is passing**, even though our lab test showed a scary-looking "blocking time" number. Once users do get a working page in front of them, it responds fine.

We're not starting from zero. We're fixing specific, identifiable things layered on top of a good foundation.

---

## Ranked Recommendations

Ranked by expected business impact relative to cost — highest-value items first.

### 🟢 Tier 1 — Do This Immediately (Days, not weeks; near-zero cost)

**1. Stop a marketing tool from jumping the line in front of real content.**
We found a third-party analytics/testing script explicitly configured to load *before* the page's own content, even though that content is 100% ready to display. This is a one-line configuration fix. Low cost, meaningful and immediate improvement to how fast the page *feels*, with essentially no engineering risk.

**2. Stop every image on the page from competing for bandwidth at once.**
Right now, all images — including ones far below what the user can actually see — load simultaneously, competing with the one image that actually matters for first impressions. A small, low-risk template change fixes this. Again: days, not weeks.

*We recommend shipping these two fixes first, in isolation, then re-measuring before committing budget to anything larger below — they're cheap enough to be a "free" experiment on how much of the problem they resolve on their own.*

### 🟡 Tier 2 — Structural, Higher Cost, Highest Overall Payoff

**3. Audit and consolidate the third-party vendor stack.**
This is the single biggest lever available and the most expensive to pull. We counted **33 separate third-party services** running on the homepage — ad networks, analytics tools, consent management, personalization engines — several of which appear to serve overlapping or redundant purposes. Combined, these vendors account for the large majority of the page's measured unresponsiveness. This is not purely an engineering fix: it requires vendor contract review, stakeholder buy-in across marketing/ad-ops/legal, and prioritization decisions about which tools are worth their performance cost. Expect this to be a project measured in weeks to months, not days, but it's where the majority of the *sustained* benefit lives.

**4. Reduce unnecessary code weight (both site code and styling).**
A meaningful share of the page's own code and styling is downloaded but never actually used on this page — the site currently ships a one-size-fits-all bundle to every page rather than tailoring it to what each page needs. This is a standard, well-understood engineering investment (a focused sprint, not a multi-month rebuild), and it compounds well with Tier 1 fixes.

### 🔵 Tier 3 — Lower Urgency / Needs Further Diagnosis

**5. Investigate a scoring anomaly specific to desktop.**
One of Google's four scoring categories (Best Practices, which flags security/code-quality issues, not raw speed) drops sharply on desktop versus mobile. We don't yet know the specific cause — this needs a short diagnostic pass before it can be scoped or costed, but it's worth a few hours of investigation given the size of the gap.

**6. Improve repeat-visit efficiency for images and reduce redundant network requests.**
Lower urgency than the items above — real but smaller-magnitude opportunities to make return visits faster and reduce total requests per load.

---

## Bottom Line for Budget Decisions

- **If we fund nothing else this quarter:** fund Tier 1. It's cheap, fast, low-risk, and gives us real data on how much of this problem is solvable without a bigger commitment.
- **If we fund one larger initiative:** it should be the third-party vendor audit (Tier 2, item 3) — everything else on this list is secondary to that single root cause.
- **What we're not recommending:** a full site rebuild. The architecture is sound. This is a cleanup and governance problem, not an engineering-foundations problem.
