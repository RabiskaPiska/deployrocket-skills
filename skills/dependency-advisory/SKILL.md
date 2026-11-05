---
name: dependency-advisory
description: Match a project's declared dependencies against public vulnerability advisories before a deploy or a dependency bump. Use when a lockfile, container image, or SBOM exists and nobody has checked it for known advisories. Not a source-code scanner and not a takeover-risk review.
---

# Dependency advisory check

None of the existing security skills do this job. CodeQL and Semgrep read source. SARIF parsing reads a scanner's output. The supply-chain auditor judges whether a dependency looks abandoned or takeable. This skill only answers: does a known public advisory already name a version this project has locked?

## What to scan

- A lockfile or manifest the package manager actually resolves: `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `go.mod` plus `go.sum`, `Cargo.lock`, `requirements.txt` or `uv.lock`, `pom.xml`, `Gemfile.lock`.
- A container image, when the deploy artifact is an image rather than a language lockfile.
- An SBOM, when one already exists. Do not invent one.

Skip dev-only surfaces when the user asks for the production set. Say which set you scanned.

## How to check

1. Prefer [OSV-Scanner](https://github.com/google/osv-scanner), the public CLI for the [OSV](https://osv.dev/) database. A source-tree scan looks like `osv-scanner scan source -r <dir>`. An image scan looks like `osv-scanner scan image <name>:<tag>`.
2. If OSV-Scanner is not installed, query the public OSV API for each locked package name, ecosystem, and version. Do not scrape advisory pages by hand.
3. Record the tool version and the time of the query. Advisory data changes daily; a result without a date is not reusable.

## What to report

For each match, write the package name, the locked version, the advisory id, the affected range, and whether a fixed version is listed. Quote the advisory id from the tool output. Do not invent a CVE number the tool did not return.

Separate three outcomes:

- **Match** — the locked version sits inside a published affected range.
- **No match** — the scanner finished and returned nothing for that ecosystem.
- **Not checked** — no lockfile, the tool was missing, or the ecosystem is outside the scanner's supported set. "Not checked" is not "clean."

## What this skill refuses

- Do not run a package manager's install or update step just to discover the graph. Read the lockfile that is already committed.
- Do not apply the scanner's guided-fix or relock command on a project you do not trust. The OSV-Scanner README warns that `osv-scanner fix` can make the package manager run scripts and contact registries named by the project.
- Do not treat a clean scan as proof the code is safe. It says nothing about vulnerabilities the project's own code introduced, and nothing about typosquat or maintainer-takeover risk.
- Do not copy exploit detail out of an advisory. The report stops at the id, the affected range, and the fixed version.
