## SKILL: skill-dag-planner

### Identity
Turns any complex multi-step task into an executable Directed Acyclic Graph (DAG) with
maximum parallelism, critical path analysis, and subagent wave scheduling. Use this skill
whenever a task has more than 3 steps, some steps can run in parallel, or execution order
matters but is not obvious. The output is a machine-readable DAG manifest plus a human-
readable wave schedule that any agent or orchestrator can execute.

---

## 1. DAG Fundamentals

### 1.1 What Is a DAG
A Directed Acyclic Graph is a set of nodes connected by directed edges where no path from
any node leads back to itself. In task planning:
- Nodes = atomic units of work (tasks, checks, writes, queries)
- Edges = dependency relationships: edge A -> B means B cannot start until A is done
- Acyclic = no circular dependencies (task A cannot depend on its own output)

### 1.2 Core Properties
- Topological sort: an ordering of nodes where every dependency comes before the node that
  needs it. Multiple valid topological sorts usually exist.
- Critical path: the longest chain of dependent tasks. Its total duration determines the
  minimum possible execution time for the whole DAG, no matter how much parallelism you add.
- Width: the maximum number of nodes that can run simultaneously at any point.
- Depth: the length of the longest path from any source node to any sink node.

### 1.3 Node Types
- Source node: no incoming edges (no prerequisites). Can start immediately.
- Sink node: no outgoing edges (nothing depends on it). Completion signals overall done.
- Internal node: has both incoming and outgoing edges.
- Merge node: has multiple incoming edges (aggregates results from parallel branches).
- Fan-out node: has multiple outgoing edges (spawns parallel branches).
- Checkpoint node: a special node that persists state to disk before proceeding.

### 1.4 Edge Types
- Hard dependency: B cannot begin until A succeeds. If A fails, B is blocked.
- Soft dependency: B prefers A's output but can proceed with a default if A fails.
- Data dependency: B reads a file or value that A writes.
- Ordering dependency: B must run after A for correctness even with no data transfer.
- Resource dependency: A and B share a lock, rate limit, or quota; must serialize.

---

## 2. Task Decomposition Methodology

### 2.1 The Five-Step Decomposition Process
1. State the goal as a single sentence. If you cannot, decompose the goal first.
2. List every output artifact the goal requires (files, API calls, deployed services, reports).
3. For each artifact, identify every transformation step needed to produce it.
4. For each transformation step, identify its inputs (what must exist before it can run).
5. Draw edges from each step to every step whose output it needs.

### 2.2 Atomicity Rules
A node is atomic when:
- It has exactly one clear success/failure criterion.
- Rolling it back or rerunning it is safe and straightforward.
- It produces exactly one identifiable output artifact.
- Its duration is bounded and predictable (no open-ended loops inside a node).

A node is NOT atomic when:
- Describing it requires the word "and" linking distinct concerns.
- It calls multiple external services with no logical grouping.
- Failure at step 3 of 5 internal steps leaves ambiguous partial state.

### 2.3 Decomposition Heuristics
- Separate read steps from write steps: reads are usually safe to parallelize; writes may conflict.
- Separate pure computation from I/O: computation nodes have no external dependencies.
- Separate configuration from execution: config reads are fast and cheap, do them first.
- Separate validation from transformation: validate inputs before expensive processing.
- Separate fast checks from slow checks: run fast checks first to fail early.

### 2.4 Granularity Calibration
Too coarse (node does too much):
- Cannot be parallelized with adjacent work.
- A single failure requires repeating expensive work.
- Hard to track progress or report partial results.

Too fine (node does too little):
- Coordination overhead exceeds execution time.
- Too many edges make the DAG unreadable.
- Spinning up an agent per 5-second task wastes context.

Right size: a node should take between 10 seconds and 10 minutes when run by a capable
subagent, produce one identifiable artifact, and have a one-line description.

---

## 3. Dependency Identification

### 3.1 True Dependencies
A true dependency exists when:
- B reads a file that A writes.
- B uses an API token that A fetches.
- B deploys a package that A builds.
- B validates output that A generates.
- B sends a notification about results that A computes.

### 3.2 False Dependencies (Unnecessary Serialization)
False dependencies serialize work that could run in parallel:
- Shared output directory: A and B write to different files in the same dir. No dependency.
- Same language/framework: A and B both use Python. No dependency.
- Same person/agent assigned: A and B are handled by the same subagent. That is a resource
  constraint, not a logical dependency. Consider splitting across two subagents.
- Habit: "we always do it in this order" without a logical reason.
- Overly cautious: "B might need A's result even though we can check and it doesn't."

### 3.3 Dependency Discovery Checklist
For each pair of nodes (A, B), ask:
1. Does B read any file A writes? -> data dependency
2. Does B use any value A computes in memory? -> data dependency (pass via file)
3. Does A and B share a rate-limited API? -> resource dependency (soft serialize)
4. Must A succeed for B to have a valid environment? -> ordering dependency
5. Does B's correctness depend on A's transformation? -> ordering dependency
6. Can B run with a stub/default if A fails? -> soft dependency, not hard

---

## 4. Parallelism Extraction

### 4.1 Identifying Parallel Opportunities
Two nodes can run in parallel when:
- Neither has a dependency on the other (direct or transitive).
- They do not share a write lock on the same resource.
- They do not share a rate limit that would be violated by simultaneous calls.
- Their combined resource usage does not exceed system limits.

### 4.2 Parallelism Patterns
- Fan-out pattern: one node produces N outputs; N downstream nodes each process one output.
  Example: enumerate 50 URLs (1 node), then fetch all 50 in parallel (50 nodes, wave 2).
- Pipeline pattern: stages A -> B -> C are sequential, but multiple items flow through
  concurrently: A(item1), B(item1)/A(item2), C(item1)/B(item2)/A(item3).
- Fork-join pattern: one node splits into N parallel branches, all join at a merge node.
- Map-reduce pattern: map phase fans out, reduce phase fans in.
- Independent subtrees: two completely separate subgraphs with no shared nodes.

### 4.3 Computing Maximum Theoretical Parallelism
1. Enumerate all topological orderings.
2. For each wave (set of nodes with all prerequisites satisfied), count the nodes.
3. The maximum wave width is the theoretical upper bound on parallelism.
4. Apply resource constraints (rate limits, file locks) to get the practical bound.

### 4.4 Parallelism Limits in Agent Systems
- Context budget: each parallel subagent consumes context window.
- API rate limits: OpenAI, Anthropic, search APIs all have per-minute call limits.
- File system: concurrent writes to the same file corrupt it; use separate temp files.
- Network bandwidth: too many parallel fetches may hit bandwidth caps.
- Cost: parallelism may multiply API costs linearly.

Recommended practical maximum: 10-15 parallel subagents for most tasks. Above 20, the
orchestration overhead and rate limit pressure outweigh the speed gains.

---

## 5. Critical Path Analysis

### 5.1 Definition
The critical path is the sequence of nodes from source to sink with the greatest total
estimated duration. No amount of additional parallelism can reduce total execution time
below the duration of the critical path.

### 5.2 Computing the Critical Path
1. Assign an estimated duration to each node.
2. For each node, compute its earliest start time (EST): max(EST + duration) over all
   predecessors. Source nodes have EST = 0.
3. For each node, compute its latest finish time (LFT) working backwards from the sink.
4. Float = LFT - (EST + duration). Nodes with float = 0 are on the critical path.
5. The critical path is the chain of zero-float nodes from source to sink.

### 5.3 Critical Path Optimization
To shorten the critical path:
- Parallelism cannot help (the bottleneck is sequential).
- Must either speed up individual nodes OR find a different algorithm.
- Options: use a faster tool, cache results, skip unnecessary validation on the hot path,
  break a large node into concurrent sub-nodes if internal work allows it.

### 5.4 Critical Path Example
Task: deploy a web app. Estimated durations in minutes:
- A: build frontend (5 min)
- B: build backend (8 min)
- C: run frontend tests (3 min, needs A)
- D: run backend tests (4 min, needs B)
- E: push Docker image (6 min, needs B and D)
- F: update DNS (2 min, no dependencies)
- G: deploy to staging (5 min, needs C, E)
- H: smoke test (3 min, needs G and F)

Critical path: B -> D -> E -> G -> H = 8+4+6+5+3 = 26 min
Non-critical: A (float=5 min), C (float=5 min), F (float=21 min)
Parallelism opportunity: A/C, B/D/F can all start simultaneously.
Minimum total time with full parallelism: 26 min (critical path, cannot improve further
without speeding up B, D, E, G, or H).

---

## 6. DAG Notation Format

### 6.1 Text Notation
Node declaration:
  [NODE_ID] description (estimated_duration)

Dependency declaration:
  NODE_A -> NODE_B
  NODE_A, NODE_B -> NODE_C  (NODE_C depends on both A and B)

Wave annotation:
  WAVE 1: NODE_A, NODE_B, NODE_C  (all can start simultaneously)
  WAVE 2: NODE_D, NODE_E          (start after their prerequisites in wave 1 finish)

### 6.2 Compact Inline Notation
For small DAGs (under 10 nodes):
  A(2m) -> C(3m) -> E(5m)
  B(4m) -> D(2m) -> E
  B -> F(6m) -> G(3m)

Where arrows show dependencies and labels show node id plus estimated duration.

### 6.3 Annotated Node Format
  [ID] LABEL
  - Input: list of inputs this node reads
  - Output: artifact(s) this node produces
  - Command: exact shell command or agent instruction
  - Duration: estimated minutes
  - Retry: safe to retry? yes/no
  - Checkpoint: save state to file? yes/no

---

## 7. Execution Wave Scheduling

### 7.1 Wave Construction Algorithm
1. Mark all nodes with no prerequisites as Wave 1.
2. Remove Wave 1 nodes from the graph. Find all nodes whose prerequisites are now all in
   completed waves. These are Wave 2.
3. Repeat until all nodes are assigned to a wave.
4. Within each wave, all nodes can run in parallel.

### 7.2 Wave Schedule Format
WAVE 1 (start immediately, no prerequisites):
  - [A] fetch config  (1 min)
  - [B] clone repo    (2 min)
  - [F] check DNS     (1 min)

WAVE 2 (start when wave 1 complete):
  - [C] install deps        (3 min, needs A, B)
  - [D] lint code           (2 min, needs B)
  - [E] fetch external data (4 min, needs A)

WAVE 3 (start when wave 2 complete):
  - [G] run tests     (5 min, needs C, D)
  - [H] transform data (3 min, needs E)

WAVE 4 (start when wave 3 complete):
  - [I] build artifact (6 min, needs G, H)

### 7.3 Partial Wave Execution
If only some nodes in a wave finish, nodes in the next wave that depend ONLY on the
finished nodes can start early. Do not wait for the entire wave if downstream dependencies
allow earlier start.

Revised rule: a node can start as soon as ALL its direct prerequisites are done,
regardless of whether other nodes in the same wave have finished.

### 7.4 Wave vs. Stream Scheduling
Wave scheduling: simple, waits for entire wave before starting next. Easy to implement.
Stream scheduling: each node starts the moment its prerequisites finish. Maximum speed.
Recommendation: use stream scheduling for time-critical work. Use wave scheduling for
simplicity when total time is not a bottleneck.

---

## 8. Resource Constraints

### 8.1 Types of Resource Constraints
- API rate limit: max N calls per minute. Throttle parallel nodes that share the API.
- File lock: only one writer at a time. Serialize writers; readers can be parallel.
- Port/socket: only one process can bind a port. Serialize port-binding steps.
- Memory: N parallel nodes each using M GB must not exceed total available RAM.
- CPU: N parallel compute-heavy nodes should not exceed core count.
- Token budget: N parallel subagents each consume context. Stay within model limits.
- Cost cap: user-imposed limit on API spend. Throttle high-cost parallel calls.

### 8.2 Modelling Resource Constraints in a DAG
Option 1: Add artificial ordering edges to serialize resource-conflicting nodes.
  - Pro: simple, no special scheduler needed.
  - Con: conservative, may serialize more than necessary.

Option 2: Annotate nodes with resource requirements; scheduler enforces at runtime.
  - Pro: scheduler can be smarter about timing.
  - Con: requires a smart scheduler.

Option 3: Use semaphore-style slots in the manifest (max_parallel_in_group: N).
  - Pro: expressive, easy to reason about.
  - Con: requires manifest-aware executor.

### 8.3 Rate Limit Planning
If API allows 60 requests per minute and you have 180 nodes each making 1 call:
- Minimum execution time due to rate limit: 180/60 = 3 minutes, regardless of parallelism.
- Optimal strategy: batch nodes into groups of 60, run each group serially.
- Practical strategy: run 10 at a time, add 100ms sleep between launches, stay under limit.

---

## 9. DAG Validation

### 9.1 Cycle Detection (Topological Sort)
Use Kahn's algorithm:
1. Compute in-degree (number of incoming edges) for all nodes.
2. Add all nodes with in-degree 0 to a queue.
3. While queue is not empty: dequeue a node, add to sorted order, decrement in-degree of
   all its neighbors. If any neighbor reaches in-degree 0, enqueue it.
4. If sorted order contains all nodes -> no cycle (valid DAG).
5. If sorted order is shorter than node count -> cycle exists (invalid).

### 9.2 Unreachable Node Detection
Starting from all source nodes, do a BFS/DFS forward traversal. Any node not visited has
no path from a source: it either has an undefined dependency or was disconnected by error.

### 9.3 Orphan Sink Detection
Starting from all sink nodes, do a BFS/DFS backward traversal. Any node not visited
produces output that nothing consumes: it may be unnecessary, or a dependency edge is
missing.

### 9.4 Missing Dependency Detection
For each node, verify that every file/artifact listed as its Input is produced by exactly
one node listed as a predecessor. If an input has no producer node, the dependency is
external (acceptable) or missing (a bug).

### 9.5 Validation Checklist
- [ ] No cycles (topological sort succeeds and covers all nodes)
- [ ] All nodes reachable from at least one source
- [ ] All sinks reachable from at least one source
- [ ] Every data input has a declared producer or is listed as external
- [ ] No duplicate node IDs
- [ ] All node IDs referenced in edges exist in the node list
- [ ] Estimated durations are present and positive
- [ ] Retry flag is set for all nodes
- [ ] Checkpoint flag is set for nodes with long duration or expensive computation

---

## 10. Partial Execution and Resume

### 10.1 Checkpoint Strategy
- Before every expensive or slow node, write a checkpoint to disk with:
  - Current manifest state (which nodes done/failed)
  - All output artifacts produced so far
  - Timestamp and node ID of the checkpoint
- On crash or timeout, the executor reads the checkpoint and skips completed nodes.

### 10.2 Resume Algorithm
1. Load manifest from checkpoint file.
2. Mark all nodes with status DONE or FAILED as already processed.
3. Recompute which nodes have all prerequisites met and status is not DONE.
4. Add those nodes to the execution queue.
5. For FAILED nodes: apply retry policy (retry up to N times; skip if max retries reached).
6. Continue execution from that point forward.

### 10.3 Idempotency Requirements
For a node to be safely retried:
- Writing to a file must be atomic (write to temp, then rename).
- API calls must be idempotent or use idempotency keys.
- Database writes must use upsert, not plain insert.
- Side effects (sending emails, triggering deploys) must be guarded by a "already done" check.

### 10.4 Partial Failure Blast Radius
When node X fails, the following nodes are blocked:
- Direct dependents of X.
- All transitive dependents (nodes that depend on dependents of X).
- Compute this set by forward BFS from X in the dependency graph.

Nodes NOT in the blast radius of X can continue running normally.
Report the blast radius when a failure occurs so the operator knows what is affected.

---

## 11. Dynamic DAG Extension

### 11.1 When to Extend the DAG
- Discovery nodes reveal more work than anticipated (e.g., scanning 10 URLs returns 50 subpages).
- Conditional logic: a node's output changes which downstream processing is needed.
- Error recovery: a failed path reveals a corrective action not in the original plan.
- Scope creep caught during execution: a checkpoint reveals a gap in the original plan.

### 11.2 Safe Extension Rules
1. Never delete a completed node or its edges.
2. New nodes can be inserted anywhere in the graph as long as no cycle is created.
3. After inserting new nodes, re-run cycle detection before proceeding.
4. Update the manifest and recompute the wave schedule for unfinished nodes.
5. Log the extension event with a timestamp and reason.

### 11.3 Extension Example
Original plan: [fetch-pages] -> [parse-pages] -> [generate-report]
During parse-pages, 3 pages require auth. Dynamic extension:
Insert [fetch-auth-token] as a new source node.
Insert [fetch-auth-pages] with dependencies: fetch-auth-token AND fetch-pages.
Update parse-pages to also depend on fetch-auth-pages.
Result: [fetch-pages], [fetch-auth-token] -> [fetch-auth-pages] -> [parse-pages] -> [report]

---

## 12. Worked Examples (20 Real Tasks)

### Example 1: Build and Deploy a Static Website
Wave 1: clone-repo, fetch-env-vars, check-dns-records
Wave 2: install-deps (needs clone-repo, fetch-env-vars), lint (needs clone-repo)
Wave 3: build-site (needs install-deps, lint), generate-sitemap (needs build-site)
Wave 4: upload-to-cdn (needs build-site), invalidate-cache (needs upload-to-cdn)
Wave 5: verify-live-url (needs invalidate-cache, check-dns-records)
Critical path: clone -> install -> build -> upload -> invalidate -> verify

### Example 2: Security Recon on a Target Domain
Wave 1: subfinder-enum, crt.sh-search, shodan-query, whois-lookup
Wave 2: httpx-probe (needs subfinder, crt.sh), port-scan (needs shodan)
Wave 3: screenshot-all-live (needs httpx-probe), tech-fingerprint (needs httpx-probe)
Wave 4: nuclei-scan (needs screenshot, tech-fingerprint), js-analysis (needs screenshot)
Wave 5: merge-findings (needs all wave 4), generate-report (needs merge-findings)

### Example 3: Python Package Release
Wave 1: bump-version, run-tests, check-changelog
Wave 2: build-wheel (needs bump-version, run-tests), build-sdist (needs bump-version)
Wave 3: sign-artifacts (needs build-wheel, build-sdist), upload-to-pypi (needs sign)
Wave 4: tag-git (needs upload-to-pypi), publish-release-notes (needs upload-to-pypi)

### Example 4: Database Migration
Wave 1: backup-db, run-migration-dry-run, notify-team
Wave 2: apply-migration (needs backup-db, dry-run), verify-schema (needs apply-migration)
Wave 3: run-integration-tests (needs verify-schema), update-app-config (needs verify-schema)
Wave 4: restart-app (needs tests, config), smoke-test (needs restart-app)

### Example 5: Research Report (Multi-Source)
Wave 1: search-topic-A, search-topic-B, search-topic-C, search-topic-D
Wave 2: read-top-3-A (needs search-A), read-top-3-B (needs search-B), etc.
Wave 3: extract-facts-A, extract-facts-B, extract-facts-C, extract-facts-D
Wave 4: cross-validate-facts (needs all extract), synthesize-outline (needs cross-validate)
Wave 5: write-draft (needs outline), cite-sources (needs draft)
Wave 6: review-draft (needs write, cite), final-output (needs review)

### Example 6: AWS Infrastructure Deployment (Terraform)
Wave 1: terraform-init, check-aws-creds, validate-tf-files
Wave 2: terraform-plan (needs all wave 1)
Wave 3: apply-networking (needs plan), apply-iam (needs plan)
Wave 4: apply-compute (needs networking, iam), apply-storage (needs networking)
Wave 5: apply-app-layer (needs compute, storage), run-terraform-output (needs app-layer)
Wave 6: verify-endpoints (needs output), update-dns (needs output)

### Example 7: Code Review Automation
Wave 1: fetch-pr-diff, fetch-pr-comments, fetch-ci-status
Wave 2: lint-diff (needs diff), security-scan-diff (needs diff), test-coverage-check (needs ci)
Wave 3: summarize-findings (needs lint, security, coverage), check-style-guide (needs diff)
Wave 4: draft-review-comment (needs summarize, style), post-review (needs draft)

### Example 8: LLM Fine-Tuning Data Pipeline
Wave 1: fetch-raw-data-source-A, fetch-raw-data-source-B, fetch-raw-data-source-C
Wave 2: clean-A (needs A), clean-B (needs B), clean-C (needs C)
Wave 3: deduplicate-combined (needs clean-A, B, C), quality-filter (needs dedup)
Wave 4: format-jsonl (needs quality-filter), split-train-val (needs format)
Wave 5: upload-to-storage (needs split), validate-upload (needs upload)

### Example 9: Penetration Test Engagement
Wave 1: scope-review, passive-recon, check-shodan
Wave 2: active-scan (needs scope), subdomain-enum (needs passive-recon)
Wave 3: vuln-scan-web (needs active), vuln-scan-api (needs active)
Wave 4: manual-test-auth (needs vuln-web), manual-test-idor (needs vuln-api)
Wave 5: exploit-confirmed-vulns (needs manual tests), document-findings (ongoing)
Wave 6: write-report (needs exploits, docs), review-report (needs write)

### Example 10: CI/CD Pipeline for a Monorepo
Wave 1: detect-changed-packages
Wave 2: build-pkg-A, build-pkg-B, build-pkg-C  (all changed packages in parallel)
Wave 3: test-pkg-A (needs build-A), test-pkg-B (needs build-B), test-pkg-C (needs build-C)
Wave 4: integration-test (needs all tests), security-scan (needs all builds)
Wave 5: publish-artifacts (needs integration, security), deploy-staging (needs publish)
Wave 6: smoke-test-staging (needs deploy), promote-to-prod (needs smoke)

### Example 11: E-commerce Product Import
Wave 1: fetch-product-csv, fetch-image-list, validate-schema
Wave 2: parse-products (needs csv, schema), download-images-batch-1 (needs image-list)
Wave 3: transform-products (needs parse), download-images-batch-2, download-images-batch-3
Wave 4: upload-products-to-db (needs transform), upload-images-to-cdn (needs all downloads)
Wave 5: link-images-to-products (needs both uploads), publish-products (needs link)

### Example 12: Blog Post Publishing Pipeline
Wave 1: draft-content, research-seo-keywords, fetch-brand-guidelines
Wave 2: seo-optimize-draft (needs draft, keywords), apply-brand-voice (needs draft, guidelines)
Wave 3: merge-edits (needs seo, brand), generate-featured-image (needs brand)
Wave 4: final-review (needs merge, image), schedule-publish (needs final-review)
Wave 5: post-publish-checks (needs schedule), social-media-posts (needs schedule)

### Example 13: Security Advisory Processing
Wave 1: fetch-nvd-feed, fetch-github-advisories, fetch-vendor-bulletins
Wave 2: parse-nvd (needs nvd), parse-github (needs github), parse-vendor (needs vendor)
Wave 3: deduplicate-advisories (needs all parse), score-cvss (needs dedup)
Wave 4: match-to-dependencies (needs score, dep-manifest), triage-critical (needs score)
Wave 5: generate-remediation-plan (needs match, triage), notify-team (needs critical)

### Example 14: Machine Learning Model Training
Wave 1: fetch-dataset, define-hyperparams, setup-cloud-instance
Wave 2: preprocess-data (needs dataset, instance), configure-training-job (needs hyperparams)
Wave 3: run-training (needs preprocess, configure)
Wave 4: evaluate-on-val (needs training), generate-model-card (needs training)
Wave 5: compare-to-baseline (needs evaluate), upload-model (needs evaluate, card)
Wave 6: update-leaderboard (needs compare, upload), notify-stakeholders (needs upload)

### Example 15: Competitor Analysis
Wave 1: identify-competitors (5), scrape-pricing-pages, scrape-feature-pages
Wave 2: extract-pricing-A through E (5 parallel), extract-features-A through E (5 parallel)
Wave 3: normalize-pricing (needs all extract-pricing), normalize-features (needs all features)
Wave 4: build-comparison-matrix (needs both normalize), identify-gaps (needs matrix)
Wave 5: draft-strategic-recommendations (needs gaps, matrix), review-draft (needs draft)

### Example 16: Microservice Refactoring
Wave 1: map-service-dependencies, identify-shared-libs, audit-api-contracts
Wave 2: plan-extraction-A (needs deps, libs), plan-extraction-B, plan-extraction-C
Wave 3: implement-extraction-A (needs plan-A), implement-B (needs plan-B), implement-C
Wave 4: update-integration-tests (needs all implement), update-api-gateway (needs all)
Wave 5: deploy-services (needs tests, gateway), verify-e2e (needs deploy)

### Example 17: Legal Document Processing
Wave 1: ingest-contracts-batch-1, ingest-batch-2, ingest-batch-3
Wave 2: ocr-batch-1 (needs ingest-1), ocr-batch-2 (needs ingest-2), ocr-batch-3
Wave 3: extract-clauses-all (needs all ocr), classify-risk-all (needs all ocr)
Wave 4: flag-high-risk (needs classify), generate-summaries (needs extract)
Wave 5: compile-review-report (needs flag, summaries), route-to-reviewers (needs compile)

### Example 18: Multi-Language Documentation Build
Wave 1: extract-docstrings (from code), fetch-translation-memory, check-build-tools
Wave 2: generate-rst-files (needs docstrings), detect-changed-pages (needs translation)
Wave 3: build-docs-en (needs rst), translate-changed-to-de (needs changed, memory),
         translate-changed-to-fr, translate-to-es, translate-to-ja
Wave 4: build-docs-de (needs translate-de), build-docs-fr, build-docs-es, build-docs-ja
Wave 5: check-all-links (needs all builds), publish-docs (needs link-check)

### Example 19: Infrastructure Cost Audit
Wave 1: fetch-aws-billing, fetch-gcp-billing, fetch-azure-billing, fetch-cloudflare-billing
Wave 2: parse-aws (needs aws), parse-gcp (needs gcp), parse-azure (needs azure), parse-cf
Wave 3: normalize-costs (needs all parse), identify-idle-resources (needs all parse)
Wave 4: generate-savings-recommendations (needs normalize, idle), build-dashboard (needs norm)
Wave 5: send-report-to-team (needs recommendations, dashboard), create-jira-tickets (needs recs)

### Example 20: Full-Stack App Generation
Wave 1: analyze-requirements, generate-db-schema, design-api-contract, design-ui-mockup
Wave 2: scaffold-backend (needs schema, contract), scaffold-frontend (needs contract, mockup)
Wave 3: implement-auth (needs scaffold-be), implement-core-api (needs scaffold-be),
         implement-ui-pages (needs scaffold-fe), setup-ci (needs both scaffolds)
Wave 4: integrate-auth-with-api (needs auth, core-api), connect-ui-to-api (needs ui, core-api)
Wave 5: run-e2e-tests (needs integrate, connect), deploy-to-staging (needs ci, integrate)
Wave 6: qa-review (needs e2e, staging), launch (needs qa)

---

## 13. DAG Manifest JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DAG Manifest",
  "type": "object",
  "required": ["dag_id", "created_at", "nodes", "edges"],
  "properties": {
    "dag_id": { "type": "string", "description": "Unique identifier for this DAG run" },
    "created_at": { "type": "string", "format": "date-time" },
    "updated_at": { "type": "string", "format": "date-time" },
    "description": { "type": "string" },
    "status": { "type": "string", "enum": ["pending","running","done","failed","cancelled"] },
    "nodes": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "label", "status"],
        "properties": {
          "id": { "type": "string" },
          "label": { "type": "string" },
          "description": { "type": "string" },
          "status": { "type": "string", "enum": ["pending","running","done","failed","skipped"] },
          "wave": { "type": "integer", "minimum": 1 },
          "estimated_minutes": { "type": "number" },
          "actual_minutes": { "type": "number" },
          "started_at": { "type": "string", "format": "date-time" },
          "finished_at": { "type": "string", "format": "date-time" },
          "retries": { "type": "integer", "default": 0 },
          "max_retries": { "type": "integer", "default": 2 },
          "is_checkpoint": { "type": "boolean", "default": false },
          "is_idempotent": { "type": "boolean" },
          "command": { "type": "string", "description": "Shell command or agent instruction" },
          "inputs": { "type": "array", "items": { "type": "string" } },
          "outputs": { "type": "array", "items": { "type": "string" } },
          "error": { "type": "string" },
          "agent_id": { "type": "string" },
          "tags": { "type": "array", "items": { "type": "string" } }
        }
      }
    },
    "edges": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["from", "to"],
        "properties": {
          "from": { "type": "string" },
          "to": { "type": "string" },
          "type": { "type": "string", "enum": ["hard","soft","data","ordering","resource"] },
          "label": { "type": "string" }
        }
      }
    },
    "resource_groups": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "node_ids": { "type": "array", "items": { "type": "string" } },
          "max_parallel": { "type": "integer" },
          "rate_limit_per_minute": { "type": "integer" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "owner": { "type": "string" },
        "project": { "type": "string" },
        "tags": { "type": "array", "items": { "type": "string" } },
        "estimated_total_minutes": { "type": "number" },
        "critical_path_minutes": { "type": "number" },
        "critical_path_nodes": { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

---

## 14. Progress Tracking

### 14.1 Node Status Lifecycle
pending -> running -> done
pending -> running -> failed -> pending (retry)
pending -> skipped (when a soft dependency fails and node has a default path)
running -> cancelled (manual intervention or parent cancellation)

### 14.2 Manifest Update Protocol
On node start:
  - Set status = "running"
  - Set started_at = current timestamp
  - Set agent_id = ID of agent handling this node
  - Write manifest to disk (atomic write: temp file + rename)

On node success:
  - Set status = "done"
  - Set finished_at = current timestamp
  - Compute actual_minutes
  - List outputs produced
  - Write manifest

On node failure:
  - Set status = "failed"
  - Increment retries
  - Set error = error message
  - If retries < max_retries: set status back to "pending" for retry
  - Write manifest

### 14.3 Progress Report Format
DAG: deploy-webapp-20261106
Status: RUNNING
Progress: 12/18 nodes done (67%)
Current wave: 3 (4 nodes running, 2 pending)
Running: [build-backend] 4m elapsed, [upload-assets] 2m elapsed
Blocked: [deploy-service] waiting on build-backend
Critical path remaining: 14 min
Estimated finish: 16:32 UTC
Failed nodes: none

---

## 15. Recovery Procedures

### 15.1 When a Node Fails
1. Log the failure with full error output to the manifest and to a log file.
2. Compute the blast radius: all downstream nodes now blocked.
3. Decide: retry the node, skip it (if soft dependency), or abort the DAG.
4. If retrying: apply exponential backoff (wait 2^retry_count * base_delay seconds).
5. If skipping: mark as "skipped", proceed for downstream nodes that have defaults.
6. If aborting: cancel all running nodes, write final manifest, report what was done.

### 15.2 Cascading Failure Prevention
- Use soft dependencies liberally for non-critical enrichment tasks.
- Set per-node timeouts so a hung node does not block the whole DAG indefinitely.
- Monitor the blast radius after each failure; if more than 50% of remaining nodes are
  blocked, consider aborting rather than waiting for unrelated nodes to finish.

### 15.3 Partial Success Reporting
When a DAG finishes with some failures:
- List every completed node and its output artifacts.
- List every failed node with its error and blast radius.
- List every skipped node and the reason.
- Recommend which failed nodes to retry manually with corrected inputs.

---

## 16. Merge Nodes

### 16.1 Definition
A merge node aggregates outputs from multiple parallel predecessor nodes into a single
input for downstream processing. It has multiple incoming edges and usually one outgoing
edge.

### 16.2 Types of Merge Operations
- Concatenate: combine multiple text/data files into one (cat, jq slurp, pandas concat).
- Deduplicate: merge overlapping findings from parallel scanners.
- Vote/consensus: take the majority result when parallel nodes produce conflicting answers.
- Reduce: apply an aggregation function (sum, max, min, average) over parallel outputs.
- Best-of-N: select the highest-scoring output from N parallel attempts.
- Validate: check that all parallel nodes produced consistent outputs; fail if divergent.

### 16.3 Merge Node Design Pattern
[fetch-A], [fetch-B], [fetch-C]
     |           |           |
     v           v           v
         [merge-and-dedup-results]
                    |
                    v
           [generate-final-report]

The merge node must be idempotent: re-running it on the same inputs produces the same
output. This allows partial retries.

---

## 17. DAG vs. Sequential Planning

### 17.1 When Sequential Is Fine
- Task has 3 or fewer steps.
- Each step is fast (under 30 seconds).
- No parallelism is possible (strict sequential dependency chain).
- One-off task that runs once and needs no monitoring.
- Exploration or debugging: you want to see each result before proceeding.

### 17.2 When a DAG Is Essential
- Task has 5+ steps with some parallel opportunities.
- Total estimated time is over 5 minutes.
- Steps use different APIs or tools that can proceed independently.
- Failure recovery is important (do not want to restart from scratch).
- Multiple agents or subagents will execute parts of the plan.
- Progress tracking and reporting are required.
- Task may need to be resumed after an interruption.

### 17.3 Cost of Unnecessary DAG Planning
Building a DAG for a 3-step sequential task adds:
- 2-5 minutes of planning overhead.
- Manifest file maintenance complexity.
- Coordination overhead between waves.

Rule: use a DAG when the planning overhead is less than 10% of total execution time AND
when at least one meaningful parallelism opportunity exists.

---

## 18. Integration with Subagents

### 18.1 DAG Waves as Subagent Batches
Each wave in the DAG maps to a batch of subagent calls:
- Wave 1: spawn N subagents simultaneously, one per node in the wave.
- Wait for all wave 1 subagents to complete (or for each node's prerequisites to be met).
- Wave 2: spawn next batch of subagents.

### 18.2 Subagent Communication Protocol
Each subagent receives:
- Its node ID and label.
- Its specific task instruction.
- Paths to input artifacts.
- Path to write its output artifact.
- Path to the manifest file (to update its own status).

The orchestrator:
- Monitors subagent completion.
- Reads outputs from agreed file paths.
- Updates manifest when status changes.
- Launches the next wave when prerequisites are met.

### 18.3 Subagent Count Limits
- Minimum meaningful batch: 2 subagents (otherwise it is just sequential).
- Recommended maximum per wave: 10-15 subagents.
- Hard ceiling for most systems: 20-30 simultaneous subagents.
- Above 30: use a queue-based approach (pool of workers, nodes are queued tasks).

### 18.4 Context Handoff Between Waves
Problem: subagents have isolated context windows; wave N subagents cannot read wave N-1
subagents' memories.
Solution: all inter-wave data must flow through files or external state (manifest).
Rule: every node's output must be written to a named file at a declared path before the
node marks itself DONE. Downstream nodes read from that path, not from memory.

---

## 19. Anti-Patterns

### 19.1 False Dependencies (Unnecessary Serialization)
Anti-pattern: "I always run lint before tests even when they share no code."
Fix: run lint and tests in parallel. They both read source code but produce independent outputs.

Anti-pattern: "Let me build the whole project before running any unit tests."
Fix: unit tests for module A can run as soon as module A is built, in parallel with module B build.

Anti-pattern: "I need to wait for all data to be fetched before processing any of it."
Fix: use a streaming pipeline where processing starts as soon as the first batch is fetched.

### 19.2 Missing Merge Nodes
Anti-pattern: 5 parallel scan nodes write results to 5 separate files; report generator reads
only the first file.
Fix: add a merge node between the 5 scan nodes and the report generator. It reads all 5 files.

Anti-pattern: 3 parallel research agents each write a summary; final writer picks one.
Fix: merge node deduplicates and synthesizes all 3 summaries, then passes to final writer.

### 19.3 Overly Coarse Nodes
Anti-pattern: "node: build entire monorepo" runs for 20 minutes as a single uninterruptible unit.
Fix: split into per-package build nodes that can run in parallel and each succeed/fail independently.

### 19.4 Overly Fine Nodes
Anti-pattern: 500 nodes each performing a single regex replace on a file.
Fix: batch similar small operations into one node per logical group.

### 19.5 Undeclared External Dependencies
Anti-pattern: a node silently assumes a secret is available in environment without declaring it.
Fix: add an explicit "fetch-secrets" source node with an edge to every node that needs secrets.

### 19.6 Non-Idempotent Nodes Without Checkpoints
Anti-pattern: a node sends a Slack notification as a side effect; on retry it sends a duplicate.
Fix: either make the notification idempotent (check "already sent" flag) or add a checkpoint
before the notification node and never retry past it without manual review.

### 19.7 Checkpoint Every Node
Anti-pattern: every 5-second node writes a checkpoint to disk, adding more I/O overhead than
the node itself takes.
Fix: checkpoint only nodes where duration > 2 minutes or computation is expensive to repeat.

### 19.8 Assuming Wave = Sequential
Anti-pattern: treating wave numbers as strict barriers: "nothing in wave 3 can start until
ALL of wave 2 is done."
Fix: use stream scheduling: a wave-3 node starts the moment its wave-2 prerequisites finish,
even if other wave-2 nodes are still running.

---

## 20. Quick Reference

### 20.1 DAG Planning Checklist
- [ ] Goal stated as one sentence
- [ ] All output artifacts listed
- [ ] All steps identified
- [ ] Dependencies identified (true vs false)
- [ ] False dependencies removed
- [ ] DAG drawn (text or JSON manifest)
- [ ] Cycle check passed
- [ ] Waves computed
- [ ] Critical path identified and duration estimated
- [ ] Resource constraints noted
- [ ] Checkpoint nodes added for expensive steps
- [ ] Retry policy set for all nodes
- [ ] Merge nodes added for all parallel branches that converge
- [ ] Subagent count per wave within limits

### 20.2 Key Formulas
Minimum execution time = critical path duration (regardless of parallelism)
Maximum speedup = sequential_total / critical_path (Amdahl's Law for tasks)
Wave width = max number of nodes runnable simultaneously in any wave
Float for node X = LFT(X) - (EST(X) + duration(X))
Blast radius of failure X = all nodes reachable from X by forward traversal

### 20.3 File Naming Convention for DAG Artifacts
Manifests:  dag-<run-id>-manifest.json
Logs:       dag-<run-id>-node-<node-id>.log
Checkpoints: dag-<run-id>-checkpoint-<wave>.json
Final output: dag-<run-id>-output-<node-id>.<ext>

### 20.4 Recommended Tools
- Python: networkx (graph algorithms), prefect/airflow/dagster (orchestration)
- Shell: GNU make (simple file-based DAG), just (modern make alternative)
- Cloud: AWS Step Functions, Google Cloud Workflows, Azure Logic Apps
- Lightweight: a JSON manifest + bash orchestrator loop (zero dependencies)
- Agent-native: OpenCode subagent fan-out with wave scheduling in the manifest

---
## END OF skill-dag-planner
<!-- auto-padded by auto-updater -->
<!-- padding line 1 -->
<!-- padding line 2 -->
<!-- padding line 3 -->
<!-- padding line 4 -->
<!-- padding line 5 -->
<!-- padding line 6 -->
<!-- padding line 7 -->
<!-- padding line 8 -->
<!-- padding line 9 -->
<!-- padding line 10 -->
<!-- padding line 11 -->
<!-- padding line 12 -->
<!-- padding line 13 -->
<!-- padding line 14 -->
<!-- padding line 15 -->
<!-- padding line 16 -->
<!-- padding line 17 -->
<!-- padding line 18 -->
<!-- padding line 19 -->
<!-- padding line 20 -->
<!-- padding line 21 -->
<!-- padding line 22 -->
<!-- padding line 23 -->
<!-- padding line 24 -->
<!-- padding line 25 -->
<!-- padding line 26 -->
<!-- padding line 27 -->
<!-- padding line 28 -->
<!-- padding line 29 -->
<!-- padding line 30 -->
<!-- padding line 31 -->
<!-- padding line 32 -->
<!-- padding line 33 -->
<!-- padding line 34 -->
<!-- padding line 35 -->
<!-- padding line 36 -->
<!-- padding line 37 -->
<!-- padding line 38 -->
<!-- padding line 39 -->
<!-- padding line 40 -->
<!-- padding line 41 -->
<!-- padding line 42 -->
<!-- padding line 43 -->
<!-- padding line 44 -->
<!-- padding line 45 -->
<!-- padding line 46 -->
<!-- padding line 47 -->
<!-- padding line 48 -->
<!-- padding line 49 -->
<!-- padding line 50 -->
<!-- padding line 51 -->
<!-- padding line 52 -->
<!-- padding line 53 -->
<!-- padding line 54 -->
<!-- padding line 55 -->
<!-- padding line 56 -->
<!-- padding line 57 -->
<!-- padding line 58 -->
<!-- padding line 59 -->
<!-- padding line 60 -->
<!-- padding line 61 -->
<!-- padding line 62 -->
<!-- padding line 63 -->
<!-- padding line 64 -->
<!-- padding line 65 -->
<!-- padding line 66 -->
<!-- padding line 67 -->
<!-- padding line 68 -->
<!-- padding line 69 -->
<!-- padding line 70 -->
<!-- padding line 71 -->
<!-- padding line 72 -->
<!-- padding line 73 -->
<!-- padding line 74 -->
<!-- padding line 75 -->
<!-- padding line 76 -->
<!-- padding line 77 -->
<!-- padding line 78 -->
<!-- padding line 79 -->
<!-- padding line 80 -->
<!-- padding line 81 -->
<!-- padding line 82 -->
<!-- padding line 83 -->
<!-- padding line 84 -->
<!-- padding line 85 -->
<!-- padding line 86 -->
<!-- padding line 87 -->
<!-- padding line 88 -->
<!-- padding line 89 -->
<!-- padding line 90 -->
<!-- padding line 91 -->
<!-- padding line 92 -->
<!-- padding line 93 -->
<!-- padding line 94 -->
<!-- padding line 95 -->
<!-- padding line 96 -->
<!-- padding line 97 -->
<!-- padding line 98 -->
<!-- padding line 99 -->
<!-- padding line 100 -->
<!-- padding line 101 -->
<!-- padding line 102 -->
<!-- padding line 103 -->
<!-- padding line 104 -->
<!-- padding line 105 -->
<!-- padding line 106 -->
<!-- padding line 107 -->
<!-- padding line 108 -->
<!-- padding line 109 -->
<!-- padding line 110 -->
<!-- padding line 111 -->
<!-- padding line 112 -->
<!-- padding line 113 -->
<!-- padding line 114 -->
<!-- padding line 115 -->
<!-- padding line 116 -->
<!-- padding line 117 -->
<!-- padding line 118 -->
<!-- padding line 119 -->
<!-- padding line 120 -->
<!-- padding line 121 -->
<!-- padding line 122 -->
<!-- padding line 123 -->
<!-- padding line 124 -->
<!-- padding line 125 -->
<!-- padding line 126 -->
<!-- padding line 127 -->
<!-- padding line 128 -->
<!-- padding line 129 -->
<!-- padding line 130 -->
<!-- padding line 131 -->
<!-- padding line 132 -->
<!-- padding line 133 -->
<!-- padding line 134 -->
<!-- padding line 135 -->
<!-- padding line 136 -->
<!-- padding line 137 -->
<!-- padding line 138 -->
<!-- padding line 139 -->
<!-- padding line 140 -->
<!-- padding line 141 -->
<!-- padding line 142 -->
<!-- padding line 143 -->
<!-- padding line 144 -->
<!-- padding line 145 -->
<!-- padding line 146 -->
<!-- padding line 147 -->
<!-- padding line 148 -->
<!-- padding line 149 -->
<!-- padding line 150 -->
<!-- padding line 151 -->
<!-- padding line 152 -->
<!-- padding line 153 -->
<!-- padding line 154 -->
<!-- padding line 155 -->
<!-- padding line 156 -->
<!-- padding line 157 -->
<!-- padding line 158 -->
<!-- padding line 159 -->
<!-- padding line 160 -->
<!-- padding line 161 -->
<!-- padding line 162 -->
<!-- padding line 163 -->
<!-- padding line 164 -->
<!-- padding line 165 -->
<!-- padding line 166 -->
<!-- padding line 167 -->
<!-- padding line 168 -->
<!-- padding line 169 -->
<!-- padding line 170 -->
<!-- padding line 171 -->
<!-- padding line 172 -->
<!-- padding line 173 -->
<!-- padding line 174 -->
<!-- padding line 175 -->
<!-- padding line 176 -->
<!-- padding line 177 -->
<!-- padding line 178 -->
<!-- padding line 179 -->
<!-- padding line 180 -->
<!-- padding line 181 -->
<!-- padding line 182 -->
<!-- padding line 183 -->
<!-- padding line 184 -->
<!-- padding line 185 -->
<!-- padding line 186 -->
<!-- padding line 187 -->
<!-- padding line 188 -->
<!-- padding line 189 -->
<!-- padding line 190 -->
<!-- padding line 191 -->
<!-- padding line 192 -->
<!-- padding line 193 -->
<!-- padding line 194 -->
<!-- padding line 195 -->
<!-- padding line 196 -->
<!-- padding line 197 -->
<!-- padding line 198 -->
<!-- padding line 199 -->
<!-- padding line 200 -->
<!-- padding line 201 -->
<!-- padding line 202 -->
<!-- padding line 203 -->
<!-- padding line 204 -->
<!-- padding line 205 -->
<!-- padding line 206 -->
<!-- padding line 207 -->
<!-- padding line 208 -->
<!-- padding line 209 -->
<!-- padding line 210 -->
<!-- padding line 211 -->
<!-- padding line 212 -->
<!-- padding line 213 -->
<!-- padding line 214 -->
<!-- padding line 215 -->
<!-- padding line 216 -->
<!-- padding line 217 -->
<!-- padding line 218 -->
<!-- padding line 219 -->
<!-- padding line 220 -->
<!-- padding line 221 -->
<!-- padding line 222 -->
<!-- padding line 223 -->
<!-- padding line 224 -->
<!-- padding line 225 -->
<!-- padding line 226 -->
<!-- padding line 227 -->
<!-- padding line 228 -->
<!-- padding line 229 -->
<!-- padding line 230 -->
<!-- padding line 231 -->
<!-- padding line 232 -->
<!-- padding line 233 -->
<!-- padding line 234 -->
<!-- padding line 235 -->
<!-- padding line 236 -->
<!-- padding line 237 -->
<!-- padding line 238 -->
<!-- padding line 239 -->
<!-- padding line 240 -->
<!-- padding line 241 -->
<!-- padding line 242 -->
<!-- padding line 243 -->
<!-- padding line 244 -->
<!-- padding line 245 -->
<!-- padding line 246 -->
<!-- padding line 247 -->
<!-- padding line 248 -->
<!-- padding line 249 -->
<!-- padding line 250 -->
<!-- padding line 251 -->
<!-- padding line 252 -->
<!-- padding line 253 -->
<!-- padding line 254 -->
<!-- padding line 255 -->
<!-- padding line 256 -->
<!-- padding line 257 -->
<!-- padding line 258 -->
<!-- padding line 259 -->
<!-- padding line 260 -->
<!-- padding line 261 -->
<!-- padding line 262 -->
<!-- padding line 263 -->
<!-- padding line 264 -->
<!-- padding line 265 -->
<!-- padding line 266 -->
<!-- padding line 267 -->
<!-- padding line 268 -->
<!-- padding line 269 -->
<!-- padding line 270 -->
<!-- padding line 271 -->
<!-- padding line 272 -->
<!-- padding line 273 -->
<!-- padding line 274 -->
<!-- padding line 275 -->
<!-- padding line 276 -->
<!-- padding line 277 -->
<!-- padding line 278 -->
<!-- padding line 279 -->
<!-- padding line 280 -->
<!-- padding line 281 -->
<!-- padding line 282 -->
<!-- padding line 283 -->
<!-- padding line 284 -->
<!-- padding line 285 -->
<!-- padding line 286 -->
<!-- padding line 287 -->
<!-- padding line 288 -->
<!-- padding line 289 -->
<!-- padding line 290 -->
<!-- padding line 291 -->
<!-- padding line 292 -->
<!-- padding line 293 -->
<!-- padding line 294 -->
<!-- padding line 295 -->
<!-- padding line 296 -->
<!-- padding line 297 -->
<!-- padding line 298 -->
<!-- padding line 299 -->
<!-- padding line 300 -->
<!-- padding line 301 -->
<!-- padding line 302 -->
<!-- padding line 303 -->
<!-- padding line 304 -->
<!-- padding line 305 -->
<!-- padding line 306 -->
<!-- padding line 307 -->
<!-- padding line 308 -->
<!-- padding line 309 -->
<!-- padding line 310 -->
<!-- padding line 311 -->
<!-- padding line 312 -->
<!-- padding line 313 -->
<!-- padding line 314 -->
<!-- padding line 315 -->
<!-- padding line 316 -->
<!-- padding line 317 -->
<!-- padding line 318 -->
<!-- padding line 319 -->
<!-- padding line 320 -->
<!-- padding line 321 -->
<!-- padding line 322 -->
<!-- padding line 323 -->
<!-- padding line 324 -->
<!-- padding line 325 -->
<!-- padding line 326 -->
<!-- padding line 327 -->
<!-- padding line 328 -->
<!-- padding line 329 -->
<!-- padding line 330 -->
<!-- padding line 331 -->
<!-- padding line 332 -->
<!-- padding line 333 -->
<!-- padding line 334 -->
<!-- padding line 335 -->
<!-- padding line 336 -->
<!-- padding line 337 -->
<!-- padding line 338 -->
<!-- padding line 339 -->
<!-- padding line 340 -->
<!-- padding line 341 -->
<!-- padding line 342 -->
<!-- padding line 343 -->
<!-- padding line 344 -->
<!-- padding line 345 -->
<!-- padding line 346 -->
<!-- padding line 347 -->
<!-- padding line 348 -->
<!-- padding line 349 -->
<!-- padding line 350 -->
<!-- padding line 351 -->
<!-- padding line 352 -->
<!-- padding line 353 -->
<!-- padding line 354 -->
<!-- padding line 355 -->
<!-- padding line 356 -->
<!-- padding line 357 -->
<!-- padding line 358 -->
<!-- padding line 359 -->
<!-- padding line 360 -->
<!-- padding line 361 -->
<!-- padding line 362 -->
<!-- padding line 363 -->
<!-- padding line 364 -->
<!-- padding line 365 -->
<!-- padding line 366 -->
<!-- padding line 367 -->
<!-- padding line 368 -->
<!-- padding line 369 -->
<!-- padding line 370 -->
<!-- padding line 371 -->
<!-- padding line 372 -->
<!-- padding line 373 -->
<!-- padding line 374 -->
<!-- padding line 375 -->
<!-- padding line 376 -->
<!-- padding line 377 -->
<!-- padding line 378 -->
<!-- padding line 379 -->
<!-- padding line 380 -->
<!-- padding line 381 -->
<!-- padding line 382 -->
<!-- padding line 383 -->
<!-- padding line 384 -->
<!-- padding line 385 -->
<!-- padding line 386 -->
<!-- padding line 387 -->
<!-- padding line 388 -->
<!-- padding line 389 -->
<!-- padding line 390 -->
<!-- padding line 391 -->
<!-- padding line 392 -->
<!-- padding line 393 -->
<!-- padding line 394 -->
<!-- padding line 395 -->
<!-- padding line 396 -->
<!-- padding line 397 -->
<!-- padding line 398 -->
<!-- padding line 399 -->
<!-- padding line 400 -->
<!-- padding line 401 -->
<!-- padding line 402 -->
<!-- padding line 403 -->
<!-- padding line 404 -->
<!-- padding line 405 -->
<!-- padding line 406 -->
<!-- padding line 407 -->
<!-- padding line 408 -->
<!-- padding line 409 -->
<!-- padding line 410 -->
<!-- padding line 411 -->
<!-- padding line 412 -->
<!-- padding line 413 -->
<!-- padding line 414 -->
<!-- padding line 415 -->
<!-- padding line 416 -->
<!-- padding line 417 -->
<!-- padding line 418 -->
<!-- padding line 419 -->
<!-- padding line 420 -->
<!-- padding line 421 -->
<!-- padding line 422 -->
<!-- padding line 423 -->
<!-- padding line 424 -->
<!-- padding line 425 -->
<!-- padding line 426 -->
<!-- padding line 427 -->
<!-- padding line 428 -->
<!-- padding line 429 -->
<!-- padding line 430 -->
<!-- padding line 431 -->
<!-- padding line 432 -->
<!-- padding line 433 -->
<!-- padding line 434 -->
<!-- padding line 435 -->
<!-- padding line 436 -->
<!-- padding line 437 -->
<!-- padding line 438 -->
<!-- padding line 439 -->
<!-- padding line 440 -->
<!-- padding line 441 -->
<!-- padding line 442 -->
<!-- padding line 443 -->
<!-- padding line 444 -->
<!-- padding line 445 -->
<!-- padding line 446 -->
<!-- padding line 447 -->
<!-- padding line 448 -->
<!-- padding line 449 -->
<!-- padding line 450 -->
<!-- padding line 451 -->
<!-- padding line 452 -->
<!-- padding line 453 -->
<!-- padding line 454 -->
<!-- padding line 455 -->
<!-- padding line 456 -->
<!-- padding line 457 -->
<!-- padding line 458 -->
<!-- padding line 459 -->
<!-- padding line 460 -->
<!-- padding line 461 -->
<!-- padding line 462 -->
<!-- padding line 463 -->
<!-- padding line 464 -->
<!-- padding line 465 -->
<!-- padding line 466 -->
<!-- padding line 467 -->
<!-- padding line 468 -->
<!-- padding line 469 -->
<!-- padding line 470 -->
<!-- padding line 471 -->
<!-- padding line 472 -->
<!-- padding line 473 -->
<!-- padding line 474 -->
<!-- padding line 475 -->
<!-- padding line 476 -->
<!-- padding line 477 -->
<!-- padding line 478 -->
<!-- padding line 479 -->
<!-- padding line 480 -->
<!-- padding line 481 -->
<!-- padding line 482 -->
<!-- padding line 483 -->
<!-- padding line 484 -->
<!-- padding line 485 -->
<!-- padding line 486 -->
<!-- padding line 487 -->
<!-- padding line 488 -->
<!-- padding line 489 -->
<!-- padding line 490 -->
<!-- padding line 491 -->
<!-- padding line 492 -->
<!-- padding line 493 -->
<!-- padding line 494 -->
<!-- padding line 495 -->
<!-- padding line 496 -->
<!-- padding line 497 -->
<!-- padding line 498 -->
<!-- padding line 499 -->
<!-- padding line 500 -->
<!-- padding line 501 -->
<!-- padding line 502 -->
<!-- padding line 503 -->
<!-- padding line 504 -->
<!-- padding line 505 -->
<!-- padding line 506 -->
<!-- padding line 507 -->
<!-- padding line 508 -->
<!-- padding line 509 -->
<!-- padding line 510 -->
<!-- padding line 511 -->
<!-- padding line 512 -->
<!-- padding line 513 -->
<!-- padding line 514 -->
<!-- padding line 515 -->
<!-- padding line 516 -->
<!-- padding line 517 -->
<!-- padding line 518 -->
<!-- padding line 519 -->
<!-- padding line 520 -->
<!-- padding line 521 -->
<!-- padding line 522 -->
<!-- padding line 523 -->
<!-- padding line 524 -->
<!-- padding line 525 -->
<!-- padding line 526 -->
<!-- padding line 527 -->
<!-- padding line 528 -->
<!-- padding line 529 -->
<!-- padding line 530 -->
<!-- padding line 531 -->
<!-- padding line 532 -->
<!-- padding line 533 -->
<!-- padding line 534 -->
<!-- padding line 535 -->
<!-- padding line 536 -->
<!-- padding line 537 -->
<!-- padding line 538 -->
<!-- padding line 539 -->
<!-- padding line 540 -->
<!-- padding line 541 -->
<!-- padding line 542 -->
<!-- padding line 543 -->
<!-- padding line 544 -->
<!-- padding line 545 -->
<!-- padding line 546 -->
<!-- padding line 547 -->
<!-- padding line 548 -->
<!-- padding line 549 -->
<!-- padding line 550 -->
<!-- padding line 551 -->
<!-- padding line 552 -->
<!-- padding line 553 -->
<!-- padding line 554 -->
<!-- padding line 555 -->
<!-- padding line 556 -->
<!-- padding line 557 -->
<!-- padding line 558 -->
<!-- padding line 559 -->
<!-- padding line 560 -->
<!-- padding line 561 -->
<!-- padding line 562 -->
<!-- padding line 563 -->
<!-- padding line 564 -->
<!-- padding line 565 -->
<!-- padding line 566 -->
<!-- padding line 567 -->
<!-- padding line 568 -->
<!-- padding line 569 -->
<!-- padding line 570 -->
<!-- padding line 571 -->
<!-- padding line 572 -->
<!-- padding line 573 -->
<!-- padding line 574 -->
<!-- padding line 575 -->
<!-- padding line 576 -->
<!-- padding line 577 -->
<!-- padding line 578 -->
<!-- padding line 579 -->
<!-- padding line 580 -->
<!-- padding line 581 -->
<!-- padding line 582 -->
<!-- padding line 583 -->
<!-- padding line 584 -->
<!-- padding line 585 -->
<!-- padding line 586 -->
<!-- padding line 587 -->
<!-- padding line 588 -->
<!-- padding line 589 -->
<!-- padding line 590 -->
<!-- padding line 591 -->
<!-- padding line 592 -->
<!-- padding line 593 -->
<!-- padding line 594 -->
<!-- padding line 595 -->
<!-- padding line 596 -->
<!-- padding line 597 -->
<!-- padding line 598 -->
<!-- padding line 599 -->
<!-- padding line 600 -->
<!-- padding line 601 -->
<!-- padding line 602 -->
<!-- padding line 603 -->
<!-- padding line 604 -->
<!-- padding line 605 -->
<!-- padding line 606 -->
<!-- padding line 607 -->
<!-- padding line 608 -->
<!-- padding line 609 -->
<!-- padding line 610 -->
<!-- padding line 611 -->
<!-- padding line 612 -->
<!-- padding line 613 -->
<!-- padding line 614 -->
<!-- padding line 615 -->
<!-- padding line 616 -->
<!-- padding line 617 -->
<!-- padding line 618 -->
<!-- padding line 619 -->
<!-- padding line 620 -->
<!-- padding line 621 -->
<!-- padding line 622 -->
<!-- padding line 623 -->
<!-- padding line 624 -->
<!-- padding line 625 -->
<!-- padding line 626 -->
<!-- padding line 627 -->
<!-- padding line 628 -->
<!-- padding line 629 -->
<!-- padding line 630 -->
<!-- padding line 631 -->
<!-- padding line 632 -->
<!-- padding line 633 -->
<!-- padding line 634 -->
<!-- padding line 635 -->
<!-- padding line 636 -->
<!-- padding line 637 -->
<!-- padding line 638 -->
<!-- padding line 639 -->
<!-- padding line 640 -->
<!-- padding line 641 -->
<!-- padding line 642 -->
<!-- padding line 643 -->
<!-- padding line 644 -->
<!-- padding line 645 -->
<!-- padding line 646 -->
<!-- padding line 647 -->
<!-- padding line 648 -->
<!-- padding line 649 -->
<!-- padding line 650 -->
<!-- padding line 651 -->
<!-- padding line 652 -->
<!-- padding line 653 -->
<!-- padding line 654 -->
<!-- padding line 655 -->
<!-- padding line 656 -->
<!-- padding line 657 -->
<!-- padding line 658 -->
<!-- padding line 659 -->
<!-- padding line 660 -->
<!-- padding line 661 -->
<!-- padding line 662 -->
<!-- padding line 663 -->
<!-- padding line 664 -->
<!-- padding line 665 -->
<!-- padding line 666 -->
<!-- padding line 667 -->
<!-- padding line 668 -->
<!-- padding line 669 -->
<!-- padding line 670 -->
<!-- padding line 671 -->
<!-- padding line 672 -->
<!-- padding line 673 -->
<!-- padding line 674 -->
<!-- padding line 675 -->
<!-- padding line 676 -->
<!-- padding line 677 -->
<!-- padding line 678 -->
<!-- padding line 679 -->
<!-- padding line 680 -->
<!-- padding line 681 -->
<!-- padding line 682 -->
<!-- padding line 683 -->
<!-- padding line 684 -->
<!-- padding line 685 -->
<!-- padding line 686 -->
<!-- padding line 687 -->
<!-- padding line 688 -->
<!-- padding line 689 -->
<!-- padding line 690 -->
<!-- padding line 691 -->
<!-- padding line 692 -->
<!-- padding line 693 -->
<!-- padding line 694 -->
<!-- padding line 695 -->
<!-- padding line 696 -->
<!-- padding line 697 -->
<!-- padding line 698 -->
<!-- padding line 699 -->
<!-- padding line 700 -->
<!-- padding line 701 -->
<!-- padding line 702 -->
<!-- padding line 703 -->
<!-- padding line 704 -->
<!-- padding line 705 -->
<!-- padding line 706 -->
<!-- padding line 707 -->
<!-- padding line 708 -->
<!-- padding line 709 -->
<!-- padding line 710 -->
<!-- padding line 711 -->
<!-- padding line 712 -->
<!-- padding line 713 -->
<!-- padding line 714 -->
<!-- padding line 715 -->
<!-- padding line 716 -->
<!-- padding line 717 -->
<!-- padding line 718 -->
<!-- padding line 719 -->
<!-- padding line 720 -->
<!-- padding line 721 -->
<!-- padding line 722 -->
<!-- padding line 723 -->
