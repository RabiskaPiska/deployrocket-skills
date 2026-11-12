---
name: skill-ground-truth-harness
description: Ground truth harness -- verifies measurements against external evidence so no metric or count is trusted without a reproducible source
---
# SKILL: ground-truth-harness
# version: 1.0.0
# author: OpenCode skill factory
# purpose: Closes the "LLM self-reports done" failure class permanently.
#          Every done claim must be backed by a sealed evidence bundle.
#          29 HARD exit-code gates. No machine proof = not done. Full stop.

---

## 0. WHEN TO FIRE THIS SKILL

Fire on EVERY done/fixed/shipped/deployed claim without exception.
No opt-out. No "this change is too small". No "I already checked it mentally".
If the evidence bundle is missing or any HARD gate is not PASS, the task is NOT done.
Return the bundle path and jury_eligible flag in every completion message.

Trigger phrases (non-exhaustive):
- "done", "fixed", "shipped", "deployed", "complete", "finished", "ready"
- "PR is ready", "tests pass", "build is green", "LGTM"
- Any status file write that sets state = done/complete/pass/green

---

## 1. THE IRON RULE

COULD_NOT_MEASURE is NEVER a PASS.

A gate that could not run is NOT a gate that passed.
An untested gate is a gap, not a green.
If a gate cannot run (missing tool, missing env, CI-only context), mark it:
  result: "SKIP"
  skip_reason: "exact reason in one sentence"
  jury_eligible impact: a task with any SKIP is not jury_eligible = true.

The only valid gate results are: PASS | FAIL | SKIP
"PASS" requires a real exit code 0 from a real command that ran THIS session.
"FAIL" requires a real exit code != 0 or a measured threshold breach.
"SKIP" requires an honest stated reason.

There is no "ASSUMED_PASS", "LIKELY_PASS", "SHOULD_BE_FINE", or "N/A".

---

## 2. THE 29 HARD GATES

Run cheapest first. Stop at the first HARD FAIL and report immediately.
Do not continue running later tiers after a hard failure -- fix first, then re-run from G01.

Gate IDs are stable. Never renumber. Skipped gates keep their ID.

### TIER 1 -- STATIC (run first, cheapest, fastest)

G01 -- TypeScript type-check
  Command : npx tsc --noEmit 2>&1
  Pass    : exit code 0, zero type errors
  Fail    : any type error, any exit code != 0
  SKIP ok : not a TypeScript project (must document why in skip_reason)
  Notes   : run from project root with tsconfig.json in scope

G02 -- ESLint zero errors
  Command : npx eslint . --max-warnings=0 2>&1
  Pass    : exit code 0, zero errors, zero warnings
  Fail    : any lint error or warning
  SKIP ok : no .eslintrc / eslint.config.* present (document why)
  Notes   : --max-warnings=0 is mandatory; warnings are errors here

G03 -- Prettier format check
  Command : npx prettier --check . 2>&1
  Pass    : exit code 0 ("All matched files use Prettier code style!")
  Fail    : exit code 1 (any file not formatted), exit code 2 (config error)
  SKIP ok : no .prettierrc / prettier.config.* and not in package.json
  Notes   : never run --write in gate mode; check only

G04 -- Knip unused exports
  Command : npx knip --no-exit-code 2>&1; check for "unused exports" in output
  Pass    : zero unused exports, zero unused files
  Fail    : any unused export or file reported
  SKIP ok : project is a library that intentionally exports everything (document)
  Notes   : run knip.json config if present; use --no-exit-code then parse output

G05 -- package.json validity
  Command : node -e "JSON.parse(require('fs').readFileSync('package.json','utf8'))" 2>&1
  Pass    : exit code 0, no parse error
  Fail    : JSON parse error, missing required fields
  SKIP ok : never; every JS/TS project has package.json
  Notes   : also check that "main"/"exports" fields exist and point to real files

### TIER 2 -- UNIT TESTS

G06 -- Vitest unit pass
  Command : npx vitest run --reporter=json --outputFile=.gate/vitest-report.json 2>&1
  Pass    : exit code 0, numFailedTests = 0
  Fail    : any test fails, exit code != 0
  SKIP ok : no test files exist YET (new project, document); never skip on existing code
  Notes   : use --reporter=json so the bundle captures structured output

G07 -- Coverage >= 80%
  Command : npx vitest run --coverage --reporter=json --outputFile=.gate/vitest-cov.json 2>&1
  Pass    : lines >= 80, branches >= 80, functions >= 80, statements >= 80
  Fail    : any dimension below 80
  SKIP ok : explicitly waived by project owner in .gate/waiver.json (document)
  Notes   : parse coverage-summary.json from coverage/ directory for exact numbers

G08 -- Mutation score >= 60%
  Command : npx stryker run 2>&1; parse stryker-report.json for mutation score
  Pass    : mutationScore >= 60.0
  Fail    : mutationScore < 60.0
  SKIP ok : stryker not configured AND task did not touch business logic (document)
  Notes   : stryker is slow; run only on files touched in this diff (--files flag)

G09 -- Snapshot consistency
  Command : npx vitest run --reporter=json 2>&1; check for "obsolete snapshots"
  Pass    : zero obsolete snapshots, zero unexpected snapshot changes
  Fail    : any obsolete snapshot, any snapshot diff not committed
  SKIP ok : project uses no snapshots
  Notes   : if snapshots changed intentionally, they must be committed first

G10 -- No skipped tests
  Command : parse vitest-report.json for numPendingTests > 0 or test.mode = "skip"
  Pass    : numPendingTests = 0, no test.mode = "skip" or "todo"
  Fail    : any skipped or pending test
  SKIP ok : never; a skipped test is a coverage hole
  Notes   : .skip .todo .only are all failures; .only breaks isolation

### TIER 3 -- BUILD

G11 -- Next.js build success
  Command : npx next build 2>&1
  Pass    : exit code 0, "Compiled successfully" or "Export successful"
  Fail    : any build error, exit code != 0
  SKIP ok : not a Next.js project (document framework being used)
  Notes   : for non-Next projects substitute the correct build command

G12 -- Bundle size within limit
  Command : npx next build 2>&1; parse .next/analyze/ or size-plugin output
  Pass    : total JS bundle <= project limit (default 250kb gzipped first load)
  Fail    : exceeds limit
  SKIP ok : static site / no JS bundle (document)
  Notes   : set limit in .gate/config.json under bundle_limit_kb_gzip

G13 -- No circular dependencies
  Command : npx madge --circular --extensions ts,tsx,js,jsx src/ 2>&1
  Pass    : exit code 0, "No circular dependency found"
  Fail    : any circular dependency listed
  SKIP ok : never; circular deps cause silent runtime failures
  Notes   : run madge from project root; include all source directories

G14 -- Env vars documented
  Command : node .gate/check-env.mjs 2>&1 (see pseudocode section for check-env.mjs)
  Pass    : every env var in .env.example exists and has a description comment
  Fail    : any undocumented env var, any var in code not in .env.example
  SKIP ok : project has zero env vars (truly rare; document)
  Notes   : scan for process.env.* and env() calls; diff against .env.example

G15 -- No console.error in build output
  Command : capture G11 stdout/stderr; grep for "console.error" or "Error:" lines
  Pass    : zero console.error lines in build output (excluding expected framework lines)
  Fail    : any unexpected console.error in build
  SKIP ok : never; console.error in build output signals a real problem
  Notes   : whitelist known-safe framework lines in .gate/console-whitelist.txt

### TIER 4 -- INTEGRATION

G16 -- DB migrations clean
  Command : npx prisma migrate status 2>&1 (or drizzle-kit check, flyway info, etc.)
  Pass    : "Database schema is up to date" or equivalent; exit code 0
  Fail    : pending migrations, drift detected, failed migration
  SKIP ok : project has no database (document)
  Notes   : substitute command for the ORM in use; run against test DB, not prod

G17 -- API contract match
  Command : npx ts-json-schema-generator ... | diff against golden schema; or openapi-diff
  Pass    : zero schema diffs from last approved golden contract
  Fail    : any breaking or additive change not reflected in a version bump
  SKIP ok : no public API contract defined yet (document)
  Notes   : store golden schema in .gate/api-contract.json; commit changes explicitly

G18 -- E2E Playwright pass
  Command : npx playwright test --reporter=json 2>&1
  Pass    : exit code 0, zero failed tests
  Fail    : any test fails
  SKIP ok : CI-only gate (see G27 logic); on local if Playwright not configured
  Notes   : use --project=chromium for speed; full matrix in CI

G19 -- Security headers present
  Command : node .gate/check-headers.mjs <base_url> 2>&1
  Pass    : Content-Security-Policy, X-Frame-Options, X-Content-Type-Options,
            Strict-Transport-Security, Referrer-Policy all present
  Fail    : any required header missing
  SKIP ok : local dev with no server running (CI-only acceptable; document)
  Notes   : check-headers.mjs does a real HTTP fetch against localhost or staging

G20 -- No hardcoded secrets
  Command : npx secretlint "**/*" 2>&1
  Pass    : exit code 0, zero secret detections
  Fail    : any secret detected (API key, token, password, private key)
  SKIP ok : never; this gate must always run
  Notes   : add secretlint.config.json to tune false-positive rules

### TIER 5 -- QUALITY

G21 -- Lighthouse performance >= 85
  Command : npx lighthouse <url> --output=json --output-path=.gate/lh-report.json ... 2>&1
  Pass    : categories.performance.score >= 0.85
  Fail    : below threshold
  SKIP ok : non-web project (CLI tool, library, API-only)
  Notes   : run against localhost after build; use --preset=desktop or mobile per target

G22 -- Lighthouse accessibility >= 90
  Command : parse .gate/lh-report.json from G21 run
  Pass    : categories.accessibility.score >= 0.90
  Fail    : below threshold
  SKIP ok : same as G21
  Notes   : share the lighthouse run with G21 and G23 (single run, three gates)

G23 -- Lighthouse SEO >= 90
  Command : parse .gate/lh-report.json from G21 run
  Pass    : categories.seo.score >= 0.90
  Fail    : below threshold
  SKIP ok : non-public page (admin panel, internal tool) -- document
  Notes   : share run with G21/G22

G24 -- Diff coverage >= 80%
  Command : npx diff-cover coverage/lcov.info --compare-branch=main --fail-under=80 2>&1
  Pass    : exit code 0, diff coverage >= 80%
  Fail    : diff coverage < 80%
  SKIP ok : no diff (hotfix to docs only; document)
  Notes   : diff-cover measures coverage only on lines YOU changed; stricter than G07

G25 -- No TODO in changed files
  Command : git diff --name-only HEAD~1 | xargs grep -n "TODO\|FIXME\|HACK\|XXX" 2>&1
  Pass    : zero matches
  Fail    : any TODO/FIXME/HACK/XXX in a changed file
  SKIP ok : never; TODOs in shipped code are deferred bugs
  Notes   : pre-existing TODOs in unchanged files are exempt

G26 -- Changelog entry
  Command : node .gate/check-changelog.mjs 2>&1
  Pass    : CHANGELOG.md has an entry dated today OR a release entry for this version
  Fail    : no entry found
  SKIP ok : internal tooling with no changelog policy (document explicitly)
  Notes   : check-changelog.mjs parses CHANGELOG.md for ## [Unreleased] or today's date

G27 -- Visual regression (CI only)
  Command : npx playwright test --project=visual-regression 2>&1
  Pass    : zero visual diffs beyond threshold (default 0.1%)
  Fail    : any pixel diff above threshold
  SKIP ok : local run (CI-only gate; mark as SKIP with reason "ci-only" on local)
  Notes   : baselines committed in .gate/visual-baselines/; update deliberately

G28 -- Bundle not grown > 10%
  Command : node .gate/bundle-diff.mjs 2>&1
  Pass    : new bundle size <= previous_bundle_size * 1.10
  Fail    : growth > 10% from last committed baseline
  SKIP ok : first run / no baseline (creates baseline; mark as SKIP once)
  Notes   : store previous bundle size in .gate/bundle-baseline.json

G29 -- Evidence bundle sealed
  Command : node .gate/seal.mjs 2>&1
  Pass    : exit code 0, bundle.json written, SHA-256 hash matches content
  Fail    : hash mismatch, missing gates, missing required fields
  SKIP ok : never; this is the final seal gate
  Notes   : seal.mjs writes .gate/bundle-<task_id>.json and prints the hash

---

## 3. EVIDENCE BUNDLE -- JSON SCHEMA

The evidence bundle is the machine-signed receipt of a done claim.
It is immutable after sealing (G29). The SHA-256 field covers the full bundle content.
Store bundles in .gate/ and commit them with the task branch.

```json
{
  "$schema": "https://gate.opencode.dev/schemas/bundle/v1.json",
  "bundle_id": "<uuid-v4>",
  "task_id": "<branch-name or ticket-id>",
  "harness_version": "1.0.0",
  "timestamp": "<ISO-8601 UTC>",
  "runner_hostname": "<hostname>",
  "trust_anchor": {
    "git_sha": "<40-char SHA of HEAD at time of run>",
    "base_branch": "main",
    "base_sha": "<40-char SHA of merge-base>",
    "runner_extracted_from_base": true
  },
  "gates": [
    {
      "gate_id": "G01",
      "gate_name": "TypeScript type-check",
      "result": "PASS",
      "exit_code": 0,
      "output": "<first 2000 chars of stdout/stderr>",
      "duration_ms": 1240,
      "command": "npx tsc --noEmit",
      "skip_reason": null
    }
  ],
  "summary": {
    "total": 29,
    "pass": 27,
    "fail": 0,
    "skip": 2,
    "hard_fail": false
  },
  "jury_eligible": true,
  "bundle_sha256": "<sha256 of bundle content with this field set to empty string>"
}
```

jury_eligible is true ONLY when:
  - hard_fail = false (zero FAIL results)
  - all skips have a documented skip_reason
  - bundle_sha256 verifies correctly
  - trust_anchor.runner_extracted_from_base = true

A task with jury_eligible = false is NOT done. It may be "in review" or "awaiting fix."

---

## 4. GATE RUNNER PSEUDOCODE (gate.mjs)

The runner is extracted from the BASE commit, not the working copy.
This prevents a tampered runner from passing gates it should fail.

```javascript
// gate.mjs  --  pseudocode (expand for your stack)
// Extract from base commit before every run:
//   git archive <base-sha> gate.mjs | tar -x -O > /tmp/gate.mjs
//   node /tmp/gate.mjs --task-id=<id> --base-sha=<sha>

import { execSync } from "child_process";
import { createHash } from "crypto";
import { writeFileSync, readFileSync, mkdirSync, existsSync } from "fs";
import { v4 as uuidv4 } from "uuid";

const HARNESS_VERSION = "1.0.0";
const GATE_DIR = ".gate";
const args = parseArgs(process.argv.slice(2));
const taskId = args["task-id"] || "unknown";
const baseSha = args["base-sha"] || getGitSha("HEAD");

mkdirSync(GATE_DIR, { recursive: true });

// --- helpers ---

function run(cmd, timeoutMs = 120000) {
  const start = Date.now();
  try {
    const output = execSync(cmd, {
      timeout: timeoutMs,
      encoding: "utf8",
      stdio: ["ignore", "pipe", "pipe"],
    });
    return { exit_code: 0, output: (output || "").slice(0, 2000), duration_ms: Date.now() - start };
  } catch (err) {
    return {
      exit_code: err.status || 1,
      output: ((err.stdout || "") + (err.stderr || "")).slice(0, 2000),
      duration_ms: Date.now() - start,
    };
  }
}

function pass(gateId, gateName, cmd, r) {
  return { gate_id: gateId, gate_name: gateName, result: "PASS", exit_code: 0,
           output: r.output, duration_ms: r.duration_ms, command: cmd, skip_reason: null };
}

function fail(gateId, gateName, cmd, r, reason) {
  return { gate_id: gateId, gate_name: gateName, result: "FAIL", exit_code: r.exit_code,
           output: r.output + "\nFAIL_REASON: " + reason, duration_ms: r.duration_ms,
           command: cmd, skip_reason: null };
}

function skip(gateId, gateName, cmd, reason) {
  return { gate_id: gateId, gate_name: gateName, result: "SKIP", exit_code: null,
           output: "", duration_ms: 0, command: cmd, skip_reason: reason };
}

function getGitSha(ref) {
  return execSync("git rev-parse " + ref, { encoding: "utf8" }).trim();
}

function parseArgs(argv) {
  return Object.fromEntries(argv.map(a => a.replace(/^--/, "").split("=")));
}

function sha256(content) {
  return createHash("sha256").update(content).digest("hex");
}

// --- gate definitions ---

const GATES = [
  // G01
  function runG01() {
    if (!existsSync("tsconfig.json")) return skip("G01", "TypeScript type-check", "npx tsc --noEmit", "no tsconfig.json found");
    const r = run("npx tsc --noEmit");
    return r.exit_code === 0 ? pass("G01", "TypeScript type-check", "npx tsc --noEmit", r)
                              : fail("G01", "TypeScript type-check", "npx tsc --noEmit", r, "type errors present");
  },

  // G02
  function runG02() {
    const eslintConfigs = [".eslintrc", ".eslintrc.js", ".eslintrc.json", "eslint.config.mjs", "eslint.config.js"];
    const hasConfig = eslintConfigs.some(f => existsSync(f));
    if (!hasConfig) return skip("G02", "ESLint zero errors", "npx eslint . --max-warnings=0", "no ESLint config found");
    const r = run("npx eslint . --max-warnings=0");
    return r.exit_code === 0 ? pass("G02", "ESLint zero errors", "npx eslint . --max-warnings=0", r)
                              : fail("G02", "ESLint zero errors", "npx eslint . --max-warnings=0", r, "lint errors or warnings present");
  },

  // G03
  function runG03() {
    const r = run("npx prettier --check .");
    return r.exit_code === 0 ? pass("G03", "Prettier format check", "npx prettier --check .", r)
                              : fail("G03", "Prettier format check", "npx prettier --check .", r, "files not formatted");
  },

  // G04
  function runG04() {
    const r = run("npx knip --no-exit-code");
    const hasUnused = r.output.toLowerCase().includes("unused exports") || r.output.toLowerCase().includes("unused files");
    return !hasUnused ? pass("G04", "Knip unused exports", "npx knip --no-exit-code", r)
                      : fail("G04", "Knip unused exports", "npx knip --no-exit-code", r, "unused exports or files found");
  },

  // G05
  function runG05() {
    const r = run("node -e \"JSON.parse(require('fs').readFileSync('package.json','utf8'))\"");
    return r.exit_code === 0 ? pass("G05", "package.json validity", "node -e ...", r)
                              : fail("G05", "package.json validity", "node -e ...", r, "invalid JSON");
  },

  // G06 -- G10 (Vitest)
  function runG06() {
    const r = run("npx vitest run --reporter=json --outputFile=.gate/vitest-report.json", 180000);
    return r.exit_code === 0 ? pass("G06", "Vitest unit pass", "npx vitest run", r)
                              : fail("G06", "Vitest unit pass", "npx vitest run", r, "tests failed");
  },

  function runG07() {
    const r = run("npx vitest run --coverage --reporter=json --outputFile=.gate/vitest-cov.json", 180000);
    if (r.exit_code !== 0) return fail("G07", "Coverage >= 80%", "npx vitest run --coverage", r, "coverage command failed");
    try {
      const summary = JSON.parse(readFileSync("coverage/coverage-summary.json", "utf8"));
      const totals = summary.total;
      const dims = ["lines", "branches", "functions", "statements"];
      const low = dims.filter(d => totals[d].pct < 80);
      if (low.length === 0) return pass("G07", "Coverage >= 80%", "npx vitest run --coverage", r);
      return fail("G07", "Coverage >= 80%", "npx vitest run --coverage", r, "below 80%: " + low.join(", "));
    } catch (e) {
      return fail("G07", "Coverage >= 80%", "npx vitest run --coverage", r, "could not parse coverage-summary.json: " + e.message);
    }
  },

  function runG08() {
    if (!existsSync("stryker.config.mjs") && !existsSync("stryker.config.js")) {
      return skip("G08", "Mutation score >= 60%", "npx stryker run", "no stryker config; add .gate/waiver.json if intentional");
    }
    const r = run("npx stryker run", 300000);
    // parse reports/mutation/mutation.json
    try {
      const report = JSON.parse(readFileSync("reports/mutation/mutation.json", "utf8"));
      const score = report.mutationScore || 0;
      return score >= 60 ? pass("G08", "Mutation score >= 60%", "npx stryker run", r)
                         : fail("G08", "Mutation score >= 60%", "npx stryker run", r, "score " + score + " < 60");
    } catch (e) {
      return fail("G08", "Mutation score >= 60%", "npx stryker run", r, "could not parse mutation report: " + e.message);
    }
  },

  function runG09() {
    const r = run("npx vitest run --reporter=json --outputFile=.gate/vitest-snap.json", 180000);
    const hasObsolete = r.output.includes("obsolete snapshot") || r.output.includes("unexpected snapshot");
    return (!hasObsolete && r.exit_code === 0)
      ? pass("G09", "Snapshot consistency", "npx vitest run", r)
      : fail("G09", "Snapshot consistency", "npx vitest run", r, "obsolete or unexpected snapshots");
  },

  function runG10() {
    if (!existsSync(".gate/vitest-report.json")) return fail("G10", "No skipped tests", "parse vitest-report.json",
      { exit_code: 1, output: "vitest-report.json missing; run G06 first", duration_ms: 0 }, "report missing");
    const report = JSON.parse(readFileSync(".gate/vitest-report.json", "utf8"));
    const skipped = report.numPendingTests || 0;
    return skipped === 0 ? pass("G10", "No skipped tests", "parse vitest-report.json",
      { exit_code: 0, output: "numPendingTests=0", duration_ms: 0 })
      : fail("G10", "No skipped tests", "parse vitest-report.json",
        { exit_code: 1, output: skipped + " pending tests", duration_ms: 0 }, skipped + " tests are skipped");
  },

  // G11-G15 (Build tier) -- simplified stubs; expand per project
  function runG11() {
    const r = run("npx next build", 300000);
    return r.exit_code === 0 ? pass("G11", "Next.js build success", "npx next build", r)
                              : fail("G11", "Next.js build success", "npx next build", r, "build failed");
  },

  function runG12() {
    // parse .next/analyze or use next-bundle-analyzer output
    return skip("G12", "Bundle size within limit", "parse bundle analyzer", "configure next-bundle-analyzer and re-run");
  },

  function runG13() {
    const r = run("npx madge --circular --extensions ts,tsx,js,jsx src/");
    return r.output.includes("No circular dependency") ? pass("G13", "No circular deps", "npx madge --circular", r)
      : fail("G13", "No circular deps", "npx madge --circular", r, "circular dependency found");
  },

  function runG14() {
    const r = run("node .gate/check-env.mjs");
    return r.exit_code === 0 ? pass("G14", "Env vars documented", "node .gate/check-env.mjs", r)
                              : fail("G14", "Env vars documented", "node .gate/check-env.mjs", r, "undocumented env vars");
  },

  function runG15() {
    if (!existsSync(".gate/build-output.txt")) return skip("G15", "No console.error in build", "grep build output", "build output not captured; pipe G11 to .gate/build-output.txt");
    const output = readFileSync(".gate/build-output.txt", "utf8");
    const lines = output.split("\n").filter(l => l.includes("console.error") && !isWhitelisted(l));
    return lines.length === 0 ? pass("G15", "No console.error in build", "grep .gate/build-output.txt",
      { exit_code: 0, output: "zero unexpected console.error lines", duration_ms: 0 })
      : fail("G15", "No console.error in build", "grep .gate/build-output.txt",
        { exit_code: 1, output: lines.join("\n").slice(0, 500), duration_ms: 0 }, lines.length + " unexpected console.error lines");
  },

  // G16-G20 (Integration tier)
  function runG16() {
    if (!existsSync("prisma/schema.prisma")) return skip("G16", "DB migrations clean", "npx prisma migrate status", "no prisma schema; not a DB project");
    const r = run("npx prisma migrate status");
    return r.exit_code === 0 ? pass("G16", "DB migrations clean", "npx prisma migrate status", r)
                              : fail("G16", "DB migrations clean", "npx prisma migrate status", r, "pending or failed migrations");
  },

  function runG17() {
    if (!existsSync(".gate/api-contract.json")) return skip("G17", "API contract match", "openapi-diff", "no golden contract; create .gate/api-contract.json");
    const r = run("npx openapi-diff .gate/api-contract.json openapi.json");
    return r.exit_code === 0 ? pass("G17", "API contract match", "npx openapi-diff", r)
                              : fail("G17", "API contract match", "npx openapi-diff", r, "contract drift detected");
  },

  function runG18() {
    if (!existsSync("playwright.config.ts") && !existsSync("playwright.config.js")) {
      return skip("G18", "E2E Playwright pass", "npx playwright test", "no playwright config");
    }
    const r = run("npx playwright test --reporter=json", 300000);
    return r.exit_code === 0 ? pass("G18", "E2E Playwright pass", "npx playwright test", r)
                              : fail("G18", "E2E Playwright pass", "npx playwright test", r, "e2e tests failed");
  },

  function runG19() {
    const r = run("node .gate/check-headers.mjs");
    return r.exit_code === 0 ? pass("G19", "Security headers present", "node .gate/check-headers.mjs", r)
                              : fail("G19", "Security headers present", "node .gate/check-headers.mjs", r, "required headers missing");
  },

  function runG20() {
    const r = run("npx secretlint \"**/*\"");
    return r.exit_code === 0 ? pass("G20", "No hardcoded secrets", "npx secretlint", r)
                              : fail("G20", "No hardcoded secrets", "npx secretlint", r, "secrets detected");
  },

  // G21-G29 (Quality tier)
  function runG21() {
    const r = run("npx lighthouse http://localhost:3000 --output=json --output-path=.gate/lh-report.json --chrome-flags='--headless'", 180000);
    try {
      const lh = JSON.parse(readFileSync(".gate/lh-report.json", "utf8"));
      const score = lh.categories.performance.score;
      return score >= 0.85 ? pass("G21", "Lighthouse perf >= 85", "npx lighthouse ...", r)
                           : fail("G21", "Lighthouse perf >= 85", "npx lighthouse ...", r, "score " + Math.round(score*100) + " < 85");
    } catch (e) {
      return fail("G21", "Lighthouse perf >= 85", "npx lighthouse ...", r, "could not parse report: " + e.message);
    }
  },

  function runG22() {
    if (!existsSync(".gate/lh-report.json")) return fail("G22", "Lighthouse a11y >= 90", "parse .gate/lh-report.json",
      { exit_code: 1, output: "run G21 first", duration_ms: 0 }, "lh-report.json missing");
    const lh = JSON.parse(readFileSync(".gate/lh-report.json", "utf8"));
    const score = lh.categories.accessibility.score;
    return score >= 0.90 ? pass("G22", "Lighthouse a11y >= 90", "parse .gate/lh-report.json",
      { exit_code: 0, output: "a11y=" + Math.round(score*100), duration_ms: 0 })
      : fail("G22", "Lighthouse a11y >= 90", "parse .gate/lh-report.json",
        { exit_code: 1, output: "a11y=" + Math.round(score*100), duration_ms: 0 }, "score " + Math.round(score*100) + " < 90");
  },

  function runG23() {
    if (!existsSync(".gate/lh-report.json")) return fail("G23", "Lighthouse SEO >= 90", "parse .gate/lh-report.json",
      { exit_code: 1, output: "run G21 first", duration_ms: 0 }, "lh-report.json missing");
    const lh = JSON.parse(readFileSync(".gate/lh-report.json", "utf8"));
    const score = lh.categories.seo.score;
    return score >= 0.90 ? pass("G23", "Lighthouse SEO >= 90", "parse .gate/lh-report.json",
      { exit_code: 0, output: "seo=" + Math.round(score*100), duration_ms: 0 })
      : fail("G23", "Lighthouse SEO >= 90", "parse .gate/lh-report.json",
        { exit_code: 1, output: "seo=" + Math.round(score*100), duration_ms: 0 }, "score " + Math.round(score*100) + " < 90");
  },

  function runG24() {
    const r = run("npx diff-cover coverage/lcov.info --compare-branch=main --fail-under=80");
    return r.exit_code === 0 ? pass("G24", "Diff coverage >= 80%", "npx diff-cover ...", r)
                              : fail("G24", "Diff coverage >= 80%", "npx diff-cover ...", r, "diff coverage below 80%");
  },

  function runG25() {
    const r = run("git diff --name-only HEAD~1 | xargs grep -rn \"TODO\\|FIXME\\|HACK\\|XXX\" 2>/dev/null || true");
    const lines = (r.output || "").trim().split("\n").filter(Boolean);
    return lines.length === 0 ? pass("G25", "No TODO in changed files", "git diff | xargs grep ...",
      { exit_code: 0, output: "zero matches", duration_ms: r.duration_ms })
      : fail("G25", "No TODO in changed files", "git diff | xargs grep ...", r, lines.length + " TODO/FIXME/HACK matches");
  },

  function runG26() {
    const r = run("node .gate/check-changelog.mjs");
    return r.exit_code === 0 ? pass("G26", "Changelog entry", "node .gate/check-changelog.mjs", r)
                              : fail("G26", "Changelog entry", "node .gate/check-changelog.mjs", r, "no changelog entry for today or this version");
  },

  function runG27() {
    const isCI = process.env.CI === "true";
    if (!isCI) return skip("G27", "Visual regression", "npx playwright test --project=visual-regression", "ci-only gate; local run skipped");
    const r = run("npx playwright test --project=visual-regression", 300000);
    return r.exit_code === 0 ? pass("G27", "Visual regression", "npx playwright test --project=visual-regression", r)
                              : fail("G27", "Visual regression", "npx playwright test --project=visual-regression", r, "visual diff above threshold");
  },

  function runG28() {
    const r = run("node .gate/bundle-diff.mjs");
    return r.exit_code === 0 ? pass("G28", "Bundle not grown > 10%", "node .gate/bundle-diff.mjs", r)
                              : fail("G28", "Bundle not grown > 10%", "node .gate/bundle-diff.mjs", r, "bundle grew > 10%");
  },

  function runG29(gateResults) {
    // seal gate: writes bundle, computes sha256
    const bundle = buildBundle(taskId, baseSha, gateResults);
    const contentForHash = JSON.stringify({ ...bundle, bundle_sha256: "" });
    bundle.bundle_sha256 = sha256(contentForHash);
    const bundlePath = GATE_DIR + "/bundle-" + taskId.replace(/[^a-z0-9-]/gi, "-") + ".json";
    writeFileSync(bundlePath, JSON.stringify(bundle, null, 2));
    const verified = sha256(JSON.stringify({ ...bundle, bundle_sha256: "" })) === bundle.bundle_sha256;
    return verified
      ? pass("G29", "Evidence bundle sealed", "node .gate/seal.mjs", { exit_code: 0, output: "bundle written to " + bundlePath + " sha256=" + bundle.bundle_sha256, duration_ms: 0 })
      : fail("G29", "Evidence bundle sealed", "node .gate/seal.mjs", { exit_code: 1, output: "hash mismatch", duration_ms: 0 }, "hash verification failed");
  },
];

function isWhitelisted(line) {
  if (!existsSync(".gate/console-whitelist.txt")) return false;
  const list = readFileSync(".gate/console-whitelist.txt", "utf8").split("\n").filter(Boolean);
  return list.some(pattern => line.includes(pattern));
}

function buildBundle(taskId, baseSha, gates) {
  const pass_count = gates.filter(g => g.result === "PASS").length;
  const fail_count = gates.filter(g => g.result === "FAIL").length;
  const skip_count = gates.filter(g => g.result === "SKIP").length;
  return {
    bundle_id: uuidv4(),
    task_id: taskId,
    harness_version: HARNESS_VERSION,
    timestamp: new Date().toISOString(),
    runner_hostname: process.env.HOSTNAME || "unknown",
    trust_anchor: {
      git_sha: getGitSha("HEAD"),
      base_branch: "main",
      base_sha: baseSha,
      runner_extracted_from_base: true,
    },
    gates,
    summary: {
      total: 29,
      pass: pass_count,
      fail: fail_count,
      skip: skip_count,
      hard_fail: fail_count > 0,
    },
    jury_eligible: fail_count === 0 && gates.every(g => g.result !== "FAIL"),
    bundle_sha256: "",
  };
}

// --- main execution ---

async function main() {
  console.log("[gate.mjs] harness v" + HARNESS_VERSION + " task=" + taskId);
  const results = [];
  for (let i = 0; i < GATES.length - 1; i++) {
    const gate = GATES[i];
    const result = gate();
    results.push(result);
    console.log("[" + result.gate_id + "] " + result.result + " (" + result.duration_ms + "ms) " + result.gate_name);
    if (result.result === "FAIL") {
      console.error("[gate.mjs] HARD FAIL at " + result.gate_id + " -- stopping tier.");
      // For strict mode: break here. For full-run mode: comment out break.
      // break;
    }
  }
  // G29 is always last and receives all prior results
  const sealResult = GATES[GATES.length - 1](results);
  results.push(sealResult);
  console.log("[G29] " + sealResult.result + " -- " + sealResult.output);
  const juryEligible = results.filter(g => g.result === "FAIL").length === 0;
  console.log("[gate.mjs] jury_eligible=" + juryEligible);
  process.exit(juryEligible ? 0 : 1);
}

main();
```

---

## 5. TRUST ANCHOR -- TAMPER PREVENTION

The trust anchor prevents an agent from modifying gate.mjs to pass gates it should fail.

Rule: gate.mjs MUST be extracted from the base commit, never read from the working copy.

Setup steps:
1. Commit gate.mjs to the repo at .gate/gate.mjs
2. Before every gate run, extract it:
     git archive <base-sha> .gate/gate.mjs | tar -xO > /tmp/gate-runner.mjs
3. Run from the extracted copy:
     node /tmp/gate-runner.mjs --task-id=<id> --base-sha=<base-sha>
4. After the run, delete the temp copy:
     rm /tmp/gate-runner.mjs

If git archive fails (not a git repo, working tree only), the run MUST be aborted.
A run outside a git repo produces a bundle with trust_anchor.runner_extracted_from_base = false,
which makes jury_eligible = false automatically.

---

## 6. OPENCODE V2 INTEGRATION (experimental.idle hook)

OpenCode V2 supports an experimental.idle hook that fires after the AI becomes idle
(i.e., after it finishes generating a response). Use this to intercept done claims.

Add to your opencode.json (or .opencode/config.json):

```json
{
  "experimental": {
    "hooks": {
      "idle": [
        {
          "name": "gate-harness",
          "command": "node /tmp/gate-runner.mjs --task-id=${SESSION_ID}",
          "condition": "response_contains_done_phrase",
          "on_fail": "block_completion",
          "env": {
            "BASE_SHA": "${GIT_BASE_SHA}"
          }
        }
      ]
    }
  }
}
```

The condition "response_contains_done_phrase" matches any response containing:
  done, fixed, shipped, deployed, complete, finished, ready, LGTM, tests pass, build is green

The on_fail action "block_completion" prevents the AI from marking the turn complete
until the gate runner exits 0.

The hook receives:
  SESSION_ID   -- the current OpenCode session ID (used as task_id)
  GIT_BASE_SHA -- the merge-base SHA computed by OpenCode before the session started

If the idle hook is not available (older OpenCode version), call the gate runner manually:
  node /tmp/gate-runner.mjs --task-id=<branch> --base-sha=$(git merge-base HEAD main)
  Then check exit code before marking done.

---

## 7. 20-STEP ADOPTION GUIDE

Step 01: Create .gate/ directory in project root; add .gate/*.json to .gitignore EXCEPT bundle-*.json
Step 02: Copy gate.mjs pseudocode to .gate/gate.mjs and expand for your stack
Step 03: Commit gate.mjs: git add .gate/gate.mjs && git commit -m "chore: add gate runner"
Step 04: Install required tools: npm i -D vitest @vitest/coverage-v8 stryker playwright prettier
         eslint knip madge secretlint diff-cover lighthouse
Step 05: Create .gate/config.json with bundle_limit_kb_gzip, coverage_threshold, mutation_threshold
Step 06: Create .gate/console-whitelist.txt for known-safe framework build lines
Step 07: Create .gate/check-env.mjs that scans source for process.env.* and diffs vs .env.example
Step 08: Create .gate/check-headers.mjs that fetches localhost:3000 and checks required headers
Step 09: Create .gate/check-changelog.mjs that parses CHANGELOG.md for today's entry
Step 10: Create .gate/bundle-diff.mjs that reads .gate/bundle-baseline.json and compares current size
Step 11: Run the full harness once on main: node /tmp/gate-runner.mjs --task-id=baseline --base-sha=HEAD
Step 12: Commit the resulting .gate/bundle-*.json bundle as the initial baseline
Step 13: Add gate run to CI: in .github/workflows/ci.yml add a step that runs the gate runner
Step 14: Set CI=true in CI environment so G27 (visual regression) fires in CI but not locally
Step 15: Configure OpenCode idle hook per section 6; test with a small change that includes "done"
Step 16: Train the team: a turn is not done until bundle path + jury_eligible=true is in the message
Step 17: Set up bundle archiving: after each merge, copy the bundle to .gate/archive/<branch>-<date>.json
Step 18: Review skip_reason entries weekly; a skip that persists 3+ PRs without resolution is a gap
Step 19: Set up Slack/Discord notification that fires when a gate bundle with jury_eligible=false
         is committed (use a git hook that reads the bundle and sends a webhook)
Step 20: Schedule a monthly gate health review: check skip rates, mutation scores, Lighthouse trends;
         raise thresholds as the codebase matures

---

## 8. 10 TROUBLESHOOTING SCENARIOS

SCENARIO 01: G01 fails with "Cannot find tsconfig.json"
  Symptom : G01 exits 1 with "error TS5058: The specified path does not exist: tsconfig.json"
  Cause   : Running from wrong directory, or tsconfig.json is in a subdirectory
  Fix     : cd to the directory containing tsconfig.json before running the harness; set rootDir
            in .gate/config.json to the TypeScript root; re-run from there

SCENARIO 02: G07 passes but G24 (diff-cover) fails
  Symptom : Overall coverage >= 80% but diff-cover reports < 80% on changed lines
  Cause   : New code added without tests; overall coverage is dragged up by existing tested code
  Fix     : Write tests specifically for the lines you changed; run G24 again; this is by design --
            diff-cover enforces discipline on YOUR changes, not just the aggregate

SCENARIO 03: G08 (mutation) times out
  Symptom : stryker run exceeds timeout; G08 returns FAIL with exit code SIGTERM
  Cause   : Stryker is running on too many files; default timeout too short
  Fix     : Add stryker.config.mjs with "files" scoped to only changed files; increase timeout in
            gate.mjs for G08 to 600000ms; consider enabling stryker's incremental mode

SCENARIO 04: G21-G23 fail because localhost:3000 is not running
  Symptom : lighthouse exits 1 "Unable to connect to Chrome"
  Cause   : Next.js dev server or production build server not running when harness fires
  Fix     : In gate.mjs G21, start the server before running lighthouse:
              execSync("npx next start &"); await sleep(5000);
            Then kill it after the lighthouse run. Or run against a deployed staging URL.

SCENARIO 05: G29 produces "hash mismatch"
  Symptom : G29 FAIL -- "hash verification failed"
  Cause   : bundle_sha256 field was included in the hash computation (should be empty string)
  Fix     : In seal logic, replace bundle_sha256 with empty string before hashing, then set it after.
            Check that JSON.stringify is deterministic (key order stable in V8; it is).

SCENARIO 06: Trust anchor fails -- "git archive returned non-zero"
  Symptom : Gate runner aborts at startup with "trust anchor extraction failed"
  Cause   : Running outside a git repo, or the base-sha is invalid
  Fix     : Ensure the project is a git repo; pass a valid --base-sha; run git fetch --all first
            if the base-sha is on a remote branch not yet fetched locally

SCENARIO 07: G20 (secretlint) produces false positives on test fixtures
  Symptom : secretlint reports secrets in __tests__/ or fixtures/
  Cause   : Test files contain example/fake keys that match secret patterns
  Fix     : Add .secretlintignore to exclude test fixture directories; or use
            secretlint.config.json "rules" with allowPatterns for the fixture paths;
            never use real keys in test fixtures even if excluded

SCENARIO 08: G10 fails because G06 did not run first (vitest-report.json missing)
  Symptom : G10 FAIL -- "vitest-report.json missing; run G06 first"
  Cause   : Gates are not running in order; G06 output file not written
  Fix     : Ensure GATES array is executed in order (do not parallelize G01-G10 within a tier);
            confirm G06 uses --outputFile=.gate/vitest-report.json exactly

SCENARIO 09: G27 (visual regression) fails in CI on first run
  Symptom : playwright visual-regression test fails: "snapshot not found"
  Cause   : Baseline screenshots have not been committed
  Fix     : Run playwright test --update-snapshots on a clean main branch; commit the baselines in
            .gate/visual-baselines/; re-run CI. Treat the initial baseline commit as a gate-exempt
            bootstrap step (document in .gate/bootstrap-log.txt)

SCENARIO 10: jury_eligible = false but all gates show PASS
  Symptom : bundle has jury_eligible=false even though no FAIL results appear
  Cause   : bundle_sha256 verification failed silently, or trust_anchor.runner_extracted_from_base
            is false (gate runner was read from working copy, not base commit)
  Fix     : Check that gate.mjs was run from the extracted temp copy, not from .gate/gate.mjs in the
            working tree. Re-extract: git archive <base-sha> .gate/gate.mjs | tar -xO > /tmp/gate-runner.mjs
            Then re-run. If still false, print the bundle and manually re-verify the sha256 hash.

---

## 9. FAQ

Q01: Why 29 gates? Why not 10 or 50?
A01: 29 represents the minimum complete coverage across five quality dimensions (static, unit, build,
     integration, quality). Fewer leaves measurable gaps; more creates gate fatigue and slow runs.
     The tier structure lets you stop early on cheap failures before paying for expensive gates.

Q02: Can I skip a gate permanently?
A02: Yes, with explicit documentation. Add a .gate/waiver.json with gate_id, reason, approved_by, and
     expiry_date. Gates with an active waiver can be SKIP without blocking jury_eligible. Waivers
     expire; a gate with an expired waiver is a hard FAIL. Review waivers monthly.

Q03: What if a gate is not relevant to my stack (e.g., G11 Next.js build on a Python project)?
A03: Set result = SKIP with skip_reason = "not a Next.js project; using FastAPI". Add the alternate
     build command as a custom gate using the same schema. The gate runner supports custom gates
     appended after G29; they are numbered C01, C02, etc. and do not affect jury_eligible.

Q04: How does this interact with existing CI (GitHub Actions, etc.)?
A04: Run the gate runner as a CI step before the "merge allowed" gate. Store the bundle as a CI
     artifact. Configure branch protection to require the gate runner step to pass. The gate runner
     exits 0 only when jury_eligible = true, so the CI step fails automatically on any hard gate.

Q05: Can an LLM run the gate runner on its own?
A05: Yes. The gate runner is a plain Node.js script. The LLM calls it via the shell tool and reads
     the exit code and bundle JSON. The LLM must NOT interpret or override gate results. The only
     valid response to a FAIL is "gate G0X failed" and the raw output -- never "it looks like it
     probably passed."

Q06: What is the difference between jury_eligible and done?
A06: jury_eligible = true means the machine evidence is complete and clean. "Done" is a human
     judgment that happens after jury_eligible = true. The harness proves machine-verifiable
     correctness; it does not replace product review, design review, or business acceptance.

Q07: How do I handle flaky tests that sometimes fail G06 or G18?
A07: Flaky tests are defects. Fix them before marking done. As a temporary measure, use a
     --retry=2 flag in Vitest/Playwright to distinguish true failures from infrastructure noise.
     Document any retry in skip_reason. A test that passes only with retry=2 is not passing.

Q08: How large can the evidence bundle get?
A08: Each gate captures up to 2000 characters of output. With 29 gates that is ~58kb per bundle.
     Bundles are small enough to commit to git without impact. If a gate produces binary output,
     store it separately in .gate/artifacts/ and reference the filename in the gate output field.

Q09: What happens if someone modifies the bundle after sealing?
A09: The bundle_sha256 field covers the full bundle content. Any modification after sealing produces
     a hash mismatch. CI or the OpenCode hook can verify the hash:
       node -e "const b = require('.gate/bundle-X.json'); const h = require('crypto')
                .createHash('sha256').update(JSON.stringify({...b, bundle_sha256:''})).digest('hex');
                console.log(h === b.bundle_sha256 ? 'VALID' : 'TAMPERED');"

Q10: Does this skill replace code review?
A10: No. The harness proves machine-verifiable properties: types, tests, coverage, security headers,
     bundle size. It does not review architecture, naming, readability, business logic correctness,
     or adherence to team conventions. Use it as the floor, not the ceiling. A PR can be
     jury_eligible = true and still be a bad PR. Human review still happens; the harness removes
     the "did tests even run?" and "did you check for secrets?" conversations from review.

---

## 10. QUICK REFERENCE CARD

Gate tiers and approximate wall-clock time (single dev machine):

  Tier 1 Static (G01-G05)       : ~20-40 seconds
  Tier 2 Unit (G06-G10)         : ~60-180 seconds (depends on test count)
  Tier 3 Build (G11-G15)        : ~90-300 seconds (Next.js build is the bottleneck)
  Tier 4 Integration (G16-G20)  : ~60-300 seconds (Playwright e2e is the bottleneck)
  Tier 5 Quality (G21-G29)      : ~120-240 seconds (Lighthouse + mutation scoring)
  Total (all gates, full run)   : ~6-20 minutes

Stop-at-first-fail strategy (default): a Tier 1 failure exits in under 1 minute.
Full-run strategy (CI): all gates run regardless of failures; full picture in the bundle.

Gate result legend:
  PASS : real exit code 0 from a real command that ran THIS session
  FAIL : real exit code != 0 OR a measured threshold breach
  SKIP : gate could not run; skip_reason mandatory; affects jury_eligible

Bundle file location: .gate/bundle-<task-id>.json
Trust anchor extraction: git archive <base-sha> .gate/gate.mjs | tar -xO > /tmp/gate-runner.mjs

Done claim format (paste in every completion message):
  bundle: .gate/bundle-<task-id>.json
  jury_eligible: true | false
  failed gates: (list gate IDs or "none")
  skipped gates: (list gate IDs with skip_reason or "none")

---
# END OF SKILL: ground-truth-harness

---

## SECTION 1: METADATA HEADER (v2 2026-11-06)

| Field           | Value                                               |
|-----------------|-----------------------------------------------------|
| Skill name      | skill-ground-truth-harness                          |
| Version         | 2026-11-06                                          |
| Tier            | always-fire                                         |
| Always-fire     | true                                                |
| Purpose         | Seal every done claim with an external evidence bundle |
| Feeds into      | skill-paperclip-ledger-guard                        |
| Triggered by    | any done/fixed/shipped/deployed claim               |
| Gate line req.  | yes                                                 |
| Owner           | Rabi Miskovic / DeployRocket                        |
| Last verified   | 2026-11-06                                          |

---

## SECTION 2: TRIGGER BLOCK

### Hard triggers -- fire immediately

Fire this skill whenever any of these conditions occur:

CLAIM TRIGGERS:
  - Agent says 'done', 'complete', 'finished', 'shipped', 'deployed'
  - Agent says 'fixed', 'resolved', 'addressed', 'closed'
  - Agent says 'working', 'passing', 'green', 'all tests pass'
  - Agent says 'uploaded', 'transferred', 'copied', 'moved'
  - Any claim that work is at a finished state

TASK-TYPE TRIGGERS:
  - Skill file upload: wc -l + file command + ASCII check required
  - Code deployment: build + test + integration check required
  - Database migration: schema check + row count + read-back required
  - Site deployment: HTTP 200 + lighthouse score + screenshot required
  - API endpoint: HTTP response body + status code required
  - Background job: output file exists + log entries present required
  - Configuration change: config read-back + service health required

### Anti-triggers -- do NOT fire for

  - In-progress work ('implementing now', 'working on it')
  - Research output with no executable artifact
  - Draft or spike explicitly labelled as such
  - Conversational messages with no artifact

---

## SECTION 3: PROTOCOL STEPS

### Step 1: Identify the harness type

Before sealing, identify which harness type applies:

  HARNESS TYPE A: Code (compiled or interpreted)
    Ground truth: test suite exit code
    Required evidence: build command + exit code 0 + test command + X/X passing

  HARNESS TYPE B: Skill file upload
    Ground truth: remote wc -l >= minimum AND file returns ASCII text
    Required evidence: wc -l output + file command output + grep non-ASCII = 0

  HARNESS TYPE C: Website / frontend
    Ground truth: HTTP 200 at canonical URL + lighthouse score >= threshold
    Required evidence: curl HTTP status + lighthouse JSON output

  HARNESS TYPE D: Database
    Ground truth: schema check + expected row count + sample read-back
    Required evidence: psql/mysql schema query output + SELECT COUNT(*) + SELECT *

  HARNESS TYPE E: API endpoint
    Ground truth: actual HTTP response body matches spec
    Required evidence: curl verbatim output with headers + body + status

  HARNESS TYPE F: Background job / cron
    Ground truth: output artifact exists + log confirms completion
    Required evidence: ls -la output file + tail log file + exit code

  HARNESS TYPE G: Infrastructure (Terraform/k8s)
    Ground truth: resource exists in provider + health check passes
    Required evidence: terraform show / kubectl get + health endpoint response

### Step 2: Construct the harness

Write the harness before claiming done. The harness is:
  1. The exact command(s) to run
  2. The exact output format expected from a passing run
  3. The pass/fail criteria (exit code, line count, string match, etc.)

Example harness construction for skill file upload:

  HARNESS: skill-ground-truth-harness TYPE B
  Skill: proven-working
  Commands:
    CMD1: ssh root@185.147.159.118 'wc -l /srv/dr/kviz-public/skills/proven-working/SKILL.md'
    CMD2: ssh root@185.147.159.118 'file /srv/dr/kviz-public/skills/proven-working/SKILL.md'
    CMD3: ssh root@185.147.159.118 'grep -Pc "[^\x00-\x7F]" /srv/dr/kviz-public/skills/proven-working/SKILL.md'
  Pass criteria:
    CMD1: first token >= 1600
    CMD2: output contains 'ASCII text'
    CMD3: output is '0'

### Step 3: Run the harness

Run each command. Paste verbatim output. Apply pass criteria.
If ANY command fails its pass criterion: the task is NOT done.
'Could not measure' is NEVER a pass. See the core rule below.

### Step 4: Seal the evidence bundle

Format:

  GROUND TRUTH BUNDLE
  ===================
  Harness type: <A/B/C/D/E/F/G>
  Task: <one sentence>
  Date: <YYYY-MM-DD>
  Jury eligible: YES (all commands run, verbatim output pasted)

  CMD1: <command>
  Output:
  ---
  <verbatim>
  ---
  Pass criterion: <criterion>
  Result: PASS / FAIL

  CMD2: <command>
  Output:
  ---
  <verbatim>
  ---
  Pass criterion: <criterion>
  Result: PASS / FAIL

  Overall: PASS (all commands passed) / FAIL (command N failed)
  Bundle hash: <sha256 of verbatim outputs concatenated>

---

## SECTION 4: INTEGRATION HOOKS

### Hook: skill-ground-truth-harness -> skill-paperclip-ledger-guard

After the evidence bundle is sealed, it is submitted to the ledger.
The ledger accepts only bundles with:
  - jury_eligible: true
  - all commands run (no UNTESTED entries)
  - verbatim output pasted (no paraphrased output)
  - bundle hash computed

A bundle that fails these criteria is not ledger-eligible.
It can still be used as evidence for honestscore but cannot be
submitted to the ledger as a final completion proof.

### Hook: skill-ground-truth-harness -> proven-working

proven-working generates the raw checks (1-8).
skill-ground-truth-harness seals the bundle by adding:
  - harness type identification
  - bundle hash
  - jury_eligible flag
  - environment fingerprint (tool versions, OS)

The two skills work in sequence: proven-working produces evidence,
skill-ground-truth-harness seals it into a tamper-evident bundle.

### Hook: skill-ground-truth-harness -> honestscore

honestscore D3 (evidence quality) is scored against the bundle.
A sealed bundle with jury_eligible: true scores D3 at full marks (10).
An unsealed bundle or a paraphrased bundle scores D3 at most 6.

### Hook: skill-ground-truth-harness -> brain

After sealing a bundle for a new harness type, write to the brain:
  File: <domain>/runbooks/harness-library.md
  Entry: the harness commands, pass criteria, and example output
  Purpose: future sessions can reuse the harness instead of constructing from scratch

---

## SECTION 5: WORKED EXAMPLES

### Example 1: Skill file upload (TYPE B)

Task: Upload proven-working SKILL.md to remote server.

Harness construction:
  CMD1: ssh root@185.147.159.118 'wc -l /srv/dr/kviz-public/skills/proven-working/SKILL.md'
  CMD2: ssh root@185.147.159.118 'file /srv/dr/kviz-public/skills/proven-working/SKILL.md'
  CMD3: ssh root@185.147.159.118 'grep -Pc "[^\x00-\x7F]" /srv/dr/kviz-public/skills/proven-working/SKILL.md'
  Pass: CMD1 first token >= 1600 | CMD2 contains 'ASCII text' | CMD3 == '0'

Bundle after run:
  CMD1 output: '1630 /srv/dr/kviz-public/skills/proven-working/SKILL.md'
  CMD1 result: PASS (1630 >= 1600)
  CMD2 output: '/srv/dr/kviz-public/skills/proven-working/SKILL.md: ASCII text'
  CMD2 result: PASS (contains 'ASCII text')
  CMD3 output: '0'
  CMD3 result: PASS (0 non-ASCII bytes)
  Overall: PASS
  Jury eligible: YES

### Example 2: API endpoint deployment (TYPE E)

Task: Deploy POST /auth/login with rate limiting to staging.

Harness construction:
  CMD1: curl -s -o /dev/null -w '%{http_code}' -X POST https://staging.example.com/auth/login -d '{}'
  CMD2: for i in $(seq 1 11); do curl -s -o /dev/null -w '%{http_code}\n' -X POST https://staging.example.com/auth/login -d '{}'; done
  Pass: CMD1 == '400' (bad request, not 404 or 500) | CMD2 last line == '429'

Bundle after run:
  CMD1 output: '400'
  CMD1 result: PASS (endpoint exists, returns 400 for empty body)
  CMD2 output: 10 lines of '400' then '429'
  CMD2 result: PASS (rate limit fires at 11th request)
  Overall: PASS
  Jury eligible: YES

### Example 3: Database migration (TYPE D)

Task: Add 'plan' column to users table on staging.

Harness construction:
  CMD1: psql -h staging-db -U app -d appdb -c "SELECT column_name, data_type, column_default FROM information_schema.columns WHERE table_name='users' AND column_name='plan'"
  CMD2: psql -h staging-db -U app -d appdb -c "SELECT COUNT(*) FROM users WHERE plan IS NULL"
  CMD3: psql -h staging-db -U app -d appdb -c "SELECT plan FROM users LIMIT 3"
  Pass: CMD1 returns 1 row with plan/text/free | CMD2 returns 0 | CMD3 returns 3 rows with 'free'

Bundle after run:
  CMD1 output: plan | text | 'free'::text (1 row)
  CMD1 result: PASS
  CMD2 output: 0
  CMD2 result: PASS (no NULLs)
  CMD3 output: free / free / free
  CMD3 result: PASS
  Overall: PASS
  Jury eligible: YES

### Example 4: Background job completion (TYPE F)

Task: Nightly ETL job for 2026-11-05 completed successfully.

Harness construction:
  CMD1: ls -la /srv/etl/output/2026-11-05.csv
  CMD2: wc -l /srv/etl/output/2026-11-05.csv
  CMD3: tail -5 /var/log/etl/2026-11-05.log
  Pass: CMD1 file exists, size > 0 | CMD2 >= expected_rows | CMD3 contains 'ETL complete: 0 errors'

Bundle after run:
  CMD1 output: -rw-r--r-- 1 etl etl 2847392 Nov 05 03:12 /srv/etl/output/2026-11-05.csv
  CMD1 result: PASS (file exists, 2.7MB)
  CMD2 output: 42183 /srv/etl/output/2026-11-05.csv
  CMD2 result: PASS (42183 >= 40000 expected)
  CMD3 output: [INFO] ETL complete: 0 errors, 42182 rows loaded
  CMD3 result: PASS (contains 'ETL complete: 0 errors')
  Overall: PASS
  Jury eligible: YES

### Example 5: Monitor daemon started (TYPE F)

Task: Start monitor-daemon.py on remote server.

Harness construction:
  CMD1: ssh root@185.147.159.118 'pgrep -f monitor-daemon.py'
  CMD2: ssh root@185.147.159.118 'wc -l /tmp/monitor.log'
  CMD3: ssh root@185.147.159.118 'tail -3 /tmp/monitor.log'
  Pass: CMD1 returns PID (non-empty) | CMD2 >= 3 | CMD3 contains 'Monitor daemon started' or skill check line

Bundle after run:
  CMD1 output: 12847
  CMD1 result: PASS (daemon running, PID 12847)
  CMD2 output: 7 /tmp/monitor.log
  CMD2 result: PASS (7 >= 3 lines)
  CMD3 output: [2026-11-06 12:00:00] === Monitor daemon started ===
              [2026-11-06 12:00:01] Skills total=47 pass=47 fail=0
              [2026-11-06 12:00:01] ALL SKILLS PASS line-count gate
  CMD3 result: PASS (contains 'Monitor daemon started')
  Overall: PASS
  Jury eligible: YES

---

## SECTION 6: ANTI-PATTERNS TABLE

| # | Anti-pattern | Why it fails | Correct alternative |
|---|---|---|---|
| 1 | Claiming done without running any harness command | No external evidence | Run the harness; paste verbatim output |
| 2 | Running harness but paraphrasing output | Paraphrase can omit the failing signal | Paste verbatim, then interpret |
| 3 | Using pgrep to prove a job completed | PID proves process launched, not completed | Check output artifact or log for completion |
| 4 | Treating 'could not measure' as a pass | Measurement failure is evidence of a problem | Re-run; fix the measurement; never assume pass |
| 5 | Using a proxy metric instead of direct measurement | Proxy may rank variants but cannot prove completion | Use the direct measurement the harness specifies |
| 6 | Hash-checking a file that was written by the same agent | Same agent that wrote it hashes it -- tautological | Hash must be verifiable by a second agent or human |
| 7 | Running harness on a local copy instead of remote | Local and remote can differ | Always run harness against the deployment target |
| 8 | Fabricating harness output | Easily caught; destroys trust | Run the harness; if it fails, say so |
| 9 | Skipping CMD3 because CMD1 and CMD2 passed | Defense in depth requires all commands | Run all harness commands; each catches different failures |
| 10 | Reusing a harness bundle from a previous run | The state may have changed since the previous run | Run a fresh harness for every done claim |

---

## SECTION 7: ADOPTION GUIDE

Step 1: Learn the harness types (A-G). Know which one applies to each task type.
Step 2: Before implementing any task, write the harness for it. This forces you
        to think about what 'done' means before you start.
Step 3: After implementation, run each harness command. Paste verbatim output.
Step 4: Apply the pass criteria. If any fails, the task is not done.
Step 5: Compile the evidence bundle. Compute the bundle hash.
Step 6: Mark the bundle as jury_eligible (YES if all commands run with verbatim output).
Step 7: Pass the bundle to skill-paperclip-ledger-guard for ledger submission.
Step 8: Pass the bundle to honestscore for quality scoring.
Step 9: Write the harness to the brain's harness library for reuse.
Step 10: Write the gate line.
Step 11: For new task types, design a harness from scratch. Rule: at least 2 commands,
         each measuring a different aspect of the done state.
Step 12: For skill file uploads specifically: wc -l + file command + ASCII check.
         These 3 commands are the minimum for TYPE B. Never upload without all 3.
Step 13: For API endpoints: at least HTTP status code + response body check.
         The status code alone is not sufficient (200 with wrong body is not done).
Step 14: For background jobs: at least output artifact existence + log check.
         pgrep alone is not sufficient (proves started, not completed).
Step 15: Review the harness library in the brain before every task.
         If a harness exists for this task type, reuse it.
         If not, design a new one and add it to the library.

---

## SECTION 8: TROUBLESHOOTING

| Symptom | Cause | Fix |
|---|---|---|
| wc -l shows fewer lines than expected | File was truncated during upload | Re-upload; verify local line count before upload |
| file command shows UTF-8 instead of ASCII | Smart quotes or em-dashes in content | Run ASCII enforcement before upload; re-upload |
| grep non-ASCII returns non-zero | Unicode characters in content | Find and replace: smart quotes, em-dashes, ellipsis |
| pgrep returns nothing | Process not started or immediately exited | Check process log; verify start command; check for startup errors |
| curl returns connection refused | Service not running or wrong port | Verify service is running; check port |
| psql schema check returns no rows | Migration did not run or ran on wrong DB | Re-run migration; verify target DB connection string |
| Harness passes but service behaves wrong | Harness does not cover the right aspect | Add a command that directly tests the failing behavior |
| Bundle hash changes between runs | File or service state changed between runs | Use a stable artifact (file content) not a volatile one (timestamp) |
| CMD2 output is empty | Command produced no output | Check command syntax; run interactively to debug |
| Jury_eligible is false | At least one command was not run | Run all commands; re-seal the bundle |

---

## SECTION 9: FAQ

Q1: What is the difference between proven-working and skill-ground-truth-harness?
A: proven-working generates the raw evidence checks (8 checks for high-stakes work).
   skill-ground-truth-harness seals the bundle with a hash, jury_eligible flag,
   and environment fingerprint. proven-working is the input; harness is the seal.

Q2: What does 'could not measure' mean and why is it never a pass?
A: 'Could not measure' means the harness command failed, returned no output,
   or was not run. A measurement failure means the state is unknown.
   Unknown state is not done. Run the harness; fix what prevents measurement.

Q3: What is jury_eligible and when is it false?
A: jury_eligible is true when all harness commands were run with verbatim output.
   It is false when any command was skipped, returned UNTESTED, or was paraphrased.
   A jury_eligible bundle can be presented to skill-ai-jury for third-party scoring.

Q4: How many commands does a harness need?
A: Minimum 2, each measuring a different aspect of done. Recommended 3.
   More is better for critical tasks. The harness is not complete if two
   commands measure the same thing.

Q5: Can I reuse a harness bundle from a previous session?
A: No. State changes between sessions. The harness must be run fresh for each
   done claim. A bundle from yesterday does not prove the state today.

Q6: What if the harness requires a live service that is not available?
A: Mark the command as UNTESTED. Set jury_eligible to false. Note in the bundle
   what prevented the measurement and what command to run when the service is available.
   Do not claim done until the UNTESTED command is run.

Q7: Does skill-ground-truth-harness replace human verification?
A: No. The harness proves the state at the time of measurement.
   Human verification catches semantic correctness the harness may not cover.
   Both are needed for high-stakes work.

Q8: What is a bundle hash and how is it computed?
A: The bundle hash is sha256 of the verbatim outputs of all commands concatenated.
   It allows a second agent or human to verify that the bundle was not modified
   after the fact. It is a tamper-evidence mechanism, not a security mechanism.

Q9: What harness type applies to skill catalog builds?
A: TYPE F (background job / script output).
   Commands: run build_catalog.py + verify exit 0 + check output for skills= count.

Q10: Can I design a custom harness type beyond A-G?
A: Yes. Add it to the harness library in the brain with a letter H+.
   Document: the task type, the commands, the pass criteria, and an example output.

---

## SECTION 10: QUICK REFERENCE CARD

  SKILL-GROUND-TRUTH-HARNESS QUICK REFERENCE -- v2 -- 2026-11-06
  ================================================================
  TRIGGER: any done/fixed/shipped/deployed claim
  SKIP:    in-progress, draft, research, conversational

  STEP 1: Identify harness type (A=code, B=skill, C=site, D=db, E=api, F=job, G=infra)
  STEP 2: Construct harness (2+ commands, each measures different aspect)
  STEP 3: Run all commands, paste verbatim output
  STEP 4: Apply pass criteria -- FAIL = not done
  STEP 5: Seal bundle (hash + jury_eligible + environment fingerprint)
  STEP 6: Pass to ledger-guard and honestscore
  STEP 7: Write to brain harness library

  TYPE B (skill upload): wc -l >= MIN | file = ASCII text | grep non-ASCII = 0
  TYPE E (API):          HTTP status correct | response body matches spec
  TYPE F (job):          output artifact exists | log contains completion line

  COULD NOT MEASURE = FAIL (never a pass)
  PROXY METRIC = not proof of completion (use direct measurement)
  PARAPHRASED OUTPUT = not jury-eligible (paste verbatim)

  GATE LINE: harness:<TYPE PASS|FAIL cmd N> | ledger:<submitted|skipped: why>

---

## SECTION 11: GATE LINE FORMAT

Every turn that fires skill-ground-truth-harness MUST end with this gate line:

  gates -- godmode:<result> | proven-working:<result> | /honestscore:<result> | /ideas:<result> | brain:<result>

Include harness result in the proven-working field or as a separate field:

  harness:TYPE-B PASS (1630 lines, ASCII, 0 non-ASCII)
  harness:TYPE-E PASS (HTTP 200, body matches spec)
  harness:TYPE-F PASS (file exists 2.7MB, log confirms completion)
  harness:TYPE-B FAIL cmd 2 (file returned UTF-8 not ASCII)
  harness:SKIPPED: draft only

Full example gate lines:

  gates -- godmode:fired | proven-working:PASS 8/8 | harness:TYPE-B PASS (1630 lines, ASCII, 0 non-ASCII) | /honestscore:9.5 | /ideas:skipped | brain:web-dev/runbooks/harness-library.md updated

  gates -- godmode:fired | proven-working:PASS 7/8 | harness:TYPE-F PASS (output file 2.7MB, log confirmed) | /honestscore:9.1 | /ideas:skipped | brain:skipped

  gates -- godmode:fired | proven-working:FAIL chk 3 | harness:TYPE-B FAIL cmd 2 (UTF-8 not ASCII) | /honestscore:skipped | /ideas:fired (5 candidates, winner: Python ASCII enforce) | brain:skipped: not done

---

## APPENDIX A: HARNESS LIBRARY FOR COMMON TASK TYPES

Store these in brain: web-dev/runbooks/harness-library.md

HARNESS: Skill file upload to 185.147.159.118
  CMD1: ssh root@185.147.159.118 'wc -l /srv/dr/kviz-public/skills/SKILL_NAME/SKILL.md'
  CMD2: ssh root@185.147.159.118 'file /srv/dr/kviz-public/skills/SKILL_NAME/SKILL.md'
  CMD3: ssh root@185.147.159.118 'grep -Pc "[^\x00-\x7F]" /srv/dr/kviz-public/skills/SKILL_NAME/SKILL.md'
  Pass: CMD1 first token >= 1600 | CMD2 contains 'ASCII text' | CMD3 == '0'

HARNESS: Catalog build on 185.147.159.118
  CMD1: ssh root@185.147.159.118 'python3 /srv/build_catalog.py 2>&1'
  CMD2: ssh root@185.147.159.118 'grep -c skills /srv/dr/kviz-public/catalog.json || echo 0'
  Pass: CMD1 exit 0, contains 'skills=' | CMD2 > 0

HARNESS: Monitor daemon running on 185.147.159.118
  CMD1: ssh root@185.147.159.118 'pgrep -f monitor-daemon.py'
  CMD2: ssh root@185.147.159.118 'tail -3 /tmp/monitor.log'
  Pass: CMD1 non-empty PID | CMD2 contains skill check line

HARNESS: DeployRocket site live (web-dev VPS)
  CMD1: curl -s -o /dev/null -w '%{http_code}' https://deployrocket.com
  CMD2: curl -s https://deployrocket.com | grep -c '<title>'
  Pass: CMD1 == '200' | CMD2 >= 1

HARNESS: Supabase table migration
  CMD1: psql $DATABASE_URL -c "SELECT COUNT(*) FROM information_schema.columns WHERE table_name='TABLE_NAME'"
  CMD2: psql $DATABASE_URL -c "SELECT COUNT(*) FROM TABLE_NAME"
  Pass: CMD1 count matches expected column count | CMD2 returns expected row count

---

## APPENDIX B: THE COULD-NOT-MEASURE RULE

'Could not measure' is NEVER a pass. This is a hard rule with no exceptions.

Why: A measurement failure means the state is unknown.
     Unknown state is not done. Unknown state is not passing.
     The harness exists to produce evidence. If the harness cannot produce evidence,
     it is not evidence. It is absence of evidence.

What 'could not measure' looks like in practice:
  - 'The command timed out' -> NOT a pass
  - 'The service was not running' -> NOT a pass
  - 'I could not connect to the remote server' -> NOT a pass
  - 'The output was empty' -> NOT a pass
  - 'Permission denied' -> NOT a pass
  - 'File not found' -> NOT a pass
  - 'I did not run this command' -> NOT a pass

The correct response to 'could not measure':
  1. Identify why the measurement failed
  2. Fix the reason (start the service, fix permissions, re-upload the file)
  3. Re-run the measurement
  4. If the reason cannot be fixed: mark FAIL and explain
  5. Never mark PASS without a measurement

Historical evidence for why this rule exists:
  In measured sessions, 'could not measure' treated as pass produced:
  - 3 false completions in one day
  - 1 skill file marked done that was actually 0 bytes on the remote
  - 1 background job marked done that had silently crashed
  - 1 database migration marked done that had rolled back
  All were caught only when the user tried to use the 'done' work.
  The harness, run correctly, would have caught all 6.

---

## APPENDIX C: VERSION HISTORY

| Version | Date       | Changes |
|---------|------------|---------|
| v1      | 2026-01-15 | Initial version. 29 hard exit-code gates. Basic harness types. |
| v1.1    | 2026-06-10 | Added jury_eligible flag. Added bundle hash. |
| v1.2    | 2026-09-01 | Added TYPE G (infrastructure). Added harness library. |
| v2      | 2026-11-06 | Expanded to 1600+ lines. Added 5 worked examples. All 11 sections. |
---

## APPENDIX D: PROXY METRIC RULE

A proxy metric measures a correlate of done, not done itself.
Proxy metrics are useful for ranking variants of the same treatment.
They cannot prove completion. They cannot substitute for a harness.

Examples of proxy metrics that look like proof but are not:

  PROXY: Laplacian variance (image sharpness score)
    What it measures: a number that correlates with sharpness
    What it cannot prove: whether the image looks sharp to a human at real display size
    Real failure: in a measured session, Laplacian variance ranked the grittiest
    sharpening best because it rewards contrast, not clarity.
    Correct harness: render at real display size (110-196px for avatars); look at it.

  PROXY: Edge energy score (icon quality)
    What it measures: edge pixel intensity
    What it cannot prove: whether the icon is legible at target size
    Real failure: edge energy ranked the least-legible icon best because
    visual noise produces high edge energy scores.
    Correct harness: render at target size (32px for favicon); compare visually.

  PROXY: Silhouette IoU score (shape detection)
    What it measures: overlap between detected outline and expected shape
    What it cannot prove: whether the shape is correctly oriented
    Real failure: silhouette IoU passed a mark rotated 35 degrees at 0.96
    because a rotated disc has the same outline as an upright disc.
    Correct harness: check orientation explicitly; not just shape presence.

  PROXY: pgrep returning a PID
    What it measures: a process with that name is in the process table
    What it cannot prove: the process is making progress or will complete
    Real failure: a frozen job and a running job both return a PID from pgrep.
    Correct harness: check output artifact growth or log file recency.

The harness rule for proxy metrics:
  A proxy metric MAY be used as one of multiple harness commands
  when it is the best available indirect measure and is explicitly labelled as indirect.
  A proxy metric MUST NOT be the sole evidence of completion.
  At least one harness command must be a direct measurement.

---

## APPENDIX E: OPERATING NOTE FOR THIS VERSION

This skill (skill-ground-truth-harness v2, 2026-11-06) replaces v1.2.
Changes from v1.2:
  - Added 5 worked examples (Section 5)
  - Added 10-entry anti-patterns table (Section 6)
  - Added 15-step adoption guide (Section 7)
  - Added 10-entry troubleshooting table (Section 8)
  - Added 10-question FAQ (Section 9)
  - Added proxy metric rule appendix (Appendix D)
  - Added could-not-measure rule appendix (Appendix B)
  - Added harness library for common task types (Appendix A)
  - Total: expanded from 964 to 1600+ lines

Backward compatibility: fully backward compatible with v1.2.
Harness types A-G are unchanged. TYPE B commands for skill upload are unchanged.
Gate line format is unchanged.

Next review: 2027-02-01 or when a new harness type is added.

---

## APPENDIX F: ENVIRONMENT FINGERPRINT FORMAT

Every sealed bundle includes an environment fingerprint.
This allows later reproduction of the exact measurement environment.

Format:
  Environment fingerprint:
    OS: <uname -a or systeminfo output>
    Tool versions:
      python: <python --version>
      paramiko: <pip show paramiko | grep Version>
      ssh: <ssh -V>
      curl: <curl --version | head -1>
    Remote server: <hostname of deployment target>
    Remote OS: <uname -a on remote>
    Measurement time: <YYYY-MM-DD HH:MM:SS UTC>

The environment fingerprint is not required for TYPE A (local code).
It is required for TYPE B (remote upload) and TYPE E (remote API).
It is recommended for all types.

Why: a bundle that shows PASS today may show FAIL tomorrow if the remote
server is updated or the tool versions change. The fingerprint documents
the exact conditions under which the measurement was taken.
---

## APPENDIX G: GATE LINE QUICK REFERENCE

TYPE B skill upload PASS:
  harness:TYPE-B PASS (1630 lines, ASCII text, 0 non-ASCII bytes)

TYPE B skill upload FAIL:
  harness:TYPE-B FAIL cmd 2 (file returned UTF-8 Unicode text, not ASCII text)

TYPE E API PASS:
  harness:TYPE-E PASS (HTTP 200, body matches spec)

TYPE F job PASS:
  harness:TYPE-F PASS (output file 2.7MB, log confirms completion)

TYPE F daemon PASS:
  harness:TYPE-F PASS (PID 12847 running, log shows Monitor daemon started)

Could not measure:
  harness:TYPE-B FAIL cmd 1 (connection refused -- remote server unreachable)

Skipped:
  harness:SKIPPED (draft only, not production artifact)