# Playbook: applying the Temporal Entity Ledger

For an agent that has been asked to set up or run an instance of this pattern. Read [README.md](README.md) first.

## Step 0: Domain profile

Before writing anything, answer these questions. Take answers from the project if it already shows them, and ask the user otherwise. Save the result as `PROFILE.md` at the root of the instance. Every later step reads it.

```markdown
# Domain profile

- **Chunk:** what one ledger entry is (a session, a day of a channel, an article, a chapter)
- **Observed-at:** where its timestamp comes from, its granularity, and the tiebreaker for equal timestamps
- **Point of view:** whose knowledge the notes represent, and what must never leak into them (GM secrets, spoilers, private channels)
- **Entity types:** the folders under notes/ (e.g. People, Places, Organizations, Items, Plots)
- **Narrative type:** what counts as a storyline here, and which statuses it moves through (Active, Resolved, Dormant...)
- **Alias sources:** known sources of naming noise (phonetic note-taking, nicknames, handles, translations)
- **Confidence vocabulary:** e.g. observed / reported / inferred, plus any domain-specific levels
- **Commentary mode:** strict or frozen (open decision 1)
- **Human edits:** annotations only, or protected regions (open decision 2)
- **Output host:** Obsidian, plain markdown, a docs site... (decides link syntax, e.g. [[wikilinks]])
- **Exclusions:** files that sit in the instance but are not part of the ledger or notes
```

## Step 1: Scaffold

Create the layout from [README.md § Structure](README.md#structure). Add `aliases.md` with a header and an empty table. Commit.

**Migrating an existing hand-maintained vault** (as in the reference implementation): don't convert the notes. Move the per-chunk raw notes into `ledger/`, run extraction over every entry, then synthesize every entity. Diff the result against the old notes. The diff shows what the old notes asserted without evidence, and those statements become annotations or get dropped.

## Step 2: Ingest a chunk

1. Write `ledger/<observed-at>--<slug>.md` with frontmatter `observed_at`, `ingested_at`, `source` and `confidence`, then the raw content. **Don't clean it up beyond formatting.** The ledger is evidence.
2. If observed-at is earlier than the newest entry, this is a **late arrival**. Say so in the commit message and follow Step 5.

## Step 3: Extract (one subagent per entry, parallel-safe)

Contract for the extractor:

> Input: one ledger entry, `aliases.md`, the list of existing entity names.
> Output: `claims/<entry>.jsonl` in the claim schema, plus a list of **proposals**: new entities and new aliases, each with the line of evidence behind it.
>
> - One claim is one fact about one entity. Split compound sentences.
> - Resolve names through `aliases.md`. If a name doesn't resolve, propose it. Don't invent a canonical name.
> - Set `confidence` honestly. Something a character *said* is `reported`, not `observed`.
> - Set `referent_time` when the claim is about a different time than the chunk.
> - Quote memorable lines verbatim in `quote`.
> - Do not interpret. "X is probably a traitor" belongs to synthesis, not extraction. A question raised in the chunk is a `question` claim.

Then the orchestrator reviews the proposals. It accepts aliases that clearly map. It creates entities that are mentioned with substance, and parks one-line mentions as proposals until a second reference appears.

## Step 4: Plan and synthesize

**Dirty set** = entities with new claims ∪ entities linked to them (from the new claims' `related` field and from existing notes' links) whose notes mention the affected entity in their interpretation.

Contract for the synthesizer, one subagent per dirty entity:

> Input: every claim about this entity with observed-at ≤ `as_of`, sorted by (observed_at, seq). The current note, if it exists. `PROFILE.md`.
> Output: the full note.
>
> 1. **Reinterpretation first.** List the statements in the current note that the new claims contradict, strengthen or reframe.
> 2. Rewrite only those statements, plus the `status` line. Keep untouched prose as it is.
> 3. Rebuild the **timeline** section mechanically: one subsection per chunk, in order.
> 4. Keep observation and interpretation visibly separate (e.g. plain or bold text for what happened, blockquotes for analysis).
> 5. Open questions: add new ones. Mark resolved ones with the chunk that resolved them instead of deleting them.
> 6. Never reference anything after `as_of`. Never use knowledge outside the point of view in `PROFILE.md`.
> 7. Every claim-bearing sentence must be traceable to the claims you were given.

Then run the **chronicler** for the new chunk (commentary as of that chunk: callbacks to earlier chunks, flags for things missed, open questions, active threads) and the **briefer** (`briefing.md`).

## Step 5: Late arrival

1. Extract the late entry (Step 3). If it adds aliases, re-resolve existing claims. That's mechanical and needs no LLM.
2. Dirty set as in Step 4. Synthesize as of *now*.
3. If commentary is **strict**: for every chunk after the late entry that touches a dirty entity, in observed-at order, regenerate its commentary as of that chunk.
4. Regenerate the briefing.
5. Commit with a message such as `Late insert: <entry> (observed <date>), N notes and M chunk commentaries regenerated`.

## Step 6: Human corrections

A correction becomes a new ledger entry: `ledger/<now>--annotation-<n>.md`, with `kind: annotation`, `corrects: <entry or entity>`, and the correction itself. Run it through extraction like anything else. Examples:

- "General Marcus never existed. It was a note-taking error for the unnamed wolf captain" → an alias merge plus retraction claims
- "The damage numbers in session 5 are unreliable" → lowers the confidence on that entry's claims

For annotations, observed-at is the time of the **thing being corrected**, unless the correction is itself a new in-world event.

## Step 7: Verify

At least once per instance, run the **shuffle test** from [README.md § Verifying it](README.md#verifying-it) on a small subset (3–5 chunks). Record the result in `PROFILE.md`.

## Commit discipline

- One commit per ingest, and one per late insert or annotation.
- The message names the chunk and lists new entities, notes with changed status, and anything reinterpreted.
- Never mix a hand edit to a derived note into an ingest commit. If you had to hand-edit, that is a missing annotation.
