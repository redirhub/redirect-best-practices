# Redirect Best Practices

> A curated guide to doing redirects right: choosing the right type, avoiding SEO damage, keeping things fast, staying secure, and verifying your work.

---

## Contents

- [Redirect Types: When to Use Which](#redirect-types-when-to-use-which)
- [SEO Best Practices](#seo-best-practices)
- [Performance & Architecture](#performance--architecture)
- [Security: Preventing Open Redirects](#security-preventing-open-redirects)
- [Testing & Monitoring](#testing--monitoring)
- [Contributing](#contributing)

---

## Redirect Types: When to Use Which

Not all redirects are created equal. Picking the wrong one can cost you rankings, confuse users, or break form submissions.

### The Quick Summary

| Code | Meaning | When to use |
|------|---------|-------------|
| **301** | Moved Permanently | Changing a URL forever, migrating domains, HTTP to HTTPS |
| **302** | Found (Temporary) | Short-term maintenance, A/B testing, temporary content moves |
| **307** | Temporary Redirect | Same as 302 but preserves the HTTP method (POST stays POST) |
| **308** | Permanent Redirect | Same as 301 but preserves the HTTP method |

### Key Resources

- **Google Search Central — [Redirects and Google Search](https://developers.google.com/search/docs/crawling-indexing/301-redirects):** The definitive guide on how Google interprets each redirect type. Covers server-side redirects, meta refresh, JavaScript redirects, and crypto redirects with implementation examples for Apache, Nginx, and PHP.
- **MDN Web Docs — [Redirections in HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections):** A complete reference for HTTP redirect status codes. Covers permanent, temporary, and special redirects, plus HTML meta refresh and JavaScript alternatives, with clear precedence rules.
- **Ahrefs — [11 Types of Redirects & Their SEO Impact](https://ahrefs.com/blog/redirects-for-seo/):** Practical breakdown of every redirect type with real examples. Explains how Google treats each one for canonicalization and links, plus how to verify Google is honoring your redirects in Search Console.

### The Rule of Thumb

Use **301** for permanent moves (domain changes, URL restructuring, HTTP→HTTPS). Use **302** for temporary situations (downtime pages, promotions, A/B tests). If your forms or API calls go through the redirect, prefer **307** and **308** to preserve the request method.

Google treats 301 as a strong canonicalization signal: link equity consolidates to the new URL. A 302 keeps the original URL indexed. Get this wrong and you can split your ranking signals across two URLs.

---

## SEO Best Practices

Redirects are one of the most powerful SEO tools you have, and also one of the easiest to mess up. The difference between a clean migration and a traffic disaster usually comes down to how you handle redirects.

### 1. Redirect to the Most Relevant Page

Every old URL should map to the closest equivalent on the new site. Redirecting everything to the homepage (a "blunt redirect" or "site-level redirect") is a common mistake. Google treats irrelevant redirects as [soft 404s](https://developers.google.com/search/docs/crawling-indexing/301-redirects), meaning you lose all the link equity from the old page.

> **Good:** `/old-product-page` → `/new-product-page`
> **Bad:** `/old-product-page` → `/homepage`

### 2. Avoid Redirect Chains

A redirect chain happens when URL A goes to B, B goes to C, and C goes to D. Each hop adds latency and dilutes link equity. Google recommends keeping it to one hop.

- **Google Lighthouse — [Avoid multiple page redirects](https://developer.chrome.com/docs/lighthouse/performance/redirects):** Explains why chains hurt Core Web Vitals and how Lighthouse flags pages with two or more redirects. Includes stack-specific guidance for Drupal and React Router.

Fix chains by updating your redirect map to point old URLs directly to the final destination.

### 3. Update Internal Links

After setting up redirects, update every internal link to point directly to the new URL. This eliminates unnecessary hops and lets crawlers discover the new URLs naturally. Use a crawler like Screaming Frog to find all internal links still pointing to old URLs.

### 4. Submit the Sitemap

After a migration, submit your new sitemap in Google Search Console and keep the old sitemap active until Google has processed all the redirects. This helps Google discover the mapping faster.

### 5. Monitor in Search Console

Keep an eye on the Index Coverage report for spikes in 404s, soft 404s, or redirect errors. The URL Inspection tool lets you verify individual redirects are being processed correctly.

### Key Resources

- **Google Search Central — [Redirects and Google Search](https://developers.google.com/search/docs/crawling-indexing/301-redirects):** The canonical reference for how Google processes redirects. Explains permanent vs temporary, server-side vs client-side, and how each redirect type affects indexing and canonicalization.
- **Ahrefs — [11 Types of Redirects & Their SEO Impact](https://ahrefs.com/blog/redirects-for-seo/):** Covers how to verify Google is consolidating link signals correctly using Search Console's external links report, plus a diagnostic method for detecting when redirects are treated as soft 404s.
- **RedirHub — [Migration Checklists](https://github.com/redirhub/migration-checklists):** Step-by-step checklists for domain migrations, platform rebrands, HTTPS migrations, and M&A scenarios. Covers pre-launch planning, redirect mapping, and post-launch monitoring.

---

## Performance & Architecture

Redirects add latency. A single redirect can add 200-500ms. A chain of three redirects can delay the page by over a second. Where you process redirects matters.

### Redirect at the Edge

The fastest redirect is the one that never reaches your origin server. Processing redirects at the CDN or edge layer eliminates the round trip to your backend.

**Where to process redirects, from fastest to slowest:**

1. **CDN/Edge** (Cloudflare, Fastly, Akamai): Sub-millisecond response, no origin hit
2. **Reverse Proxy** (Nginx, Caddy, Traefik): Single-digit milliseconds, on your infrastructure
3. **Application Layer** (Node.js, PHP, Django): 10-100ms+ depending on framework and database calls
4. **Client-Side** (JavaScript, meta refresh): Full page load + render before redirect fires

### Key Resources

- **Chrome for Developers — [Avoid multiple page redirects](https://developer.chrome.com/docs/lighthouse/performance/redirects):** Covers the performance cost of redirect chains and how Lighthouse measures them. Shows how each redirect adds a full network round trip for resources in the critical rendering path.
- **Cloudflare — [URL Forwarding](https://developers.cloudflare.com/rules/url-forwarding/) docs:** How to configure redirects at the edge using Bulk Redirects, Single Redirects, and Page Rules. Edge redirects avoid origin hits entirely.
- **Fastly — [Redirects at the Edge](https://docs.fastly.com/en/guides/redirects):** Setting up redirect logic in VCL at the CDN layer for zero-latency redirects.

### Reduce Redirects in the Critical Path

Resources needed for first paint (CSS, fonts, above-the-fold images) should never go through redirects. Each redirect on a critical resource delays rendering for all users. Use your browser's DevTools Network tab with "Preserve log" enabled to spot redirected resources in the waterfall.

---

## Security: Preventing Open Redirects

An open redirect is when your application takes a user-supplied URL and redirects the browser there without validation. Attackers exploit this to make phishing links appear to come from your domain.

### The Problem

```
https://yoursite.com/redirect?url=https://evil-site.com
```

A user sees `yoursite.com` in the link and trusts it, but lands on `evil-site.com`. Because the redirect originated from your domain, browsers and email filters are less likely to flag it.

### How to Prevent Open Redirects

1. **Avoid user-controlled redirect targets.** If you don't need redirect parameters, don't add them.
2. **Use an allowlist of known-safe URLs.** Map short names or tokens to full URLs server-side instead of accepting raw URLs as input.
3. **Use relative paths only.** If the redirect must be dynamic, only accept paths like `/dashboard` and prepend your domain.
4. **Validate any URL input.** Check the protocol (`https` only), domain (your allowlist), and path before redirecting.
5. **Add an interstitial page.** Show users they're leaving your site with the full destination URL displayed.

### Key Resources

- **OWASP — [Unvalidated Redirects and Forwards Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html):** The definitive security reference. Covers dangerous patterns in Java, PHP, ASP.NET, Rails, and Rust, plus detailed prevention strategies including URL validation, token-based redirect mapping, and interstitial pages.
- **OWASP — [Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html):** URL validation techniques that apply to redirect security. Covers DNS resolution checks, allowlist approaches, and network-layer controls.

---

## Testing & Monitoring

Setting up redirects is half the work. Verifying they work correctly is the other half.

### What to Test

- [ ] Every redirect returns the expected status code (301 vs 302)
- [ ] The destination URL is correct (no typos, no broken paths)
- [ ] No redirect chains longer than one hop
- [ ] Canonical tags on destination pages match the new URL
- [ ] HTTPS is enforced (no downgrade from HTTPS to HTTP)
- [ ] Old URLs are not in sitemaps, new URLs are
- [ ] No redirect loops

### Tools

- **`curl -I -L`**: Quick command-line verification. Add `-o /dev/null -w "%{url_effective}\n"` to see the final destination.
- **Screaming Frog SEO Spider**: Desktop crawler with redirect chain visualization. Free for up to 500 URLs.
- **Lighthouse / PageSpeed Insights**: Flags multiple redirect chains affecting performance.
- **Google Search Console — URL Inspection**: Check how Google is processing individual URLs after a migration.
- **whatsmydns.net**: Verify DNS propagation when changing domains or CDN providers.

### Monitoring After Migration

- **Search Console Index Coverage**: Watch for 404 spikes and redirect errors in the days after launch.
- **Traffic monitoring**: Organic traffic typically dips 10-20% immediately after a major migration, then recovers within 2-8 weeks. A dip beyond 30% or recovery taking longer than 12 weeks signals redirect issues.
- **Backlink monitoring**: Use Ahrefs or Semrush to verify referring domains are hitting your redirects correctly. Lost referring domains → lost link equity.
- **Crawl budget**: Google doesn't crawl redirects infinitely. If you have 100,000+ redirects, monitor crawl stats to ensure Google isn't wasting budget on old URLs.

### Key Resources

- **Google Search Central — [Redirects and Google Search](https://developers.google.com/search/docs/crawling-indexing/301-redirects):** Implementation examples for Apache, Nginx, and PHP. Explains how different redirect methods affect Google's processing.
- **Chrome for Developers — [Avoid multiple page redirects](https://developer.chrome.com/docs/lighthouse/performance/redirects):** Performance testing methodology for redirect chains and stack-specific fixes.
- **RedirHub — [Migration Checklists](https://github.com/redirhub/migration-checklists):** Testing checklists with URL inventory scripts, redirect mapping templates, and monitoring plans for different migration scenarios.

---

## Contributing

Contributions welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## License

[![CC BY-SA 4.0](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by-sa.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
