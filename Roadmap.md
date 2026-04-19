Crucible Roadmap v4 — Line-Item Implementation Plan
Supersedes: v1, v2, v3.
Pairs with: CRUCIBLE_BUILD_PLAN.md (architecture reference).
What changed from v3
	•	Memory model is now four-kinds: shards (raw, dated), MEMORY.md and USER.md (nightly digests of shards), solutions (successful patterns, queued for review), skills (promoted patterns, actively loaded). Replaces v3’s simpler Hermes-style model.
	•	solution_reviewer module added (Phase 4b). Overnight pipeline that processes queued solutions through static checks → error/exception probes → alternative suggestions → verdict. Promotes trusted solutions into skills.
	•	skill_workshop module added (Phase 8b). Four internal roles (assessor, coder, tester, reviewer) running on potentially different models, sharing a work-item file with full history. Handles skill evolution and Path B self-improvement.
	•	Path A/B self-improvement split is now explicit in policy. Path A (Crucible core or module source changes) always produces a gitea PR for human+Claude review. Path B (skill and memory iteration inside a module’s own namespace) can run autonomously after a trust-building dry-run period. Policy engine enforces which path applies.
	•	Permanent architectural invariant added: Crucible never merges its own proposals, never restarts itself with new code, never self-deploys. Written into ARCHITECTURE.md as a rule, not a default.
Phase summary
