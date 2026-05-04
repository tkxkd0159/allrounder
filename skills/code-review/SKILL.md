---
name: code-review
description: Use when reviewing branch or PR changes against a base branch for correctness, security, concurrency, architecture, maintainability, testing, and performance risks.
argument-hint: '[--base BRANCH] [--comment LANGUAGE]'
disable-model-invocation: true
---

# Code Review

Review the current branch or PR against a base branch with adversarial, evidence-based scrutiny. Prefer concrete regressions over style feedback.

Dispatching fresh subagent per activated review lens. We delegate tasks to specialized review agents with isolated context.

## Inputs

- Default base branch: `origin/main`
- `--base BRANCH`: compare against `BRANCH`
- `--comment [LANGUAGE]`: after the review, enter a file-based triage gate. The agent writes findings to `<repo-root>/.code-review/<head-sha-short>.md`; the user edits per-finding actions (`keep | drop | dive`) and optional `notes:`, then types `proceed` or `cancel` in chat. Only the user-approved subset is posted via `add-pr-comments`. Use `LANGUAGE` for comment text, or English if omitted. Without `--comment`, no draft file is created and no gate runs; the report is the terminal artifact.

## Workflow

When `--comment` is set, posting is gated by a file-based triage step (step 8). The agent never calls `add-pr-comments` or any `gh api` write before the user has explicitly approved the kept finding set by typing `proceed` in chat.

1. Pin the comparison. Set `BASE_BRANCH` from `--base` or use `origin/main`, then:

   ```bash
   HEAD_SHA=$(git rev-parse HEAD)
   MERGE_BASE=$(git merge-base "$BASE_BRANCH" "$HEAD_SHA")
   ```

   Review `"$MERGE_BASE".."$HEAD_SHA"` for the entire pass.

2. Inventory the diff: changed files, shortstat, added/deleted files, and diff body. Ignore generated files, vendored code, lockfiles, snapshots, and docs-only changes unless they affect runtime, build, or security.

3. Choose lenses. Run `correctness` always; run other lenses when the trigger matches. Record activated and skipped lenses with one-line reasons.

   | Lens              | Trigger                                                                                                           | Review for                                                                                                         |
   | ----------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
   | `correctness`     | Always                                                                                                            | logic, null handling, contracts, stale callers, data corruption, migrations, cache staleness, unreachable code     |
   | `security`        | Auth, tokens, secrets, validation, deserialization, exec/subprocess, SQL, network I/O, endpoints, middleware, PII | auth/authz, injection, unsafe deserialization, SSRF, path traversal, tenant or PII leaks                           |
   | `concurrency`     | Shared state, locks, async, threads, queues, retries, cancellation, idempotency                                   | non-atomic updates, lock ordering, cancellation hazards, retry interactions, double-submit or double-process paths |
   | `architecture`    | 3+ files changed, new modules, public exports, import-graph or boundary changes                                   | boundary breaks, dependency direction, wrong-layer abstractions, cross-module coupling                             |
   | `maintainability` | 100+ added lines, new abstractions, duplication, hidden coupling, rename-heavy diffs                              | brittle abstractions, unclear invariants, accidental complexity, obvious simplification not taken                  |
   | `testing`         | Test changes, production changes without adjacent tests, brittle mocks, flaky assertions                          | coverage gaps for changed behavior, weak assertions, flakiness, fixture pollution                                  |
   | `performance`     | DB queries, loops over user data, hot paths, network calls, caching, avoidable round trips                        | query shape, N+1s, hot-path allocations, nested loops over user data, missing or wrong caching                     |

4. Review each activated lens:
   - Gather evidence first: cited code ranges, prior vs current behavior, touched callers, reachable inputs, unknowns.
   - Create findings only from evidence. Include title, severity, confidence, file/line, failure mode, and fix.
   - Self-critique each finding from the PR author's perspective; keep, downgrade, or drop it.

5. Validate every surviving finding directly in the current code and reviewed diff. Drop anything not supported by the cited range. Move plausible but unverified concerns to residual risks.

6. Aggregate duplicate or overlapping findings, keep the strongest severity/confidence, and run one cross-lens check for combined risks.

7. Report in this order:
   - Executive summary: base, merge-base SHA, changed-file count, activated/skipped lenses, blockers, residual risks
   - Findings by lens: title, `SEV/CONF`, `file:line`, failure mode, fix
   - Cross-cutting risks, if any
   - Approval: exactly one of `ready to approve`, `ready after fixes`, or `not ready`

### Follow-up handling (without `--comment`)

When `--comment` is not set, step 7 is the terminal output and step 8 (the triage gate) does not run. The agent should still respond to user follow-ups conversationally:

- **Clarification questions** ("why is F2 high severity?", "did you check the auth path?"): answer from the cited evidence; if the question is outside the diff, say so.
- **Deep-dive requests** ("F4 looks shallow — did you consider the lock at line 138?"): dispatch a fresh subagent with the original finding, the cited code (file + line range with surrounding context), the original lens, and the user's hint as guidance. Report the refined finding inline. No file is written.
- **Additional lens runs** ("can you do a security pass too?"): activate the requested lens against the same merge-base / HEAD pair and report.

This is asymmetric with the `--comment` path on purpose: with `--comment` there is a permanent posted action that requires a structured commitment protocol; without it, findings are ephemeral and natural conversation suffices.

8. Triage gate (only when `--comment` is set). Halt before any `gh api` write. Posting is forbidden until the user explicitly types `proceed` in chat with the draft in an acceptable state.

   **Setup (first iteration only):**

   - Compute `HEAD_SHA_SHORT=$(git rev-parse --short=7 HEAD)`.
   - File path: `<repo-root>/.code-review/<HEAD_SHA_SHORT>.md`. Repo root is the output of `git rev-parse --show-toplevel`.
   - If the file already exists for this SHA, ask the user "resume existing draft or start fresh?" Do not silently overwrite. On "fresh", overwrite; on "resume", skip the initial emission and proceed straight to the loop body.
   - Ensure `.code-review/` is in the project's `.gitignore`. If missing, append the line. Do not commit changes to `.gitignore` automatically — leave that to the user.
   - Assign each surviving finding a stable ID (`F1`, `F2`, …) in report order. IDs are not reassigned across loops.

   **Initial emission format:**

   The canonical format is in [`TRIAGE_GATE_TEMPLATE.md`](./TRIAGE_GATE_TEMPLATE.md) (next to this file). To emit:

   - Read the template. Skip everything up to (but not including) the first `<!-- TEMPLATE: HEADER -->` marker line.
   - Emit the HEADER block (between `<!-- TEMPLATE: HEADER -->` and `<!-- /TEMPLATE: HEADER -->`) once, substituting file-level placeholders: `<BASE_BRANCH>`, `<MERGE_BASE_SHORT>`, `<HEAD_SHA>`, `<ISO_8601_UTC>`, `<LOOP>` (set to `1` on first emission).
   - For each surviving finding in stable-ID order, emit one FINDING block (between `<!-- TEMPLATE: FINDING -->` and `<!-- /TEMPLATE: FINDING -->`), substituting per-finding placeholders: `<id>`, `<SEV>`, `<CONF>`, `<Title>`, `<lens>`, `<file:line>`, `<description>`, `<fix>`.
   - Do not emit any `<!-- TEMPLATE: ... -->` marker comment.

   See the template's "Placeholders" section for details on each placeholder.

   After writing the file, message the user: "Triage draft at `<path>`. Edit `action:` per finding (and `notes:` for any dives), then say `proceed` or `cancel`."

   **Loop body** (each time the user says `proceed`, or any equivalent affirmation):

   1. **Read** the draft file. If missing → tell the user "draft was deleted; aborting" and exit without posting.
   2. **HEAD-SHA check:** compare `git rev-parse HEAD` against the frontmatter `head-sha`. If they differ, halt and ask the user (a) re-run from scratch (new SHA, new draft, current findings discarded) or (b) proceed against the original SHA, with a warning that `add-pr-comments` will post against current HEAD and line anchors may have shifted.
   3. **Parse** frontmatter and each finding's fenced `triage` block.
   4. **Validate** before applying any action (fail fast — never apply partial actions):
      - Every finding has a `triage` block. If any are missing, halt with: "F<id> is missing its `triage` block; restore it and try again." Do not re-emit.
      - Every `action:` value is one of `keep | drop | dive` (case-insensitive). On invalid value, halt with: "F<id> has invalid action `<value>`; valid values are keep, drop, dive." Do not re-emit.
      - Every `dive` finding has a non-empty `notes:` value. On empty notes, halt with: "F<id> is marked dive but has empty notes — please tell the second pass what the first missed." Do not re-emit.
   5. **Apply actions:**
      - `drop` → remove the finding from the working set.
      - `dive` → dispatch one fresh subagent per dive, in parallel via the Agent tool. Each subagent receives, in its prompt: the original finding's full content, the cited code (file + line range with surrounding context), the original lens (correctness / security / concurrency / etc.), the user's `notes:` text, and instructions to either (a) produce a refined finding using the same `[SEV/CONF] Title` + Lens / Location / Failure mode / Fix shape, or (b) report "cannot substantiate" with a one-line reason. A dive may produce 2+ refined findings if the original was conflating distinct bugs.
      - `keep` → unchanged in working set.

      Wait for all dive subagents in this iteration to return before proceeding to step 6.
   6. **Decide next step:**
      - **No `dive` actions in this iteration** (only `keep` / `drop`): apply drops; the remaining set is final. Print a one-line summary (e.g., "dropped F2, F5 per user; posting F1, F3, F4"). Hand off to `add-pr-comments` with the kept set and `LANGUAGE` (or English default). Preserve the draft file. Exit. Do not re-emit.
      - **Any `dive` action**: re-emit the file with:
        - kept findings unchanged (preserve any descriptive edits the user made outside the `triage` block)
        - refined dives in place of their originals; if a dive produced multiple findings, the first replaces the original ID and the rest are appended with fresh IDs at the bottom
        - dropped findings replaced by `<!-- F<id> dropped by user -->` markers at their old positions
        - "cannot substantiate" dive results replaced by `<!-- F<id> dropped: dive could not substantiate — <reason> -->` markers
        - `loop:` counter incremented
        - every `action:` reset to `keep` and every `notes:` cleared (the user starts fresh each round)

      Then message the user "loop <N> draft updated at `<path>`; review and `proceed` or `cancel`" and wait.

   **On `cancel`:** delete the draft file. No `gh api` write. Exit.

   **All findings dropped:** if after applying drops the working set is empty, do not invoke `add-pr-comments`. Tell the user "nothing to post (all findings dropped)" and exit. The draft file is preserved.

   **Concurrent dives:** wait for all dive subagents in an iteration to return before re-emitting. If the user says `proceed` again while a previous iteration's dives are still running, respond "still working on previous loop's dives; one moment" and do not start a new iteration.

## Severity

- `critical`: exploitable security issue, auth bypass, data loss, corruption, severe outage
- `high`: likely production failure or serious regression
- `medium`: real bug under a plausible edge case
- `low`: non-trivial issue worth fixing, not a blocker

## Confidence

- `high`: directly supported by cited code
- `medium`: strong evidence with one assumption
- `low`: plausible but speculative; prefer residual risk over a low-confidence finding

## Avoid

- Pre-existing issues this change neither introduces nor worsens
- Style nits, formatting, subjective preferences, or linter-level issues
- Missing tests without a concrete regression risk
- General advice without a failure mode
- Claims that cannot be verified from the diff and current code
