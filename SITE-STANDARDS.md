# Site standards

The basics every site must have before and after launch. Copy this file into each site's repository and
import it from that repository's `CLAUDE.md` (a line reading `@SITE-STANDARDS.md`), so every Claude
session working on the site follows it. When a site is set up or changed, check each section and fix
anything missing. Where something needs the Cloudflare dashboard or a registrar, list it for the owner
with exact steps rather than skipping it.

Secrets (API tokens, keys, passwords) are never pasted into chat or committed. They go in Cloudflare
secrets, GitHub Actions secrets or the cloud environment's settings.

## 1. Domain and DNS (Cloudflare)

- The domain uses Cloudflare's nameservers and the zone is active.
- One canonical host: the bare domain (example.co.uk). www redirects to it with a 301, keeping the path
  and query string (in the app, or a Cloudflare Redirect Rule for static sites).
- Workers are attached with **custom domains** (`"custom_domain": true` in wrangler config), which create
  the DNS records and certificates. No leftover or dangling DNS records.
- **DNSSEC on** (DNS → Settings). If the domain is registered elsewhere, the DS record is added at the
  registrar.
- **CAA records** limiting certificates to the authorities Cloudflare uses (letsencrypt.org, pki.goog,
  ssl.com, digicert.com), with an `iodef` contact.

## 2. HTTPS and TLS

- SSL/TLS mode **Full (strict)**.
- **Always Use HTTPS** on: http:// redirects to https://.
- Minimum TLS **1.2**, TLS **1.3** on, Automatic HTTPS Rewrites on.
- **HSTS** sent by the site: `max-age=63072000; includeSubDomains`. Add `preload` and submit to
  hstspreload.org once the site has run cleanly for a few weeks.
- No mixed content: every asset, image and API on https.

## 3. Email (even if the site never sends any)

- Mail to the contact address works: **Cloudflare Email Routing** forwarding hello@ (or the provider's
  MX records), tested by sending a real email.
- **SPF**: exactly one SPF record. With Email Routing it includes `_spf.mx.cloudflare.net`; add the
  include for any service that sends as the domain. A domain that sends nothing ends in `-all`.
- **DKIM** for every service that sends as the domain (Google Workspace, Resend, Postmark...).
- **DMARC** at `_dmarc`:
  - Domain sends nothing: `v=DMARC1; p=reject; rua=mailto:hello@domain`.
  - Domain sends mail: start at `p=none` with reports, move to `p=quarantine`, then `p=reject` once
    SPF and DKIM pass for every sender.
- A domain that never receives mail either: null MX (`MX 0 .`) and `v=spf1 -all`.
- Optional: MTA-STS and TLS-RPT records once mail is set up.

## 4. Security headers (every response)

- `Content-Security-Policy` listing only the sources the site uses, with `frame-ancestors 'none'`,
  `base-uri 'self'`, `form-action` limited to the form endpoints, and `upgrade-insecure-requests`.
- `Strict-Transport-Security` (above), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  `Referrer-Policy: strict-origin-when-cross-origin`, a `Permissions-Policy` turning off camera,
  microphone, geolocation, payment and usb unless used.
- No `X-Powered-By` header.
- `/.well-known/security.txt` with a contact and an expiry date a year ahead.
- Static sites: these go in `public/_headers`; Next.js on Workers: in `next.config` headers.
- Check with `curl -sI https://domain/` and securityheaders.com (aim for A or A+).

## 5. Cloudflare security settings

- **Bot Fight Mode** on (verified search engine bots still get through; see the search engine rule below).
- **WAF**: the Cloudflare Free Managed Ruleset on.
- **Rate limiting** rule on every form or API that writes (for example POST /api/reviews: 10 a minute
  per IP, block for a minute).
- Forms protected against spam: honeypot field, minimum fill time, link blocking, per-IP daily limits
  (IP stored only as a salted hash), and **Turnstile** if spam gets through.
- **AI Crawl Control** blocking AI training crawlers, backing up robots.txt.
- Cloudflare notifications on for the account (certificate, DDoS, Worker errors).
- **Never block or challenge search engines.** Google and Bing crawl from the US, so a country rule (or any
  challenge or block rule) that catches them stops indexing: Search Console reports the sitemap as
  "Couldn't fetch" or HTTP 403. Every custom challenge, block and rate-limit rule must:
  - exclude verified bots: `and not cf.client.bot`;
  - never apply to `/robots.txt`, `/sitemap*.xml`, `/.well-known/*` or `/favicon.ico`
    (`and not ends_with(http.request.uri.path, ".xml") and not ends_with(http.request.uri.path, ".txt")`);
  - block crawlers by user agent only by name (the AI training bots and scrapers in section 6), never a
    generic "bot" or "crawler" match.
- **Bot Fight Mode on the free plan cannot be skipped by any rule.** If Search Console still shows 403s with
  the rules above in place, turn Bot Fight Mode off and rely on the WAF rules instead.
- **After any WAF, bot or rate-limit change**, in Google Search Console run URL Inspection → Test live URL on
  the home page and `/sitemap.xml`, and check the Sitemaps report reads "Success"; check Security → Events
  for blocks or challenges on Googlebot or Bingbot.

## 6. Crawlers, robots and indexing

- `robots.txt` allows search engines and AI search assistants, and blocks AI training crawlers:
  GPTBot, ClaudeBot, anthropic-ai, Google-Extended, Applebot-Extended, CCBot, Bytespider, Amazonbot,
  meta-externalagent, FacebookBot, cohere-ai, cohere-training-data-crawler,
  Diffbot, Omgilibot, ImagesiftBot, GoogleOther, and others as they appear.
- `sitemap.xml` listing every indexable page, referenced from robots.txt.
- Preview hosts (workers.dev, pages.dev, www before its redirect) send `X-Robots-Tag: noindex`.
- Thin or private pages (saved lists, search results with no content) are `noindex`.
- Site verified in **Google Search Console** and **Bing Webmaster Tools** (DNS TXT), sitemap submitted.

## 7. SEO basics on every page

- A unique `<title>` and meta description, a canonical URL on the canonical host, one `<h1>`.
- Open Graph and Twitter card tags, with a share image: a site-wide one at least, and per-page images
  for the pages people share most (an item's score card, a ranking's top three).
- JSON-LD where it's true: Organization, WebSite, BreadcrumbList, FAQPage for real FAQs, and the
  page's own type (Place, Product, Article...).
- Breadcrumbs, internal links between related pages, and a helpful 404 page.
- Flat, readable URLs (`/area/ls6`, `/town/leeds`) with hierarchy shown in breadcrumbs, not deep paths.
- Pages that could compete for the same search (a county and a city that is most of it) are merged or
  given clearly different jobs; never publish near-duplicates.
- **One canonical address per page:** the clean `https://` address on the bare domain, with no `www`, no
  trailing slash and no query string. Pages filled from the query string (tool results, filters, tracking
  tags) keep the canonical on the clean address. `http://` and `www` 301 to it. Search Console's "Alternate
  page with proper canonical tag" for `http://`, `www` or query-string addresses is Google agreeing, not an
  error; act only if real pages appear there or under "Duplicate, Google chose different canonical".
- **No orphan pages:** every page in the sitemap is linked from at least one other page (a hub, an A to Z, a
  "nearby" or "related" list), never only from the sitemap. Check with a crawl that compares the sitemap with
  the pages it reaches through links (`pnpm check:seo` in the site template does this).

## 7a. Indexing and technical soundness

Search engines only rank pages they can fetch, render and trust. These keep every page indexable.

- **Real status codes.** A page that doesn't exist returns **404** (or 410 once deleted), never a 200 with
  "not found" text (a soft 404). Empty results (a search with no matches, an empty filter, a deleted record)
  are `noindex` or 404, never thin indexable pages. Planned downtime returns **503** with `Retry-After`.
- **Redirects:** 301, straight to the final address, at most two hops (`http://www` to `https://` bare
  domain), never a loop or a chain ending in an error. Renamed pages always get a 301 from the old address.
- **Content and links in the HTML.** Everything that matters (text, headings, links to other pages, structured
  data) is in the server-rendered or static HTML, not added later by JavaScript. Check with URL Inspection
  → View crawled page.
- **robots.txt never blocks pages or the CSS and JavaScript they need**; `noindex` (not robots.txt) keeps
  thin pages out, because a blocked page's `noindex` can't be read. Never `noindex` and canonical-to-another
  on the same page.
- **Sitemap:** returns 200 with an XML content type, valid against the sitemap protocol, under 50,000 URLs
  per file (a sitemap index above that), only canonical, indexable, 200 pages, with `lastmod` that changes
  only when the content does. Resubmit in Search Console after big changes.
- **Speed for crawlers:** pages served as static files or from the edge cache, so time to first byte stays
  low and Google crawls more of the site.
- **Same content on mobile and desktop** (Google indexes the mobile version): nothing hidden or removed on
  small screens.
- **Every month, Search Console → Pages:** check "Crawled, currently not indexed" (usually thin or
  near-duplicate pages: improve, merge or `noindex` them), "Discovered, currently not indexed" (needs more
  internal links), "Soft 404", "Duplicate, Google chose different canonical", "Server error (5xx)" and
  "Redirect error", and Security & Manual Actions. A sudden drop in indexed pages: look first for an
  accidental site-wide `noindex`, a robots.txt change, 5xx errors or a WAF rule.
- **After any fix,** use URL Inspection → Request indexing on an example page, and Validate fix on the report.
- Expected and fine: "Alternate page with proper canonical tag" and "Page with redirect" for `http://`, `www`
  and query-string addresses; "Excluded by noindex" for pages that are meant to be `noindex`.

## 8. Sharing and saving (every site)

Every page someone might send to a friend has share and save options. They are plain links and local
storage, so no social site's scripts load and nothing personal reaches us.

- **Share buttons** (a `Share` component): the phone's native share sheet where available, then
  WhatsApp, Facebook, X, email, and Copy link with a "Copied" confirmation.
  - Share text is specific to the page ("Leith (EH6) scores 66/100: 100th of 390 in Scotland"),
    never just the page title.
  - Placed in the hero of every item, list, ranking, town and tool-result page.
  - A tool's results (search, filters, a calculator) live in the URL query string so the shared
    link opens the same results.
- **Save** (a `SaveButton` on every item page): a heart button that saves to the visitor's browser
  (localStorage, nothing sent to the server), with:
  - a **Saved** link in the header with a count (icon-only on small phones so the header fits),
  - a **/saved** page comparing saved items side by side, best figure in each row highlighted,
  - a **shareable shortlist link** (`/saved?items=a,b,c`) that opens the same comparison for whoever
    receives it, with "Add these to my saved list",
  - `noindex` on /saved.
- Share and save clicks are tracked as analytics events (Fathom `trackEvent`).
- Per-page share images (section 7) so shared links look good in WhatsApp and social feeds.

## 9. Growth features worth having

- Rankings and "best of" lists, by country, region, county and city, each with a clear job.
- A "find the right one for me" tool with presets and shareable results.
- Comparisons between items, and "on the rise" or "most improved" lists that change over time.
- Fresh data on a schedule (monthly refresh workflow) with "updated" dates shown.
- Content people want to send: score cards, top 10s, surprising facts, local pride.

## 10. Performance and hosting

- Serve pages as **static assets** wherever possible (free and unlimited on Cloudflare); only APIs run
  as a Worker (`run_worker_first` for `/api/*`).
- Keep under Cloudflare limits: 20,000 files per deploy (free plan), 25 MiB per file, Worker bundle
  under 3 MB compressed (free) by reading big data at build time.
- Images: right-sized thumbnails, `loading="lazy"`, fixed aspect boxes so nothing jumps.
- Fonts self-hosted or system; no blocking third-party scripts.
- Core Web Vitals green on mobile (PageSpeed Insights).

## 11. Mobile and accessibility (WCAG 2.2 AA)

- Nothing wider than the screen at 360 to 390px (check `scrollWidth - innerWidth` is 0 on every
  template), 16px side gutters, tap targets at least 44px.
- Keyboard usable, visible focus, labels on every control, alt text on images, enough contrast,
  headings in order, `prefers-reduced-motion` respected, light and dark modes both readable.
- An accessibility statement page.

## 12. Legal and trust (UK)

- **Privacy policy** (UK GDPR): what is collected, why, who processes it, retention, rights, ICO.
- **Cookies**: no non-essential cookies, so no banner is needed (cookieless analytics such as
  Fathom). If any are added, a consent banner first.
- **Terms of use**, **contact page**, and an **accessibility statement**.
- Footer with the company name, number and registered office (Companies Act), and ICO registration
  number.
- Credit every data licence the site uses (Open Government Licence, ODbL for OpenStreetMap,
  Creative Commons authors for photos, Crown copyright notices).
- A corrections process: "Something wrong? Tell us" linked from pages with data.

## 13. Analytics and monitoring

- Privacy-friendly analytics (Fathom), with events for shares, saves, searches and form sends.
- Uptime monitoring with alerts (Cloudflare health checks or an external monitor).
- Worker observability on; scheduled jobs open a GitHub issue when they fail.
- Data refresh jobs check the new data looks sane before publishing.

## 14. Data and storage

- Structured user content (reviews, submissions, lists) in **Cloudflare D1**, not KV: it needs queries,
  limits and moderation. KV only for simple key lookups and caches.
- Never store raw IP addresses or personal data you don't need; hash with a salt.
- Moderation: report buttons that hide content after several reports, and a way to review it.
- Back ups: D1 Time Travel covers 30 days; export important tables before schema changes.

## 15. Costs

- Stay on free tiers: static assets, Workers free plan for APIs only, D1 free limits, GitHub Actions
  minutes. Check usage monthly and before adding anything that runs per request.

## 16. Launch checklist (verify, don't assume)

```sh
dig +short example.co.uk A; dig +short www.example.co.uk A                 # resolves
curl -sI http://example.co.uk/ | grep -i location                           # -> https
curl -sI https://www.example.co.uk/ | grep -i location                      # -> bare domain
curl -sI https://example.co.uk/ | grep -iE "strict-transport|content-security|x-frame|referrer|permissions"
dig +short example.co.uk MX; dig +short example.co.uk TXT; dig +short _dmarc.example.co.uk TXT
dig +short example.co.uk DS                                                 # DNSSEC
dig +short example.co.uk CAA
curl -s https://example.co.uk/robots.txt; curl -s https://example.co.uk/sitemap.xml | head
curl -s -o /dev/null -w "%{http_code}\n" -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.co.uk/sitemap.xml   # 200
curl -sI https://example.workers.dev/ | grep -i x-robots                     # noindex on previews
```

Then: in Search Console, Test live URL on the home page and sitemap and check the Sitemaps report reads
"Success" (repeat after every WAF or bot change). Send a test email to the contact address, submit a test through every form, share a page to
WhatsApp and check the preview image, save items and open the shortlist link in another browser, and
check every template on a phone.
