---
name: a11y-visual-qa
description: Accessibility auditing and visual regression QA for frontend code — axe-core/Playwright integration for WCAG 2.2 AA automated checks (57% of issues), pa11y-ci for multi-URL CI gates, Lighthouse CI score trending, eslint-plugin-jsx-a11y for React/TSX static analysis, Playwright toHaveScreenshot with pinned Docker baselines and clock-freeze for deterministic visual regression testing, Argos CI for screenshot review UI. Use when adding accessibility gates to CI, auditing a page for WCAG violations, or setting up Playwright visual regression tests that are stable across platforms and animations.
---

## Trigger

Fire when: user says "accessibility", "a11y", "WCAG", "axe", "pa11y", "screen reader", "aria", "visual regression", "screenshot test", "toHaveScreenshot", "Playwright snapshot", "visual QA", "baseline screenshot", "pixel diff", or any request to catch UI regressions or accessibility violations automatically.

---

## Honest Limits

axe-core finds **~57% of WCAG issues automatically** (stated on axe-core's own README). The other 43% require structured manual testing — keyboard-only navigation, screen reader testing with NVDA/JAWS/VoiceOver, cognitive load review. Automation is the floor, not the ceiling.

---

## 1. axe-core + Playwright (Primary Pick)

**Install:**
```bash
npm install -D @axe-core/playwright axe-core @playwright/test
```

### Basic accessibility test
```typescript
// tests/a11y.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("homepage has no WCAG 2.2 AA violations", async ({ page }) => {
  await page.goto("https://localhost:3000/");

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21aa", "wcag22aa"])
    .analyze();

  expect(results.violations).toEqual([]);
});

test("checkout flow is accessible", async ({ page }) => {
  await page.goto("https://localhost:3000/checkout");

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa"])
    .exclude(".third-party-widget")   // exclude known FP
    .analyze();

  // Log violations with detail
  if (results.violations.length > 0) {
    console.log(JSON.stringify(results.violations, null, 2));
  }
  expect(results.violations).toHaveLength(0);
});
```

### Scan multiple pages in one test
```typescript
const pages = ["/", "/about", "/products", "/contact", "/checkout"];

for (const path of pages) {
  test(`${path} passes WCAG 2.2 AA`, async ({ page }) => {
    await page.goto(`http://localhost:3000${path}`);
    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa", "wcag22aa"])
      .analyze();
    expect(results.violations).toEqual([]);
  });
}
```

---

## 2. axe-core CLI — Quick Terminal Scan

**Install:** `npm install -g @axe-core/cli`

```bash
# Scan a URL
axe https://localhost:3000/ --exit

# WCAG 2.2 AA only
axe https://localhost:3000/ --tags wcag2a,wcag2aa,wcag22aa --exit

# Output JSON
axe https://localhost:3000/ --stdout > a11y-results.json

# Multiple URLs
axe https://localhost:3000/ https://localhost:3000/about --exit
```

---

## 3. pa11y-ci — Multi-URL CI Gate

**Install:** `npm install -g pa11y-ci`

### Config file
```json
// .pa11yci.json
{
  "defaults": {
    "standard": "WCAG2AA",
    "runners": ["axe", "htmlcs"],
    "wait": 1000,
    "threshold": 0,
    "chromeLaunchConfig": { "args": ["--no-sandbox"] }
  },
  "urls": [
    "http://localhost:3000/",
    "http://localhost:3000/about",
    "http://localhost:3000/checkout",
    {
      "url": "http://localhost:3000/admin",
      "threshold": 5
    }
  ]
}
```

```bash
# Run the gate
pa11y-ci --config .pa11yci.json

# With sitemap
pa11y-ci --sitemap https://localhost:3000/sitemap.xml \
  --sitemap-find "https://example.com" \
  --sitemap-replace "http://localhost:3000"
```

---

## 4. eslint-plugin-jsx-a11y — Static Analysis (React/TSX)

**Install:** `npm install -D eslint-plugin-jsx-a11y`

```json
// .eslintrc.json
{
  "plugins": ["jsx-a11y"],
  "extends": ["plugin:jsx-a11y/recommended"],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-is-valid": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/click-events-have-key-events": "warn",
    "jsx-a11y/interactive-supports-focus": "warn",
    "jsx-a11y/label-has-associated-control": "error",
    "jsx-a11y/no-autofocus": "warn",
    "jsx-a11y/tabindex-no-positive": "warn"
  }
}
```

---

## 5. Lighthouse CI — Score Trending

**Install:** `npm install -g @lhci/cli`

```bash
# Single run
lhci autorun --upload.target=temporary-public-storage \
  --collect.url=http://localhost:3000/

# CI with score gate
lhci assert --config=.lighthouserc.json
```

```json
// .lighthouserc.json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:3000/", "http://localhost:3000/about"],
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:accessibility": ["error", { "minScore": 0.9 }],
        "categories:best-practices": ["warn", { "minScore": 0.85 }]
      }
    },
    "upload": {
      "target": "lhci",
      "serverBaseUrl": "https://your-lhci-server.com"
    }
  }
}
```

---

## 6. Playwright Visual Regression — Production Setup

The single tightest production configuration found across verified repos:

### playwright.config.ts
```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/visual",
  snapshotDir: "./__snapshots__",

  // Platform-segregated baselines (prevents CI false positives)
  snapshotPathTemplate: "{testDir}/__snapshots__/{testFilePath}/{arg}-{platform}{ext}",

  expect: {
    toHaveScreenshot: {
      threshold: 0.05,           // tightest production value; catches rgba overlays
      maxDiffPixels: 5,          // absolute pixel allowance
      animations: "disabled",    // freeze CSS animations
    },
  },

  use: {
    // Pin to digest for full rendering determinism
    // Run: docker pull mcr.microsoft.com/playwright:v1.64.0-noble
    launchOptions: { executablePath: process.env.PLAYWRIGHT_BROWSERS_PATH },
  },

  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
  ],
});
```

### Visual test with determinism helpers
```typescript
// tests/visual/homepage.spec.ts
import { test, expect } from "@playwright/test";

// Add fixture to freeze time and disable animations globally
test.use({
  async page({ page }, use) {
    // Freeze Date.now() BEFORE navigation — eliminates timestamp/chart flake
    await page.clock.setFixedTime(new Date("2026-01-15T10:00:00Z"));
    await use(page);
  },
});

test("homepage visual baseline", async ({ page }) => {
  await page.goto("http://localhost:3000/");
  await page.waitForLoadState("networkidle");

  // Inject animation-zero + caret-hide (fires before any page JS)
  await page.addStyleTag({
    content: `
      *, *::before, *::after {
        animation-duration: 0ms !important;
        animation-delay: 0ms !important;
        transition-duration: 0ms !important;
      }
      * { caret-color: transparent !important; }
    `,
  });

  await expect(page).toHaveScreenshot("homepage.png");
});

test("checkout visual baseline", async ({ page }) => {
  await page.goto("http://localhost:3000/checkout");

  // Mask dynamic content
  await expect(page).toHaveScreenshot("checkout.png", {
    mask: [page.locator(".order-timestamp"), page.locator(".dynamic-price")],
  });
});
```

### Baseline update workflow (GitHub Actions)
```yaml
# .github/workflows/update-visual-baselines.yml
name: Update Visual Baselines
on:
  workflow_dispatch:

jobs:
  update:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    container:
      image: mcr.microsoft.com/playwright:v1.64.0-noble
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright test --update-snapshots
      - name: Commit only if changed
        run: |
          git diff --exit-code -- "*/__snapshots__/**" || \
          (git config user.email "ci@bot" && \
           git config user.name "CI Bot" && \
           git add "*/__snapshots__/**" && \
           git commit -m "chore: update visual baselines [${{ matrix.os }}]" && \
           git push)
```

---

## 7. Argos CI — Screenshot Review UI

Drop-in Playwright integration, 5,000 screenshots/month free, no committed PNGs, GitHub OIDC.

```bash
npm install -D @argos-ci/playwright
```

```typescript
// playwright.config.ts — add Argos reporter
import { defineConfig } from "@playwright/test";
import { ArgosReporter } from "@argos-ci/playwright";

export default defineConfig({
  reporter: [
    ["list"],
    [ArgosReporter],    // uploads diffs to Argos on every run
  ],
});
```

---

## WCAG 2.2 Quick Checklist (New in 2.2)

| SC | Name | Test method |
|----|------|-------------|
| 2.4.11 | Focus Not Obscured (Minimum) | Keyboard-navigate every interactive element; focus must be partially visible | 
| 2.4.12 | Focus Not Obscured (Enhanced) | Focus must be fully visible | 
| 2.4.13 | Focus Appearance | Focus indicator ≥3px contrast ratio ≥3:1 vs adjacent colour | 
| 2.5.7 | Dragging Movements | All drag operations have a single-pointer alternative | 
| 2.5.8 | Target Size (Minimum) | 24×24px minimum, or spacing formula | 
| 3.2.6 | Consistent Help | Help mechanisms in same location across pages | 
| 3.3.7 | Redundant Entry | Previously entered info in same session is auto-populated | 
| 3.3.8 | Accessible Authentication (Minimum) | No cognitive test required (OR copy-paste allowed) | 

---

## References

- `dequelabs/axe-core` — https://github.com/dequelabs/axe-core (7,700 stars, MPL-2.0)
- `@axe-core/playwright` — https://github.com/dequelabs/axe-core-npm (MIT)
- `pa11y/pa11y-ci` — https://github.com/pa11y/pa11y-ci (MIT)
- `jsx-eslint/eslint-plugin-jsx-a11y` — https://github.com/jsx-eslint/eslint-plugin-jsx-a11y (MIT)
- `GoogleChrome/lighthouse-ci` — https://github.com/GoogleChrome/lighthouse-ci (Apache-2.0)
- `microsoft/playwright` — https://github.com/microsoft/playwright (Apache-2.0)
- `argos-ci/argos` — https://argos-ci.com (5,000 screenshots/month free)
- `petrkindlmann/qa-skills` visual-testing SKILL.md — 327-line reference
- Source reports: `a11y-auditor-deep-research-2026-10-27.md`, `pixel-qa-deep-research-2026-10-27.md`