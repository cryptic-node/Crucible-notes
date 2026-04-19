# Crucible Roadmap v4 — Line-Item Implementation Plan

**Supersedes:** v1, v2, v3.
**Pairs with:** `CRUCIBLE_BUILD_PLAN.md` (architecture reference).

## What changed from v3

- **Memory model is now four-kinds:** shards (raw, dated), MEMORY.md and USER.md (nightly digests of shards), solutions (successful patterns, queued for review), skills (promoted patterns, actively loaded). Replaces v3's simpler Hermes-style model.
- **`solution_reviewer` module added** (Phase 4b). Overnight pipeline that processes queued solutions through static checks → error/exception probes → alternative suggestions → verdict. Promotes trusted solutions into skills.
- **`skill_workshop` module added** (Phase 8b). Four internal roles (assessor, coder, tester, reviewer) running on potentially different models, sharing a work-item file with full history. Handles skill evolution and Path B self-improvement.
- **Path A/B self-improvement split is now explicit in policy.** Path A (Crucible core or module source changes) always produces a gitea PR for human+Claude review. Path B (skill and memory iteration inside a module's own namespace) can run autonomously after a trust-building dry-run period. Policy engine enforces which path applies.
- **Permanent architectural invariant added:** Crucible never merges its own proposals, never restarts itself with new code, never self-deploys. Written into ARCHITECTURE.md as a rule, not a default.

## Phase summary

| Phase | Topic | Output |
|---|---|---|
| 1 | Core foundation | Manifest, broker, audit, SDK, three isolation launchers |
| 2 | LLM layer | Provider interface, Ollama + cloud adapters, router, budgets |
| 3 | Git-backed KB | Store, index, schema, git backend, gitea remote |
| 4a | Closed-loop memory | Four-kinds model, shards → digests → queues |
| 4b | Solution reviewer | Overnight pipeline promoting solutions to skills |
| 5 | Wiki researcher | Flagship module — nightly research + KB maintenance |
| 6 | Hardening | Approval UX, observability, recovery, backup, tests, docs |
| 7 | Web UI | FastAPI + React over Tailscale+Caddy |
| 8a | self_dev (Path A) | Crucible proposes core/module changes as gitea PRs |
| 8b | skill_workshop (Path B) | Four-role skill evolution inside modules |
| 9 | site_builder | Private website builder on tailnet |
| 10+ | Remaining modules | Per-module implementations, priority-ordered |

---

# Phase 1 — Core Foundation (line items 1.01 – 1.12)

Unchanged from v3.

## 1.01 Architecture lock-in
**Deliverable:** `ARCHITECTURE.md` and `CONTEXT.md`. Includes the permanent invariant: "Crucible will never merge its own proposals into main, never restart itself with new code, and never apply changes to its own running environment. All such actions require a human to run the upgrade script manually."
**Sessions:** 1.
**Acceptance:** re-reading cold the next morning, you could implement against the docs.

## 1.02 Manifest schema and validation
**Deliverable:** `crucible/core/manifest.py` Pydantic models. Fields: name, version, scopes (with resource, risk, budget, allowlist, rate_limit), schedule, guardrails, isolation, memory (bool + config), self_improvement_policy (path_a | path_b | none, with constraints).
**Contract touched:** **manifest schema (frozen after this).**
**Sessions:** 1.
**Acceptance:** valid manifests load; `shell.exec: SAFE` is rejected; `isolation: inprocess` on production-tagged module warns; a module declaring `self_improvement_policy: path_b` for a scope it doesn't own is rejected.

## 1.03 Policy engine
**Deliverable:** `crucible/core/policy.py`. Pure function (manifest + request) → Decision. Tests every branch including the new self-improvement policy checks.
**Sessions:** 1.
**Acceptance:** all four risk tiers tested; budget, rate, schedule, scope-not-declared all tested; Path A vs Path B decisions tested.

## 1.04 Broker skeleton
**Deliverable:** `crucible/core/broker.py` with stub execution and in-memory approval queue.
**Contract touched:** **broker interface (frozen after this).**
**Sessions:** 1.
**Acceptance:** broker returns ALLOW/DENY/PENDING; queue inspectable.

## 1.05 Audit logger with hash chain
**Deliverable:** `crucible/core/audit.py`. Append-only JSONL, hash-chained, with `verify()`.
**Contract touched:** **audit log format (frozen after this).**
**Sessions:** 1.
**Acceptance:** 1000 records verify clean; corrupting one byte is detected at exact index.

## 1.06 Event bus
**Deliverable:** `crucible/core/events.py`. In-process pub/sub.
**Sessions:** 1.
**Acceptance:** N subscribers all receive; slow subscribers don't block fast ones.

## 1.07 Secrets via keyring
**Deliverable:** `crucible/core/secrets.py`. Keyring-first, age-encrypted-file fallback.
**Sessions:** 1.
**Acceptance:** round-trips a fake key; missing secret fails config validation loudly.

## 1.08 Module SDK base class
**Deliverable:** `crucible/modules/sdk.py`. `CrucibleModule` base class, `ctx` object, lifecycle hooks. Stubs for `self.memory` (wired in Phase 4a).
**Contract touched:** **module SDK (frozen after this).**
**Sessions:** 1.
**Acceptance:** 20-line subclass with lifecycle logging works.

## 1.09 Module launcher — inprocess
**Sessions:** 1.
**Acceptance:** hello-world runs, SAFE requests allowed + audited.

## 1.10 Module launcher — subprocess with bwrap + seccomp
**Sessions:** 2.
**Acceptance:** hello-world unchanged; `open("/etc/passwd")` fails; `socket()` fails; module crash handled cleanly.

## 1.11 Module launcher — docker
**Sessions:** 2.
**Acceptance:** isolation: docker module runs; `curl google.com` fails from inside; deadline kills container cleanly.

## 1.12 Module registry and loader
**Sessions:** 1.
**Acceptance:** `crucible modules list` shows three modules across three tiers.

**Phase 1 checkpoint:** three hello-world modules at three isolation levels, all audited, hash chain verifies.

---

# Phase 2 — LLM Layer (line items 2.01 – 2.05)

Unchanged from v3.

## 2.01 LLMClient interface
**Sessions:** 1.

## 2.02 Ollama adapter
**Sessions:** 1.

## 2.03 Cloud adapter stubs (OpenAI, Anthropic, xAI, OpenRouter)
**Sessions:** 2.

## 2.04 Provider router
**Sessions:** 1.
**Note:** router now also handles role-specific model selection — a module can request "this call is the reviewer role, prefer a different model family than the coder role." This enables skill_workshop's independent-judgment pattern.

## 2.05 Budget tracker
**Sessions:** 1.

**Phase 2 checkpoint:** modules make LLM calls through the broker, routed by role and capability, budgeted, audited.

---

# Phase 3 — Git-backed Knowledge Base (line items 3.01 – 3.06)

Unchanged from v3.

## 3.01 KB storage layer
**Sessions:** 1.

## 3.02 KB search index (SQLite FTS5)
**Sessions:** 1.

## 3.03 Research entry schema
**Contract touched:** **research entry format (frozen after this).**
**Sessions:** 1.

## 3.04 Git backend (signed commits, per-write and batched cadences)
**Sessions:** 2.

## 3.05 Gitea remote with offline fallback
**Sessions:** 1.

## 3.06 KB proxy for modules
**Sessions:** 1.

**Phase 3 checkpoint:** every KB write is a signed commit pushed to Umbrel gitea over Tailscale.

---

# Phase 4a — Four-Kinds Memory Model (line items 4a.01 – 4a.07)

The core closed-loop learning primitive. Every module gets structured memory from day one of Phase 5 onward.

## 4a.01 Memory directory layout and schema
**Deliverable:** the on-disk convention for every module's `memory/` directory:

```
modules/<name>/memory/
├── MEMORY.md                    ← long-form domain summary
├── USER.md                      ← operator preferences
├── shards/
│   └── YYYY-MM-DD-HHMM-<id>.md  ← atomic observations
├── solutions/
│   ├── YYYY-MM-DD-<slug>.md     ← successful patterns
│   └── queue.yaml               ← awaiting review
└── skills/
    └── <slug>.md                ← promoted patterns, loaded at run start
```

Plus the front-matter schemas for each kind. Shard front-matter: `{id, created_at, run_id, topic_tags, retention_hint}`. Solution front-matter: `{id, created_at, run_id, problem_context, approach, outcome, confidence, queue_status}`. Skill front-matter: `{id, promoted_at, source_solution_id, version, tested_against}`. MEMORY.md and USER.md are frontmatter-less but have a top-block with `{last_digested_at, source_shard_count, token_count}`.

**Contract touched:** **memory schema (frozen after this).**
**Sessions:** 1.
**Acceptance:** all four kinds validate against their schemas; examples committed; the retention_hint values are enumerated (ephemeral, normal, persistent).

## 4a.02 Memory proxy — the `self.memory` API
**Deliverable:** `crucible/core/memory.py`. Methods:
- `shard(content, topic_tags=[], retention_hint='normal')` → append a shard.
- `note(content)` → convenience wrapper for MEMORY.md-bound observations; actually creates a shard tagged `digest_candidate`.
- `recall(query, kind=None, limit=10)` → FTS search over this module's memory.
- `recall_cross(query, namespaces=[...], kind=None)` → cross-namespace search, gated by manifest.
- `solution_propose(problem_context, approach, outcome, confidence)` → writes a solution file and enqueues it.
- `skill_get(name)` → returns a loaded skill by id.
- `skills_active()` → list of skills currently loaded for this run.

All calls broker-mediated and audited. Writes go to the module's own namespace; cross-namespace reads gated by manifest scopes.

**Sessions:** 2.
**Acceptance:** a test module calls every proxy method; every call audited; writes outside namespace rejected; cross-namespace reads gated.

## 4a.03 Memory in the git backend
**Deliverable:** memory directories are tracked in per-module gitea repos (`crucible-memory-<module>`), separate from the KB repo. Commit cadences: shards batch-committed per run (one commit per run for all shards produced), solutions per-write (each is meaningful), skills per-write, MEMORY.md and USER.md per-digest-run.
**Sessions:** 1.
**Acceptance:** writes appear as signed commits; reverting a commit and pulling restores prior memory state.

## 4a.04 FTS5 search over memory
**Deliverable:** extend the existing SQLite FTS5 index from 3.02 to cover memory files, namespaced by module and kind. Sub-100ms recall on 10k shards.
**Sessions:** 1.
**Acceptance:** recall returns ranked results; kind filters work (`kind='skill'`, `kind='solution'`); cross-namespace queries respect manifest gating.

## 4a.05 Nightly digest pass — shards into MEMORY.md and USER.md
**Deliverable:** a scheduled task (not a module — a core responsibility) that, per module, reads the day's shards, passes them to a small LLM with the current MEMORY.md and USER.md, and produces updated versions. Constraints: MEMORY.md stays under a configurable token budget (default 8k), USER.md under 2k. Preference-tagged shards route to USER.md; everything else to MEMORY.md. Shards are NOT deleted after digest — they remain searchable and are the audit trail for what went into the summary.

**Sessions:** 2.
**Acceptance:** seeding 50 shards including several preference observations produces a MEMORY.md under budget and a USER.md that correctly captures the preferences; digest is idempotent (re-running on same shards produces same output); the LLM call is budgeted against the module's budget.

## 4a.06 Shard pruning policy
**Deliverable:** shards respect their `retention_hint`: `ephemeral` (pruned after digest), `normal` (pruned after 90 days), `persistent` (never pruned). Pruning is a separate scheduled task. Pruned shards are deleted via git commit, so the history is preserved in gitea even though the working copy shrinks.
**Sessions:** 1.
**Acceptance:** after simulated 90 days, `normal` shards are pruned, `persistent` remain, git log shows the deletion commits.

## 4a.07 Memory loading at run start
**Deliverable:** when a module starts a run, its `ctx` is pre-populated with MEMORY.md, USER.md, and a list of active skills. Shards and solutions are not loaded (too much context) but are searchable via `self.memory.recall`. A module can opt into loading specific skill files based on the task at hand.

**Sessions:** 1.
**Acceptance:** a module's run receives MEMORY.md and USER.md automatically; `ctx.skills` lists available skills with their descriptions; the module can load skills by name mid-run without re-reading disk.

**Phase 4a checkpoint:** every module that ships from Phase 5 onward has structured memory. The digest pass condenses noisy shards into stable summaries. Preferences accumulate into USER.md. The audit trail is intact in gitea.

---

# Phase 4b — Solution Reviewer Module (line items 4b.01 – 4b.05)

A small overnight module that turns the solutions queue into a curated skill library.

## 4b.01 Module scaffold and manifest
**Deliverable:** `modules/solution_reviewer/manifest.yaml`. Scopes: `llm.call`, `fs.read` on every module's memory directory (this is the one cross-namespace memory read in the system, and it's explicit here), `fs.write` scoped to each module's `solutions/queue.yaml` and `skills/` directory — gated with a per-target approval policy so the reviewer can promote solutions to skills but cannot silently edit unrelated files. Isolation: subprocess. Schedule: 02:00 nightly, after main modules have finished.
**Sessions:** 1.
**Acceptance:** loads, manifest validates, dry run reads the test module's solutions queue without modifying anything.

## 4b.02 Static checks pass
**Deliverable:** for each queued solution, run structural checks before any LLM call: does it reference files that exist? Does it have complete front-matter? Is it a near-duplicate of an existing skill (fuzzy match on approach)? Cheap rejects and auto-merges happen here. Each static-check outcome is logged to the solution's `review_history` field.
**Sessions:** 1.
**Acceptance:** a malformed solution is rejected; a near-duplicate is flagged for merge; a clean solution proceeds; all outcomes audit-logged.

## 4b.03 Error and exception probe
**Deliverable:** for solutions passing static checks, an LLM call constructs 3–5 edge cases and failure modes, then predicts the solution's behavior in each. Output is added to `review_history` with pass/fail per case.
**Sessions:** 2.
**Acceptance:** probe on a test solution with known fragility correctly flags at least the obvious edge case; probe budget is respected (fails gracefully if exhausted).

## 4b.04 Alternative suggestion pass
**Deliverable:** an LLM call asks "is there a better way to do this?" If yes, draft the alternative as a new solution entry in the same queue with a cross-reference to the original. Original stays in queue; new alternative also requires review. Prevents infinite alternative-chains with a `generation` counter capped at 2.
**Sessions:** 1.
**Acceptance:** a deliberately-suboptimal test solution produces a sensible alternative; the alternative appears in the queue; the generation cap is enforced.

## 4b.05 Verdict and promotion
**Deliverable:** combine outputs from static checks, probe, alternatives into a final verdict: `promote`, `needs_revision`, `reject`, `alternative_pending`. Promoted solutions are copied to the module's `skills/` directory with a skill-format front-matter block linking back to the source solution. `queue.yaml` entry is marked resolved. All decisions logged.
**Sessions:** 1.
**Acceptance:** over a mixed batch of 10 test solutions, verdicts are sensible (not all-promote, not all-reject); promoted skills appear in the right module's skills directory with correct provenance links; the morning `!review` CLI command surfaces the `needs_revision` items for you to look at.

**Phase 4b checkpoint:** solutions accumulate during the day, get reviewed overnight, and promoted solutions become skills that modules load the next time they run. You wake up to a short list of `needs_revision` items — not a pile of raw output.

---

# Phase 5 — Wiki/KB Maintainer-Researcher Module (line items 5.01 – 5.08)

The flagship first module. Subprocess isolation. Benefits from Phase 4a/4b memory from run one.

## 5.01 Module scaffold and manifest
**Deliverable:** manifest with scopes for llm.call, web.search, web.fetch (allowlist), kb.write + kb.read in own namespace, fs.write to workspace. `memory: true` with `retention: normal`. `self_improvement_policy: path_b` for its own skills directory (can iterate its research-technique skills autonomously). Schedule for overnight research.
**Sessions:** 1.

## 5.02 Web capability
**Deliverable:** `crucible/tools/web.py` with SearXNG primary, DuckDuckGo fallback. First-time domain requests RISKY, trust-window-then-SENSITIVE.
**Sessions:** 2.

## 5.03 Research mode — plan phase
**Sessions:** 1.

## 5.04 Research mode — gather phase
**Sessions:** 2.

## 5.05 Research mode — draft → critique → revise loop
**Sessions:** 2.

## 5.06 Research mode — finalize and publish
**Sessions:** 1.

## 5.07 Maintain mode
**Sessions:** 2.

## 5.08 Topic queue and scheduler
**Sessions:** 1.

**Phase 5 checkpoint:** real nightly research bot, memory-enabled, gitea-backed, and producing solution candidates that Phase 4b will promote into skills over time.

---

# Phase 6 — Hardening and UX (line items 6.01 – 6.07)

## 6.01 CLI approval flow polish + memory/review commands
**Deliverable:** existing approval commands plus `!memory <module>` (browse a module's memory), `!review` (show last night's solution_reviewer verdicts), `!skills <module>` (list active skills).
**Sessions:** 1.

## 6.02 Observability and health
**Sessions:** 1.

## 6.03 Recovery and resume
**Sessions:** 1.

## 6.04 Log rotation with chain preservation
**Sessions:** 1.

## 6.05 End-to-end test harness
**Deliverable:** e2e scenarios now include memory persistence (recall across simulated days), solution promotion (queue → skill), digest correctness (shards → MEMORY.md).
**Sessions:** 2.

## 6.06 Backup and restore
**Deliverable:** backup includes all per-module memory repos, not just the KB.
**Sessions:** 1.

## 6.07 Documentation pass
**Sessions:** 1.

**Phase 6 checkpoint:** v0.1 ready for overnight trust. Weekend release.

---

# Phase 7 — Web UI over Tailscale + Caddy (line items 7.01 – 7.07)

## 7.01 FastAPI backend skeleton
**Deliverable:** REST endpoints for modules, approvals, audit, KB, memories (per module, per kind), solution queue, budgets. WebSocket for live events.
**Sessions:** 2.

## 7.02 Caddy for Tailscale
**Sessions:** 1.

## 7.03 React frontend — chat + approvals
**Sessions:** 2.

## 7.04 Frontend — modules, audit, KB, memories, solutions
**Deliverable:** module list, audit viewer, KB browser, per-module memory inspector (shards tab, solutions tab with queue status, skills tab), solution_reviewer overnight results.
**Sessions:** 2.

## 7.05 Frontend — settings, providers, schedules
**Sessions:** 1.

## 7.06 Auth — Tailscale identity
**Sessions:** 1.

## 7.07 Auth — Nostr login (optional, for public paths)
**Sessions:** 2.

**Phase 7 checkpoint:** web UI over Tailscale. You can browse every module's accumulated memory, see what last night's reviewer promoted, and intervene where needed.

---

# Phase 8a — self_dev Module (Path A) (line items 8a.01 – 8a.06)

Crucible proposes changes to Crucible's own source code or to any module's source code. Always Path A: produces reviewable PRs in gitea, never auto-merges, never auto-restarts. Docker isolation.

## 8a.01 Module scaffold, manifest, and threat model
**Deliverable:** manifest (docker isolation), `THREAT_MODEL.md`. Scopes: `http.gitea_api` scoped to Crucible source and module source repos, `llm.call` with premium budget, `kb.write` in `self_dev` namespace, `memory: true`, `fs.read` on a read-only bind-mount of Crucible source. **Explicitly no** `fs.write` on source, no `shell.exec` on host, no `self_improvement_policy: path_b` — this module cannot modify itself via Path B.
**Sessions:** 2.
**Acceptance:** threat model reviewed; manifest rejects anything unenumerated; module can read source but not write to it.

## 8a.02 Source analysis — read, index, understand
**Deliverable:** startup pass that indexes the Crucible codebase (file tree, module manifests, tests, TODOs, dependency graph between modules). Index stored as a KB entry, browseable in the UI, updated on each run.
**Sessions:** 2.

## 8a.03 Proposal engine — generate change proposals
**Deliverable:** given a prompt, produce a git branch in the target repo on gitea with code changes, matching test changes, and a `PROPOSAL.md` at branch root (what, why, risk assessment, how it was tested). The "copy to Claude" section is included by default in PROPOSAL.md: a pre-formatted brief with goal, current state, proposed change, and context, ready for you to paste into claude.ai.
**Sessions:** 3.
**Acceptance:** simple prompt produces a branch with valid change, passing test, and a PROPOSAL.md including the Claude-ready section.

## 8a.04 Sandboxed test runner
**Deliverable:** ephemeral test container, checks out proposed branch, runs full suite, captures results into PROPOSAL.md. Running Crucible is never touched. Failed tests trigger up-to-N iterations before giving up.
**Sessions:** 2.

## 8a.05 Pull request creation
**Deliverable:** open PR in gitea from branch against `main`. PR tags include complexity estimate, risk tier, modules/contracts touched. Contract-touching PRs auto-flagged RISKY; PR template makes review fast. PR body includes the Claude-ready brief.
**Sessions:** 1.

## 8a.06 Upgrade runbook generation
**Deliverable:** after you merge a PR, the module produces `UPGRADE.md` describing safe deployment steps, expected downtime, rollback procedure. This is paired with a pre-filled approval request in Crucible's own queue: when you hit "approved to deploy" in the web UI, the module walks you through the commands to run (it generates and displays them; you run them), with real-time validation at each step. No auto-application.
**Sessions:** 2.

**Phase 8a checkpoint:** you can ask Crucible to improve Crucible or any module. It proposes, tests in a sandbox, opens a PR, provides a Claude-ready brief for external review, and holds your hand through deployment without ever applying changes itself.

---

# Phase 8b — skill_workshop Module (Path B) (line items 8b.01 – 8b.07)

A single module that runs four internal roles (assessor, coder, tester, reviewer) to evolve skills and memory inside any target module's own namespace. Path B scope only: never touches source code, never touches contracts. Docker isolation.

## 8b.01 Module scaffold, manifest, and role architecture
**Deliverable:** manifest (docker isolation). Scopes: `llm.call` (can request different providers per role), `fs.read` on target module's memory and skills, `fs.write` scoped per-target to `skills/` and `solutions/` directories of modules that have declared `self_improvement_policy: path_b`. Internal architecture: four roles (assessor, coder, tester, reviewer) each with their own system prompt and preferred-model config, communicating via a shared work-item YAML file per job. Roles run sequentially.
**Sessions:** 2.
**Acceptance:** a work item walks through all four roles; each role's output lands in the work-item's `history` array with model, timestamp, and verdict; work-item can be inspected mid-flight.

## 8b.02 Work-item schema
**Deliverable:** the work-item YAML format:
```yaml
id: wi-2026-04-19-a3f2
target_module: weather_tracker
target_kind: skill       # or solution, memory_digest
target_id: retry-on-429
status: assessed         # queued → assessed → coded → tested → reviewing → approved|rejected|needs_human
history:
  - role: assessor
    timestamp: ...
    model: ...
    verdict: improve
    notes: ...
  # ...
dry_run: true            # first week of operation, no writes
```
**Contract touched:** **work-item schema (frozen after this line item).**
**Sessions:** 1.

## 8b.03 Assessor role
**Deliverable:** given a target skill or memory artifact, the assessor role reads it plus the target module's recent run history and produces a verdict: `improve`, `keep`, `deprecate`, or `needs_human`. "Improve" proposals include a concrete description of what to change and why. Model preference: a cheap local model (Ollama) is sufficient for most cases. Writes to work-item history.
**Sessions:** 1.
**Acceptance:** given a skill that has produced failures recently, assessor identifies improvement opportunity; given a clean skill, assessor marks `keep`.

## 8b.04 Coder role
**Deliverable:** given an assessor's "improve" verdict, the coder role drafts the new version of the skill or memory file. Model preference: premium (Anthropic, OpenAI) for quality. Writes draft to work-item and to a scratch path; does not yet replace the live file.
**Sessions:** 2.

## 8b.05 Tester role
**Deliverable:** given a coder's draft, the tester role synthesizes test cases and evaluates the draft against them. For skill files, tests are "would applying this skill to these past runs produce better outcomes?" For memory digests, tests are "does this digest still answer the queries that the old digest answered correctly?" Model preference: a different model family than the coder (for independent judgment).
**Sessions:** 2.
**Acceptance:** tester's evaluations are audit-logged with full reasoning; a draft with regressions is flagged.

## 8b.06 Reviewer role and promotion
**Deliverable:** given a tester's results, the reviewer role makes the final call: `approve`, `approve_with_notes`, `reject`, `needs_human`. On approve, the live skill or memory file is replaced (via normal memory proxy writes, which are git-committed and audit-logged). On `needs_human`, the work-item shows up in the morning `!review` list. Reviewer uses yet another model configuration to maximize independence.
**Sessions:** 2.

## 8b.07 Dry-run mode and trust calibration
**Deliverable:** the module ships with dry-run mode ON. In dry-run, every role runs and every decision is recorded, but no files are actually modified. Work-items accumulate for your inspection. A CLI command `!workshop review` shows the last N work-items with their verdicts. After you've reviewed enough to trust the output, a config flag turns dry-run off.
**Sessions:** 1.
**Acceptance:** dry-run mode produces full audit trails with zero side effects; flipping to live mode begins producing actual memory updates; re-flipping to dry-run stops them immediately.

**Phase 8b checkpoint:** skills and memory digests inside Path B-enabled modules evolve overnight via the four-role pipeline. You start in dry-run, you calibrate for a week, you trust it. Modules like weather_tracker continuously refine their retry heuristics and summary phrasing without you ever touching their code.

---

# Phase 9 — site_builder Module (line items 9.01 – 9.06)

Unchanged from v3. Setup wizard, scaffolds sites in gitea, serves on tailnet by default, Tailscale Serve as opt-in public path.

## 9.01 Module scaffold, manifest, setup wizard
**Sessions:** 2.

## 9.02 Site scaffolder — generate the initial codebase
**Sessions:** 2.

## 9.03 Caddy fragment manager
**Sessions:** 1.

## 9.04 Content iteration mode
**Sessions:** 2.

## 9.05 Tailscale Serve integration (opt-in public path)
**Sessions:** 1.

## 9.06 Site analytics via KB (optional, minimal, no PII)
**Sessions:** 1.

**Phase 9 checkpoint:** private website builder on your tailnet, one weekend.

---

# Phase 10+ — Remaining Modules

Same recipe: manifest (1 session), core logic + broker (1–2 sessions), tests + polish (1 session).

| Module | Isolation | Self-improvement | Sessions | Prereqs |
|---|---|---|---|---|
| system_maintenance | subprocess | path_b (runbooks only) | 3 | Phase 6 |
| coding_assistant (python) | subprocess | path_b (coding heuristics) | 5 | Phase 6 |
| nostr_watch | subprocess | path_b | 4 | Phase 6 |
| bitcoin_watch | docker | none | 4 | Phase 6 |
| weather_tracker | subprocess | path_b | 2 | Phase 5 |
| price_tracker | subprocess | path_b | 3 | Phase 5 |
| pantry_manager | subprocess | path_b | 4 | Phase 5 |
| podcast_assistant | subprocess | path_b | 5 | Phase 5 |
| health_tracker | docker | none (sensitive data) | 6 | Phase 6 + encryption design |
| business_manager | docker | none (money + legal) | 5 | Phase 6 |
| stock_researcher | subprocess → docker (when trading) | path_b (research) / none (trading) | 5 | Phase 6 |
| nostr_account_manager | docker | none | 6 | nostr_watch done |
| lightning_node_manager | docker | none | 8 | bitcoin_watch + threat model |
| 3d_printer_manager | subprocess | path_b | 4 | Phase 6 |
| nas_media_manager | subprocess | path_b | 4 | Phase 6 |
| gamedev_bot | subprocess | path_b | ongoing | coding_assistant |
| gitea_manager | subprocess | path_b | 5 | Phase 6 |

Note: `self_improvement_policy: none` is the default for modules touching money, legal filings, health data, keys, or external accounts. Path B is for cosmetic, heuristic, and technique-level iteration only.

---

# Session-opener template

> Working on Crucible, line item **X.YY**. Previous line item delivered **[brief]**. Today's goal: **[deliverable from this doc]**. Contracts pinned: manifest schema, broker interface, audit log format, module SDK, memory schema, work-item schema (see `CONTEXT.md`). Please produce the deliverable for line item X.YY with tests in the same response, targeting the acceptance test: **[acceptance]**.

---

# Weekend-release cadence

- End of weekend **Phase 1** complete: v0.1.0-alpha, three-tier isolation working.
- End of weekend **Phase 3** complete: v0.1.0-beta, gitea-backed KB.
- End of weekend **Phase 4a+4b** complete: v0.1.0-rc1, four-kinds memory and solution reviewer working.
- End of weekend **Phase 5** complete: v0.1.0, wiki_researcher ships.
- End of weekend **Phase 6** complete: v0.2.0, hardened.
- End of weekend **Phase 7** complete: v0.3.0, web UI.
- End of weekend **Phase 8a** complete: v0.4.0, self_dev (Path A).
- End of weekend **Phase 8b** complete: v0.5.0, skill_workshop (Path B, dry-run at first).
- End of weekend **Phase 9** complete: v0.6.0, site_builder.
