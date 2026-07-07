# Course Project Performance Audit Report

## Website

**AP News** — [https://apnews.com/](https://apnews.com/)

## Why this site is a good audit candidate

AP News is the digital home of the Associated Press, one of the largest news wire services in the world, and it's a strong candidate for a performance audit because it combines nearly every content type an auditor cares about at very high traffic volume:

- **Static content** — standard article pages, author/about pages
- **Dynamic content** — homepage story feeds, topic/section pages, breaking-news modules that update without a full reload
- **Interactive features** — live blogs/real-time trackers for ongoing events (elections, major breaking news), search, interactive maps and data graphics
- **In-page loaders** — infinite-scroll feeds, lazy-loaded images, auto-refreshing live-update modules
- **Authentication** — newsletter/account sign-up and login
- **Third-party embeds** — video players, ad units, analytics/tracking scripts, social embeds

That mix of real-time live-blog infrastructure, video, ads, and interactive graphics — layered on top of an image- and article-heavy news site — makes it very likely that at least one page comes back with a poor Performance score, while simpler static pages (like an author page) should score better, giving a good spread for the audit.

## PageSpeed Insights — Homepage Scores

Tested against **https://apnews.com/**:

| Category | Mobile | Desktop |
|---|---|---|
| Performance | **35** 🔴 | **26** 🔴 |
| Accessibility | 76 🟠 | 79 🟠 |
| Best Practices | 77 🟠 | **35** 🔴 |
| SEO | 85 🟠 | 85 🟠 |

Notably, both Mobile and Desktop Performance land in the red without needing to test a secondary page — and Desktop Performance (26) is actually *worse* than Mobile (35), an unusual pattern worth investigating further. Desktop Best Practices also drops sharply to 35 versus 77 on mobile, suggesting a desktop-specific issue (e.g. a console error, deprecated API, or image/security-policy audit) distinct from the general performance problems.

## Pages to Include in the Audit

1. **Homepage** — [https://apnews.com/](https://apnews.com/)
   The main entry point and highest-traffic page; combines a dynamic story feed, ad units, and breaking-news modules, making it the natural baseline for the rest of the audit.

2. **Live Blog / Breaking News Tracker** — e.g. the current top story's live updates page (search apnews.com for "Live updates" on any ongoing event)
   The most real-time, dynamic page type on the site — auto-refreshing content and frequent in-page updates make it a strong candidate for a poor Performance/interactivity score.

3. **A Standard Article Page** — any current front-page story
   Represents the most common page type by page-view volume; tests the combination of hero images, embedded ads, and related-article widgets typical of most visits to the site.

4. **Topic/Section Page** — e.g. [https://apnews.com/politics](https://apnews.com/politics)
   Tests dynamic feed loading and infinite scroll/lazy-loaded thumbnails across a list of articles, separate from the homepage's mixed content types.

5. **Video Hub** — [https://apnews.com/video](https://apnews.com/video)
   Video embeds are one of the heaviest asset types on any news site; a good test of how well the site defers non-critical media loading.

6. **Search Results Page** — search any keyword from the site's search bar
   Tests dynamic query handling and result rendering, a common weak point for in-page loaders.

7. **Interactive Data/Graphics Page** — e.g. an election results map or interactive graphic tied to a major ongoing story
   Combines heavy client-side JavaScript with real-time or near-real-time data — likely the most JavaScript-intensive page on the site and a strong candidate for the required poor/red score.

8. **Newsletter Sign-up / Account Login Page**
   Represents the authentication and lead-capture flow; important to test separately since forms and login gating often introduce their own scripts and redirects.

---
*Fill in the PageSpeed Insights table above with real results before publishing the final report.*
