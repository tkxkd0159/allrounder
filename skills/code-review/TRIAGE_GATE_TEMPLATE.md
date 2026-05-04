# Triage Gate Template

## Emission

Skip everything in this file up to (but not including) the first `<!-- TEMPLATE: HEADER -->` marker line. Then:

1. Emit the HEADER block (between `<!-- TEMPLATE: HEADER -->` and `<!-- /TEMPLATE: HEADER -->`) once, with file-level placeholders substituted.
2. For each surviving finding in stable-ID order, emit one FINDING block (between `<!-- TEMPLATE: FINDING -->` and `<!-- /TEMPLATE: FINDING -->`), with per-finding placeholders substituted.
3. Do **not** emit any `<!-- TEMPLATE: ... -->` or `<!-- /TEMPLATE: ... -->` marker line in the output.

## Placeholders

**File-level (used in HEADER):**

- `<BASE_BRANCH>` — e.g. `origin/main`
- `<MERGE_BASE_SHORT>` — first 7 chars of `git merge-base` SHA
- `<HEAD_SHA>` — full HEAD SHA
- `<ISO_8601_UTC>` — current UTC time, ISO-8601 (e.g. `2026-05-05T14:30:00Z`)
- `<LOOP>` — `1` on first emission; incremented each round

**Per finding (used in FINDING):**

- `<id>` — `1`, `2`, `3`, … (stable across loops; never renumbered)
- `<SEV>` — one of `critical`, `high`, `medium`, `low`
- `<CONF>` — one of `high`, `medium`, `low`
- `<Title>` — short finding title
- `<lens>` — one of `correctness`, `security`, `concurrency`, `architecture`, `maintainability`, `testing`, `performance`
- `<file:line>` — e.g. `internal/cache/lru.go:142` or `api/exports.go:88-95`
- `<description>` — one-paragraph failure mode
- `<fix>` — actionable fix

---

<!-- TEMPLATE: HEADER -->
<!--
  Triage draft — edit, then say `proceed` (or `cancel`) in chat.

  For each finding, set `action:` to one of:
    keep — include when posting
    drop — exclude (won't be posted)
    dive — re-investigate; a fresh subagent will replace this finding,
           or drop it if it can't substantiate. Use `notes:` to tell
           the second pass what the first missed.

  Heading format:  F<id> — [SEV/CONF] Title
    SEV  (severity):   critical | high | medium | low — failure-mode impact
    CONF (confidence): high | medium | low — strength of evidence

  IDs are stable across loops. Dives that split into multiple findings get
  fresh IDs appended at the bottom. The `loop:` value in frontmatter
  increments each round.
-->

---
review-base: <BASE_BRANCH>
merge-base: <MERGE_BASE_SHORT>
head-sha: <HEAD_SHA>
generated-at: <ISO_8601_UTC>
loop: <LOOP>
---
<!-- /TEMPLATE: HEADER -->

<!-- TEMPLATE: FINDING -->
## F<id> — [<SEV>/<CONF>] <Title>

**Lens:** <lens>
**Location:** <file:line>
**Failure mode:** <description>
**Fix:** <fix>

```triage
action: keep
notes:
```
<!-- /TEMPLATE: FINDING -->
