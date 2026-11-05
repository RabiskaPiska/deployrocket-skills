# site-audit-gate — Pre-Ship Site Quality Gate

## Frontmatter

```yaml
id: site-audit-gate
name: site-audit-gate
description: >
  Pre-ship site quality gate that runs before any client site goes live.
  Full checklist covering Lighthouse ≥90 performance / 100 accessibility /
  100 best-practices, axe zero WCAG 2.2 AA failures, no console errors,
  mobile 375px no overflow, all forms tested end-to-end, designed 404 page,
  load time <3s on simulated 3G, and security header basics. Defines which
  items are BLOCKING (site does not ship) vs WARNING (fix within 48h).
  Use when "ship the site", "deploy to production", "site is ready", or
  before any cloudflare-automation deployment.
triggers:
  - ship the site
  - deploy to production
  - site is ready
  - go live
  - before cloudflare-automation deploy
  - pre-ship checklist
  - site audit
  - Lighthouse score
  - accessibility audit
```

---

## 0. Decision Rule

**BLOCKING item** = site does not ship until fixed.
**WARNING item** = ship is allowed; fix must land within 48 hours with a tracked ticket.

If the operator says "ship it anyway", document each blocking item explicitly, get written acknowledgment, and log it to the client project's runbook.

---

## 1. Lighthouse Gate (BLOCKING)

Minimum acceptable scores on the **production URL** (not localhost):

| Category | Minimum | Ideal |
|---|---|---|
| Performance | 90 | 95+ |
| Accessibility | 100 | 100 |
| Best Practices | 100 | 100 |
| SEO | 90 | 100 |

**How to run:**

```sh
# Option A — Chrome DevTools (manual, one page)
# Open DevTools → Lighthouse tab → Mobile → Generate report

# Option B — Lighthouse CLI (scriptable, CI-friendly)
npm install -g lighthouse
lighthouse https://example.com \
  --output=json,html \
  --output-path=./lighthouse-report \
  --chrome-flags="--headless" \
  --only-categories=performance,accessibility,best-practices,seo
```

```sh
# Option C — Lighthouse CI (blocks CI pipeline)
npm install -g @lhci/cli
lhci autorun \
  --collect.url=https://example.com \
  --assert.assertions.categories:performance=["error",{"minScore":0.9}] \
  --assert.assertions.categories:accessibility=["error",{"minScore":1.0}] \
  --assert.assertions.categories:best-practices=["error",{"minScore":1.0}]
```

**Blocking threshold:** any category below minimum = BLOCKING.
**Run on:** homepage + at least two inner pages (a product/service page + contact/form page).

---

## 2. Accessibility Gate — axe-core (BLOCKING)

Zero WCAG 2.2 AA violations on all tested pages.

```sh
# Install
npm install -g axe-cli

# Run on a page
axe https://example.com --tags wcag2a,wcag2aa,wcag21aa,wcag22aa

# Run on multiple pages
axe \
  https://example.com \
  https://example.com/about \
  https://example.com/contact \
  --tags wcag2a,wcag2aa,wcag21aa,wcag22aa \
  --exit
```

The `--exit` flag makes the process exit non-zero on any violation — use in CI.

**Manual checks axe cannot catch (also BLOCKING):**
- Tab order is logical (tab through every interactive element)
- Focus indicator visible on all focusable elements
- Form labels are associated and read by a screen reader (test with NVDA/VoiceOver)
- Images with meaningful content have descriptive alt text
- Color contrast ≥ 4.5:1 for normal text, ≥ 3:1 for large text

---

## 3. Console Error Gate (BLOCKING)

Open the browser DevTools console on each page. The console must be **completely clean**:

- Zero `[Error]` messages
- Zero `[Warning]` messages from the application (framework warnings about key props, deprecated APIs, etc.)
- Zero failed network requests (404 on JS/CSS/images = BLOCKING)
- Zero mixed content warnings (HTTP resource on HTTPS page = BLOCKING)

**Automated check with Playwright:**

```js
// check-console.js
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const errors = [];
  page.on('console', msg => {
    if (msg.type() === 'error') errors.push(msg.text());
  });
  page.on('pageerror', err => errors.push(err.message));
  await page.goto('https://example.com');
  await page.waitForLoadState('networkidle');
  await browser.close();
  if (errors.length) {
    console.error('Console errors found:', errors);
    process.exit(1);
  }
  console.log('✅ No console errors');
})();
```

```sh
node check-console.js
```

---

## 4. Mobile Viewport Gate — 375px No Overflow (BLOCKING)

Test at exactly **375 × 812 px** (iPhone SE / common small baseline).

**What to check:**
- No horizontal scroll at any breakpoint from 320px to 768px
- No content cut off at the right edge
- All CTAs and buttons have ≥ 44 × 44 px tap targets
- Text is ≥ 16px (no forced zoom on iOS)
- Images do not exceed viewport width
- Navigation is usable (hamburger menu opens/closes correctly)

**Automated check:**

```sh
# Using Playwright
npx playwright test --project=chromium --viewport-width=375 --viewport-height=812
```

```js
// In Playwright test:
test('no horizontal overflow at 375px', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('https://example.com');
  const overflow = await page.evaluate(() => document.body.scrollWidth > window.innerWidth);
  expect(overflow).toBe(false);
});
```

Manual fallback: Chrome DevTools → Toggle device toolbar → set to 375 × 812 → scroll every page.

---

## 5. Load Time Gate — < 3s on Simulated 3G (BLOCKING)

Lighthouse already measures this (Time to Interactive / LCP). Supplement with:

```sh
# WebPageTest CLI
npm install -g webpagetest
webpagetest test https://example.com \
  --connectivity 3GFast \
  --location Dulles:Chrome \
  --runs 3 \
  --first

# Or use the free web UI: webpagetest.org
```

**Thresholds:**

| Metric | BLOCKING if worse than | Warning if worse than |
|---|---|---|
| LCP (Largest Contentful Paint) | 4.0 s | 2.5 s |
| FID / INP | 300 ms | 100 ms |
| CLS (Cumulative Layout Shift) | 0.25 | 0.10 |
| TTI (Time to Interactive) | 7.3 s | 3.8 s |
| Total page weight | 5 MB | 2 MB |

**Quick wins if failing:**
- Images: convert to WebP/AVIF, add `loading="lazy"`, set explicit `width`/`height`
- Fonts: `font-display: swap`, preload critical fonts
- JS: defer non-critical, remove unused (check Coverage tab in DevTools)
- CSS: remove unused rules (PurgeCSS), inline critical CSS
- Enable Cloudflare caching + Brotli compression

---

## 6. Forms Gate (BLOCKING)

Every form on the site must be tested end-to-end before ship:

- [ ] Happy path: valid data submits successfully, success message shown
- [ ] Required fields: submitting blank triggers validation (client + server)
- [ ] Email field: invalid format rejected before submit
- [ ] Spam protection active (Cloudflare Turnstile, honeypot, or reCAPTCHA)
- [ ] Form data reaches the backend (check Supabase / email inbox / CRM)
- [ ] Confirmation email sent to user (if applicable)
- [ ] Error states display correctly (network failure, server error)
- [ ] Form is keyboard-navigable (tab order, Enter submits)
- [ ] Labels associated with inputs (not placeholder-only)

**Document each form tested:** form name, URL, date tested, tester initials.

---

## 7. 404 Page Gate (BLOCKING)

The 404 page must:
- Exist (navigating to `/nonexistent-page` returns a proper 404 status code AND shows a designed page)
- Match the site's design (not a raw server error or blank white page)
- Include: navigation back to home, search (if applicable), helpful copy
- Return HTTP status **404** (not 200 — a "soft 404" is a SEO problem)

```sh
# Verify status code
curl -o /dev/null -s -w "%{http_code}" https://example.com/this-does-not-exist
# Must print 404, not 200
```

---

## 8. Security Headers Gate (WARNING — fix within 48h)

```sh
# Check with securityheaders.com or locally:
curl -I https://example.com | grep -iE "strict-transport|content-security|x-frame|x-content-type|referrer-policy|permissions-policy"
```

Required headers:

| Header | Minimum value |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `SAMEORIGIN` or CSP `frame-ancestors` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | At minimum: `default-src 'self'` + allowed origins |
| `Permissions-Policy` | Restrict camera/mic/geolocation if unused |

Cloudflare adds HSTS automatically when "Always Use HTTPS" is on. Set the rest in `_headers` (Cloudflare Pages) or via Transform Rules.

Missing headers = **WARNING**, not blocking (unless the site handles auth or payments, in which case CSP and HSTS are BLOCKING).

---

## 9. Cross-Browser Smoke Test (WARNING)

Test on:
- Chrome (latest) — desktop + mobile emulation
- Safari (latest) — especially on macOS/iOS for CSS grid/flexbox edge cases
- Firefox (latest)
- Edge (latest)
- Real iOS Safari on an actual device (simulator is insufficient for touch events)

Minimum per browser: homepage loads, main CTA works, contact form submits.

---

## 10. Pre-Ship Checklist — Full Run Order

Run in this order. Mark each ✅ (pass) / ❌ (fail — BLOCKING) / ⚠️ (warning):

```
LIGHTHOUSE
[ ] Performance ≥ 90 on homepage              BLOCKING
[ ] Performance ≥ 90 on key inner pages       BLOCKING
[ ] Accessibility = 100 all tested pages      BLOCKING
[ ] Best Practices = 100 all tested pages     BLOCKING
[ ] SEO ≥ 90 all tested pages                 BLOCKING

ACCESSIBILITY
[ ] axe-core zero AA violations               BLOCKING
[ ] Tab order logical                         BLOCKING
[ ] Focus indicator visible                   BLOCKING
[ ] Screen reader test passed                 BLOCKING

CONSOLE
[ ] Zero errors in DevTools console           BLOCKING
[ ] Zero failed network requests              BLOCKING
[ ] Zero mixed content warnings               BLOCKING

MOBILE
[ ] 375px no horizontal overflow              BLOCKING
[ ] All tap targets ≥ 44×44px                 BLOCKING
[ ] Text ≥ 16px no forced zoom                BLOCKING

PERFORMANCE
[ ] LCP < 4.0s on 3G                          BLOCKING
[ ] CLS < 0.25                                BLOCKING
[ ] Total weight < 5MB                        WARNING

FORMS
[ ] All forms happy-path tested               BLOCKING
[ ] All forms validation tested               BLOCKING
[ ] Spam protection active                    BLOCKING
[ ] Data reaches backend confirmed            BLOCKING

404
[ ] 404 page designed and returns HTTP 404    BLOCKING

SECURITY HEADERS
[ ] HSTS present                              BLOCKING (auth/payments), WARNING otherwise
[ ] X-Content-Type-Options: nosniff           WARNING
[ ] X-Frame-Options or CSP frame-ancestors   WARNING
[ ] CSP policy present                        BLOCKING (auth/payments), WARNING otherwise

CROSS-BROWSER
[ ] Chrome desktop                            WARNING
[ ] Safari macOS                              WARNING
[ ] Firefox                                   WARNING
[ ] Real iOS device                           WARNING
```

---

## 11. Gate Sign-Off

Before triggering cloudflare-automation deploy, record:

```
SITE AUDIT GATE SIGN-OFF
Date:
Auditor:
Site URL:
Pages tested:

BLOCKING items: [list any with status — must all be ✅]
WARNING items:  [list any with remediation tickets]

Lighthouse scores:
  Homepage perf: __  a11y: __  bp: __  seo: __
  [page 2]:      __  a11y: __  bp: __  seo: __

axe violations: 0
Console errors: 0
Mobile overflow (375px): none
Form tests: [list forms tested]
404 status: 404 confirmed

APPROVED TO SHIP: YES / NO
```

Store this document in the project's `/docs/audits/` folder and commit it.

---

gates — godmode:activation: fired | /honestscore: skipped: skill file, not code | /ideas: skipped: skill authoring, no defect | brain: skipped: subagent seat, no brain access
