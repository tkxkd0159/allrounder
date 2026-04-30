---
name: code-review
description: Use when reviewing branch or PR changes against a base branch for correctness, security, concurrency, architecture, maintainability, testing, and performance risks.
---

# Code Review

Review the current branch or PR against a base branch with adversarial, evidence-based scrutiny. Prefer subagent-driven review workers when the current AI CLI supports them; otherwise run the same lens passes inline.

## Inputs

- Default base branch: `origin/main`
- Read options from the user's message; standard skills do not receive a special `$ARGUMENTS` variable.
- If the user writes `--base BRANCH`, compare against that branch.
- If the user writes `--comment` or asks to post comments, first produce the review, then use the `add-pr-comments` skill if available.
- Example request: `Use code-review --base origin/develop --comment`.

## Worker Strategy

- If the environment supports spawning workers from plain text prompts, use one worker per activated lens and run them in parallel.
- In Codex CLI, this maps to `spawn_agent` with a generic `worker` agent when multi-agent support is enabled. If `spawn_agent` is unavailable, continue inline. In Claude Code, this maps to the general `Task` tool.
- In any other CLI, use its equivalent generic worker feature if present.

## Rubrics

**Severity**

- `critical` - exploitable security, auth bypass, data loss, corruption, severe outage
- `high` - likely production failure or serious regression under realistic conditions
- `medium` - real bug under a plausible edge case
- `low` - non-trivial, worth fixing, not a blocker

**Confidence**

- `high` - directly supported by the cited code
- `medium` - strong evidence, one assumption remains
- `low` - plausible but speculative; prefer residual risk over a low-confidence finding

## Procedure

1. **Pin the comparison.**
   - Resolve `BASE_BRANCH` from the user request or use `origin/main`.
   - Run `HEAD_SHA=$(git rev-parse HEAD)`.
   - Run `MERGE_BASE=$(git merge-base "$BASE_BRANCH" "$HEAD_SHA")`.
   - Diff against `"$MERGE_BASE".."$HEAD_SHA"` for the entire review.

2. **Inventory the diff.**
   - Inspect changed files, shortstat, added/deleted files, and the diff body.
   - Exclude generated files, vendored code, lockfiles, snapshots, and docs-only changes unless they affect runtime, build, or security.

3. **Activate lenses.**
   - `correctness`: always.
   - `architecture`: 3+ files changed, new top-level module/directory, import-graph changes, or public-export changes.
   - `security`: auth/session/token/crypto/secret/endpoint/api/middleware paths, validation, deserialization, subprocess/exec, SQL construction, network I/O, or PII.
   - `maintainability`: about 100+ added lines, new abstractions, hidden coupling, duplication, or rename-heavy diffs.
   - `testing`: test changes, production changes without adjacent tests, flaky assertions, brittle mocks, or weak coverage of changed behavior.
   - `performance`: DB query changes, loops over user data, hot-path handlers, network calls, caching, memoization, or avoidable round trips.
   - `concurrency`: shared mutable state, locks, async/await, threads, goroutines, channels, queues, retries, cancellation, or idempotency.
   - Heuristics are not hard limits. Run any lens plainly needed by the diff.
   - Record activated and skipped lenses with one-line reasons.

4. **Review each activated lens as a separate pass or worker.**
   Use this three-phase protocol for each lens:
   - **Evidence:** record 5-20 observations with `file:line-line`, current vs. prior behavior, touched invariants/callers, and reachable inputs. Mark unknowns as `unknown: <what to check>`. Do not assign severity yet.
   - **Candidates:** propose findings only when they cite phase-1 evidence. Include Title, Severity, Confidence, Evidence, Failure mode, and Recommended fix.
   - **Self-critique:** steelman the PR author for each candidate, then mark `KEEP`, `DOWNGRADE`, or `DROP`. Keep at most 5 findings per lens.
   - If using workers, give each worker only one lens scope plus the common rubrics, pinned SHAs, and reviewed diff range. Require a `FINAL` block with `Verdict`, `Checked`, `Findings`, and `Residual risks`.

5. **Validate findings directly.**
   - Re-open every cited range in the current code.
   - Confirm the cited line exists in the reviewed diff and supports the claimed failure mode.
   - Drop findings that cannot be verified from code. Move plausible but unverified concerns to residual risks.

6. **Aggregate.**
   - Merge duplicates across lenses when line ranges overlap, fall within about 5 lines, or describe the same failure mode.
   - Keep the higher severity/confidence and mention corroborating lenses.
   - Do one cross-lens gap analysis pass for combined risks that no single lens exposes.

7. **Report.**
   Output in this order:
   - **Executive summary:** base branch, merge-base SHA, changed-file count, activated lenses, skipped lenses, blockers, and residual risks.
   - One section per activated lens with surviving findings: Title, `SEV/CONF`, `file:line`, Failure mode, Fix.
   - **Cross-cutting risks:** only issues from the gap analysis.
   - **Approval:** exactly one of `ready to approve`, `ready after fixes`, or `not ready`.

## Lens Scopes

| Lens            | Scope                                                                                                                                                                                                                              |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| correctness     | Logic errors, null handling, off-by-one errors, control flow, contract mismatches, stale callers, silent data corruption, partial failures, rollback gaps, migration/schema mismatches, cache staleness, dead or unreachable code. |
| security        | Auth/authz, input validation, SQL/command/template injection, secret exposure, unsafe deserialization, SSRF, path traversal, privilege boundaries, multi-tenant leakage, unsafe defaults, PII handling.                            |
| concurrency     | Non-atomic read-modify-write, lock ordering, cancellation hazards, idempotency gaps, visibility/ordering bugs, retry interactions, double-submit or double-process paths.                                                          |
| architecture    | Module boundaries, dependency direction, abstractions placed in the wrong layer, ad-hoc cross-cutting concerns, structural layering breaks, new cross-module coupling.                                                             |
| maintainability | Hidden coupling, brittle abstractions, duplication, dead flags, unclear invariants, misleading names, obvious simplification not taken, accidental complexity.                                                                     |
| testing         | Coverage gaps for changed behavior, assertions that do not test what they claim, time/random/async flakiness, shared fixtures, order coupling, brittle mocks.                                                                      |
| performance     | DB query shape, N+1 queries, hot-path allocations, nested loops over user data, avoidable round trips, missing or wrong caching, sync work that should be async.                                                                   |

Do not let lens boundaries hide a real issue. If one concern spans lenses, classify it where the primary failure mode belongs and mention the supporting lens during aggregation.

## False Positives To Avoid

- Pre-existing issues this change neither introduces nor worsens.
- Stylistic nits, formatting, subjective preferences, or anything a linter would catch.
- Missing tests unless the gap creates a concrete regression risk in this change.
- General code-quality advice without a concrete failure mode.
- Issues already suppressed in code.
- Claims that cannot be verified from the diff and current code.

## Posting Findings

If posting was requested, ask which language to use unless the user already specified one. Then invoke the `add-pr-comments` skill with the final validated finding list. Do not post unvalidated findings.
