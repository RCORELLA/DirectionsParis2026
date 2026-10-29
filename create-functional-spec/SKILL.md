---
name: create-functional-spec
description: 'Turns a customer issue into a functional specification, asking clarifying functional questions when something is unclear instead of inventing it.'
disable-model-invocation: true
---
# Functional Specification Workflow

Your goal is to turn `${input:issue}` — a raw customer issue or requirement —
into a **functional specification** the customer can review and approve.

This is **NOT** the design or architecture phase. No object names, no table
structure, no AL code, no technical decisions of any kind belong here.

This is Phase 1 of a two-phase spec pipeline. Phase 2
(`create-technical-spec.prompt.md`) reads the file this prompt produces and
refuses to start unless its `Status` field says `Ready for Architecture`.
Treat that status transition as a real handoff contract, not a label.

## Guardrails

- **Never invent** a business rule, an edge case, or a missing detail. If
  it's not explicitly stated in the issue, it goes to Open Questions — it
  never becomes an assumption written as fact. There is no "obvious common
  sense" exception: if it isn't written, it isn't known.
- **Only functional questions.** A functional question asks what the
  business needs ("what happens if the equipment comes back damaged?"). A
  design question asks how to build it ("should this be a new field or a
  new table?") — those never appear here; that's Phase 2's job.
- Do not create or modify any AL object in this phase. This phase produces
  documents only.
- Markdown output goes to
  `.github/plans/${input:req_name}/${input:req_name}.functional-spec.md`.
- HTML companion goes to
  `.github/plans/${input:req_name}/${input:req_name}.functional-spec.html`,
  built using the `spec-html-report` skill (`doc-functional` variant). The
  HTML is a rendering of the same content — never add facts to the HTML
  that aren't in the Markdown, and never let them drift out of sync.
- Language for the document body and for the Step 4 question to the human:
  `${input:lang}` (default: `en`). Keep this prompt's own instructions in
  English regardless of `${input:lang}`.

## Step 0 — Check for an existing spec

Before reading the issue, check whether
`.github/plans/${input:req_name}/${input:req_name}.functional-spec.md`
already exists.

- If it doesn't exist, continue to Step 1.
- If it exists, STOP and ask whether this is:
  (a) an update to an existing spec (in which case treat the existing file
  as the starting point and the new issue text as additional input), or
  (b) an accidental name collision (in which case ask for a different
  `req_name`).
  Never overwrite an existing spec silently.

## Step 1 — Read the issue

Read `${input:issue}` in full. Identify:

- Who is asking (role/persona, if stated)
- What they want to happen
- Any business rule already stated explicitly

## Step 2 — Find the functional gaps

Go through the issue looking only for **functional** ambiguity — not design
ambiguity. Typical functional gaps:

- What happens in edge cases the issue doesn't mention (cancellations,
  partial cases, errors)
- Who needs to be notified, and when
- Whether an action has a cost, a limit, or an approval step
- What "done" looks like for this feature, in business terms

Do **not** ask about: field types, table names, page layout, object IDs, or
any implementation detail — those are out of scope here by design.

Sort every gap you find into one of two buckets:

- **Blocking** — Phase 2 cannot start without an answer (it would have to
  guess the data model or the process).
- **Nice to have** — a reasonable default exists and can be proposed, but
  still needs explicit customer confirmation before it's treated as fact.

## Step 3 — Generate the document

Create the Markdown file with this structure:

```markdown
# ${input:req_name} — Functional Specification

**Status:** Draft — pending customer review
**Requested by:** [role/persona from the issue, or "not stated"]
**Date:** [today's date]

## 1. Summary
[1-3 sentences: what the customer is asking for, in plain business language]

## 2. What we understood
[Bullet list of the business rules that ARE clear from the issue as stated]

## 3. Open Questions

### Blocking
[Numbered list. Each question must be answerable by the customer without
any technical knowledge. If none, write "None."]

### Nice to have
[Numbered list, each with a proposed default in parentheses. If none,
write "None."]

If both lists are empty, replace this whole section with:
"None — the issue is fully specified."

## 4. Acceptance Criteria
[Bullet list, in business terms, of observable conditions that confirm the
feature works as requested — e.g. "the customer receives an email within
5 minutes of the equipment being marked as damaged." These are what the
Reviewer checks against once the feature is built.]

## 5. Out of scope
[What this feature explicitly does NOT cover, to avoid scope creep]
```

Then render the HTML companion using the `spec-html-report` skill,
`doc-functional` variant, with the stamp showing the current `Status`
value verbatim.

## Step 4 — 🛑 STOP

Show the generated document in the chat (Markdown is enough here; the HTML
file is for the customer, not for this review). Do not write anything to
disk yet. List the Open Questions separately, grouped Blocking / Nice to
have, and ask explicitly, in `${input:lang}`:

"Do I send these questions to the customer before continuing, or do you
already have the answers so I can proceed?"

Wait for an explicit answer before proceeding.

## Step 5 — Apply

Only after the human confirms (either they supply answers or say there are
no open questions):

1. Fold every answer into section 2 ("What we understood"), removing the
   corresponding item from section 3.
2. Recompute `Status`:
   - If section 3 is now empty → `Status: Ready for Architecture`
   - If some Blocking questions remain unanswered → `Status: Draft — blocking questions remain`
   - If only Nice-to-have questions remain → ask the human whether to
     accept the proposed defaults; if yes, resolve them the same way, if
     no, leave `Status: Draft — pending customer review`
3. Write the `.md` file to disk.
4. Re-render the `.html` companion so the stamp matches the new `Status`.
5. If `Status` is now `Ready for Architecture`, tell the human this spec is
   ready for `create-technical-spec.prompt.md` — do not invoke it
   automatically, that is a separate, explicit step.
