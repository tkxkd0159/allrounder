---
name: improve-article
description: Improve articles by analyzing structure, improving clarity, and tightening prose while preserving meaning, accuracy.
disable-model-invocation: true
---

## Goal

Improve the article’s:

- structure
- clarity
- coherence
- flow
- readability

Preserve:

- factual meaning
- core argument
- technical accuracy
- authorial intent
- authorial voice

Do not invent facts, add unsupported claims, remove important nuance, or turn the article into generic polished writing.

## Workflow

### Step 1: Structure analysis

Divide the article into logical sections based on headings and content.

For each section, identify:

- main point
- key claims
- prerequisite ideas
- later ideas that depend on it
- whether its current position is logical

Treat information as a dependency graph.

Concepts should be introduced before they are used. If the current order is weak, propose a better order.

Output the proposed outline in this format:

```text
1. [Section title]
   Purpose:
   Key points:
   Dependencies:
   Suggested changes:
```

After presenting the outline, stop and ask the user to confirm before rewriting.

Do not continue to section rewriting until the user confirms.

### Step 2: Section rewrite

After the user confirms the outline, rewrite one section at a time.

For each section:

- improve clarity, coherence, and flow
- preserve meaning and technical precision
- remove redundancy
- make implicit logic explicit
- keep necessary technical terms
- avoid unnecessary jargon
- keep paragraphs concise, preferably under 400 characters
- exceed 400 characters only when needed for accuracy or readability

Output the rewritten section in this format:

```markdown
## [Section Title]

[Rewritten section]
```

Then summarize the changes:

```markdown
Changes:

- ...
- ...
```

After each section, stop and ask whether to approve, revise, or continue.

Do not rewrite the next section until the user confirms.

### Step 3: Final pass

After all sections are approved without remaining issues, provide the complete revised article.

Then provide:

```markdown
## Editorial Summary

Structural changes:

- ...

Clarity improvements:

- ...
```

## Accuracy rule

If a claim is unclear, unsupported, ambiguous, or technically questionable, do not silently rewrite it as if it were true and ask user to clarify.

## Style rules

Use clear, direct prose.

Prefer strong topic sentences.

Avoid filler phrases such as:

- “it is important to note”
- “as we all know”
- “in today’s world”
- “this article explores”
- “various aspects”

Prefer concrete explanations over abstract statements.

Do not make the article sound like marketing copy unless the original article is clearly marketing-oriented.
