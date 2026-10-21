# Playbook: applying the Temporal Entity Ledger

For an agent that has been asked to set up or run an instance of this pattern. Read [README.md](README.md) first.

## Step 0: Domain profile

Start from the closest sketch in [domain-profiles.md](domain-profiles.md). Fill in or confirm each field from the project if it shows the answer, and ask the user otherwise. Save the result as `PROFILE.md` at the root of the instance. Every later step reads it.

```markdown
# Domain profile

- **Chunk:** what one ledger entry is (a session, a video, an article, a chapter, a thread snapshot)
- **Knowledge axis:** what `known_at` means here (publish date, session date, reading order), its granularity, and the tiebreaker
- **Narrative axis:** what `narrative_time` means here, or "same as knowledge axis"
- **Viewpoint:** whose knowledge the notes represent, and what must never leak into them (GM secrets, spoilers, private channels)
- **Entity types:** the folders under notes/
- **Narrative types:** what counts as a storyline, and its statuses (Active, Resolved, Dormant...)
- **Alias noise:** known sources of naming noise (phonetic notes, captions, handles, epithets, edition renames)
- **Authority & precedence:** source ranking, and how contradictions resolve (newer edition wins; corrections supersede; unreliable narrators)
- **Confidence vocabulary:** e.g. observed / reported / inferred
- **Volume & batching:** expected chunks per day; filters at ingest; synthesis batch size
- **Commentary period:** per chunk, per day, per week, per book
- **Output host:** Obsidian, plain markdown, a docs site... (decides link syntax)
- **Exclusions:** files in the instance that are not part of the ledger or notes
```

## Step 1: Scaffold

Create the layout from [README.md § Structure](README.md#structure). Add `aliases.md` with a header and an empty table. Commit.

**Migrating an existing hand-maintained vault** (as in the reference implementation): don't convert the notes. Move the raw per-chunk material into `ledger/`, run extraction over every entry, then synthesize every entity. Diff the result against the old notes. The diff shows what the old notes asserted without evidence, and those statements become annotations or get dropped.

**Building on an existing source-faithful archive** (e.g. youtube-synthesizer output): those notes already *are* ledger entries. Index them into `ledger/` by `known_at` (symlink or manifest) instead of copying them.

## Step 2: Ingest

Write `ledger/<known_at>--<slug>.md` with frontmatter `known_at`, `narrative_time` (if it differs and is known), `ingested_at`, `source`, `authority` and `confidence`, then the raw content. **Don't clean it up beyond formatting.** The ledger is evidence.

If `known_at` is earlier than the newest entry, the entry is **late**. The procedure is identical, but Step 5 reaches further. Say so in the commit message.

Re-fetches of mutable sources (edited posts, updated articles) are **new** entries that reference the original. Never overwrite.

## Step 3: Extract (one subagent per entry, parallel-safe)

Contract for the extractor:

> Input: one ledger entry, `aliases.md`, the list of existing entity names, `PROFILE.md`.
> Output: `claims/<entry>.jsonl` in the claim schema, plus **proposals**: new entities, new aliases, and same-event links, each with the evidence behind it.
>
> - One claim is one fact about one entity. Split compound sentences.
> - Resolve names through `aliases.md`. If a name doesn't resolve, propose it. Don't invent a canonical name.
> - Set `source`, `authority` and `confidence` honestly. Something a character or commenter *said* is `reported` and attributed.
> - Set `narrative_time` when the claim is about a different time than the chunk. Fuzzy times are allowed.
> - Set `reframes: true` when a claim contradicts or recontextualizes something central in the existing entity list (a hidden identity, a retcon, a betrayal revealed, a retraction).
> - **Never resolve contradictions.** Record what this source says.
> - Do not interpret. "X is probably a traitor" belongs to synthesis. A question the chunk raises is a `question` claim.

Then the orchestrator reviews the proposals. It accepts aliases that clearly map. It creates entities that are mentioned with substance, and parks one-line mentions until a second reference appears. If new aliases were accepted, it re-resolves existing claims, which is mechanical.

## Step 4: Synthesize to a fixpoint

**Initial dirty set** = entities with new claims. If any claim has `reframes: true`, also include every entity linked to the reframed one, or go straight to full re-synthesis when the reframe is central.

Contract for the synthesizer, one subagent per dirty entity:

> Input: every claim about this entity with `known_at` ≤ `as_of`, the current note if it exists, `PROFILE.md`. For final notes, `as_of` is everything.
> Output: the full note, plus `interpretation_changed: true|false`.
>
> 1. **Reinterpretation first.** List the statements in the current note that the claims now contradict, strengthen or reframe.
> 2. Rewrite only those statements, plus the `status` line. Keep untouched prose as it is.
> 3. Rebuild the **timeline** mechanically, ordered by `narrative_time` (falling back to `known_at`, then `seq`). Mark fuzzy placements.
> 4. **Resolve contradictions** by the profile's precedence rules. Cite both sources when the conflict matters to a reader.
> 5. Keep observation and interpretation visibly separate (e.g. plain or bold text for what happened, blockquotes for analysis).
> 6. Open questions: add new ones. Mark resolved ones with the entry that resolved them instead of deleting them.
> 7. Never use a claim after `as_of`, or knowledge outside the viewpoint.
> 8. Every claim-bearing sentence must be traceable to the claims you were given.
> 9. `interpretation_changed` is true only if the status, overview or analysis materially changed. Timeline additions alone don't count.

**Loop:** for every note returned with `interpretation_changed: true`, add the notes that link to it (and that it links to) to the next dirty set. Repeat until a pass changes no interpretation. **Cap at 5 passes.** If the cap is hit, stop and report which notes are still changing, because that usually means two notes contradict each other and a human needs to rule.

Filter the claims by `as_of` in code before the subagent sees them. Don't rely on the prompt for that.

## Step 5: Chronicle

1. Write the new chunk's commentary as of itself: callbacks to earlier chunks, flags for things missed, open questions, active threads. For batched domains, write it per commentary period instead.
2. **If the entry is late:** for every chunk *after* it on the knowledge axis whose commentary touches a note that changed interpretation in Step 4, regenerate that commentary as of its own chunk, in knowledge-axis order.

## Step 6: Brief

Regenerate `briefing.md`: a compressed "read this first" covering current state, active narratives, open questions and anything reframed in this ingest.

## Step 7: Human corrections

A correction becomes a new ledger entry: `ledger/<known_at>--annotation-<n>.md`, with `kind: annotation`, `corrects: <entry or entity>`, and the correction itself. Run it through Steps 3–6 like anything else. Examples:

- "General Marcus never existed. It was a note-taking error for the unnamed wolf captain" → an alias merge plus `retraction` claims
- "The damage numbers in session 5 are unreliable" → lowers the confidence on that entry's claims
- "In this wiki, the 2nd-edition version of the god's death is canon" → a precedence rule. This one also belongs in `PROFILE.md`

For an annotation, `known_at` is the position of the **thing being corrected**, so the correction applies from there onward.

## Step 8: Verify

At least once per instance, run the **shuffle test** and the **reframe test** from [README.md § Verifying it](README.md#verifying-it) on a small subset (3–5 chunks). Record the results in `PROFILE.md`.

## Commit discipline

- One commit per ingest, or per batch in high-volume domains.
- The message names the entries; flags late inserts and reframes; and lists new entities, notes with changed status, the fixpoint pass count, and regenerated commentary.
- Never mix a hand edit to a derived note into an ingest commit. If you had to hand-edit, that is a missing annotation.
