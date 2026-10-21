# Temporal Entity Ledger

> Status: **Draft**. Design decisions settled. Not yet validated by a second implementation.

## Intent

Take a stream of time-ordered, messy, partial observations, such as video series, sourcebooks, news feeds, community threads, novels or game sessions, and keep a **graph of entity notes** up to date from it. The graph covers people, places, organizations and objects, and also **narratives** (plots, storylines, incidents, debates, arcs). Each note shows the entity's history and the current understanding of it.

Observations can arrive **in any order**: a backfilled archive, a forgotten session, a prequel read after the series. Whatever the ingest order, the final product must represent **the total body of observations**, as if everything had been processed in its proper order. Late data is allowed to change the narrative fundamentally, and when it does, everything it reframes is rewritten.

## Also known as

Chronicle → wiki. Incremental knowledge graph. Event-sourced notes.

## Motivation

Raw logs answer "what happened in entry N". They are bad at answering "what do we know about X", "what is still unresolved" or "what changed our minds". Entity notes answer those questions, but if an agent maintains them by hand-editing state, three things go wrong:

1. **Interpretation drifts away from evidence.** Nobody can tell which sentences were observed and which were inferred.
2. **Later evidence doesn't revise earlier conclusions.** Agents append readily but rarely go back and change what they already wrote.
3. **Late data has nowhere to go.** A prequel or a backfilled archive lands in a state that already "knows the future", and there's no principled way to rewrite around it.

The fix is to make the **ledger** the only source of truth and to treat every note as a **derived view** that can be regenerated from it.

## Applicability

Use it when:

- the input arrives in chunks, each with a meaningful position in time
- the same nouns recur across chunks, under inconsistent names
- the questions people care about are about entities and storylines, not individual chunks
- later information regularly changes what earlier information means
- chunks can arrive late, be backfilled, be corrected or be retconned

Don't use it for one-shot summaries, or when the stream has no recurring entities.

Target domains (filled-in profiles are in [domain-profiles.md](domain-profiles.md)):

| Domain | Chunk | Knowledge axis | Narrative axis | What makes it hard |
|---|---|---|---|---|
| YouTube series | video | publish date | publish date (or a topic's own chronology) | backfilling a back catalog; recurring concepts named differently |
| RPG setting from sourcebooks | book / chapter | publication (and edition) | in-world calendar | retcons across editions; read out of order |
| Local news via RSS | article | publication time | event time | volume; the same story from many outlets; corrections |
| Reddit community | thread or comment batch | post/comment time | post/comment time | mutable sources (edits, deletions, late comments); many unreliable voices |
| Book-series wiki | chapter | publication (reading) order | in-universe chronology | prequels that reframe everything; unreliable narrators |
| Tabletop campaign | session | session date | session date (plus in-world revelations) | phonetic note-taking; secrets revealed late |

## Core ideas

### 1. Two axes of time, plus ingestion

| Axis | Meaning | Used for |
|---|---|---|
| **Knowledge axis** (`known_at`) | when this information became available to the viewpoint: publish date, session date, post time | `as_of` views: what was knowable at T. Spoiler-free reads. Ordering the per-chunk commentary |
| **Narrative axis** (`narrative_time`) | where the described events sit in the world's own timeline | ordering timelines inside notes. "What happened before what" |
| **Ingested-at** | when the system processed it | audit only. **Never** affects the output |

They often coincide (news, Reddit, most of D&D), and the profile can declare that. They come apart exactly where it gets interesting. A prequel is *known late* but *happens early*. A Legend Lore vision is *known* in session 15 but *happened* a century earlier. A news retrospective covers last year's events.

Ingestion order is never an axis. That is the whole promise: **the output is a function of the ledger's contents, not of the order they arrived in.**

### 2. Observation vs. interpretation

Every derived document keeps what was observed visibly separate from what is inferred from it. That is the precondition for revising interpretations safely.

### 3. Narratives are entities

Plots, storylines, controversies and arcs get their own notes, with status and priority. The abstract nouns are often what readers actually want.

### 4. Two layers of processing

| Layer | Input → output | Depends on order? | Cached? |
|---|---|---|---|
| **Extraction** | one ledger entry + the alias registry → atomic **claims** | no | yes, one file per entry |
| **Synthesis** | all claims about an entity ≤ `as_of`, ordered → that entity's note | **yes** | no, regenerated when dirty |

Almost all of the expensive reading happens in extraction, and extraction is order-independent. Synthesis reads compact claims, not raw chunks. That makes re-synthesis, even of the whole graph, affordable.

### 5. Reframing propagates to a fixpoint

Late data can do more than add facts. It can **reframe**: a prequel reveals the hero's mentor was the villain all along. That change doesn't stay one hop away. Every note whose interpretation leaned on the old framing has to be rewritten, and then the notes that leaned on *those*.

So the dirty set is computed by **iterating to a fixpoint**: synthesize the dirty notes, collect the ones whose *interpretation* materially changed (status, overview, analysis, as opposed to timeline additions only), add their linked notes to the dirty set, and repeat until a pass changes no interpretation. A depth cap guards against oscillation. Hitting it is a signal for a human, not something to ignore.

When the extractor marks a claim as `reframes: true` (it contradicts or recontextualizes an established understanding), the planner may skip straight to **full re-synthesis**. Because synthesis works from claims, that costs one pass over the graph, not a re-read of the source.

### 6. Authority and contradiction

Sources disagree: a later edition retcons an earlier one, an outlet issues a correction, a commenter contradicts the OP, an unreliable narrator lies. **Extraction never resolves contradictions.** It records both claims with their `source` and `authority`. **Synthesis** resolves them using the precedence rules in the domain profile ("newer edition wins", "corrections supersede", "the narrator in book 2 is unreliable"), and surfaces the conflict when it matters.

### 7. Append history, rewrite the summary

Inside a note, the **timeline** is mechanically derived from claims, ordered by narrative time. The **status, overview, analysis and open questions** are interpretation and get rewritten on synthesis.

### 8. `as_of` views

Any document can be synthesized `as_of` a point on the knowledge axis, using only claims known by then. The **final product** is synthesized as of *everything*. Per-chunk commentary is synthesized as of its own chunk. A spoiler-safe wiki ("everything through book 3") is just another `as_of`.

## Structure

```
PROFILE.md                     # domain profile: axes, viewpoint, entity types, precedence rules
ledger/                        # immutable. One file per observation chunk
  2026-08-13--session-13.md    #   frontmatter: known_at, narrative_time?, ingested_at, source, authority
  2026-10-20--annotation-07.md #   human corrections are ledger entries too
claims/                        # extraction cache. One file per ledger entry
  2026-08-13--session-13.jsonl
aliases.md                     # canonical name <- variants. Derived from claims and annotations
notes/                         # synthesized, as of everything
  People/  Places/  Organizations/  Items/  Plots/
chunks/                        # per-chunk (or per-period) commentary, each as of itself
  Session 13.md
briefing.md                    # "read this first": the current state, compressed
```

The ledger files are named by knowledge axis. Ingestion order lives only in git history.

### Claim schema

One JSON object per line in `claims/<entry>.jsonl`:

```json
{"entry": "2026-08-13--session-13", "seq": 14,
 "known_at": "2026-08-13", "narrative_time": null,
 "entity": "Samus", "kind": "event|attribute|relation|status|alias|question|retraction",
 "text": "Offered to free Martavis in exchange for the helms of three lieutenants",
 "related": ["General Martavis", "Farraday", "Celine", "Draymon"],
 "source": "session notes (player)", "authority": "primary",
 "confidence": "observed|reported|inferred",
 "reframes": false,
 "quote": "Bring me their helms."}
```

- `seq` preserves order within a chunk.
- `narrative_time` may be fuzzy ("a century before", "between book 2 and 3"). Synthesis orders fuzzy times as well as it can and says so.
- `confidence` is about the observation ("a character said it", "a player guessed"). `authority` is about the source's standing (canon vs. fan, primary vs. aggregator, OP vs. random commenter).
- `retraction` claims (from annotations or source corrections) cancel earlier claims by reference.

### Note frontmatter

```yaml
type: person | place | organization | item | plot
status: one line, current as of `as_of`
as_of: everything            # or a ledger entry id for a historical view
derived_from: [2026-03-12--session-05, ..., 2026-09-10--session-16]
aliases: [Almondrill]
```

## Participants

| Role | Reads | Writes | Notes |
|---|---|---|---|
| **Ingester** | raw input | `ledger/` | normalizes format, assigns axes and source, never interprets |
| **Extractor** | one ledger entry, `aliases.md`, entity list, `PROFILE.md` | `claims/<entry>.jsonl`, proposals | one per entry, in parallel. Flags `reframes` |
| **Planner** | new claims, notes' `derived_from` and links | the dirty set, and drives the fixpoint loop | deterministic apart from "did the interpretation change" |
| **Synthesizer** | one entity's ordered claims ≤ `as_of`, its current note, `PROFILE.md` | that note, plus a changed-interpretation flag | one per dirty entity, in parallel |
| **Chronicler** | one chunk or period, notes as of that point | `chunks/…` | commentary: callbacks, flags, open questions |
| **Briefer** | the latest notes and open questions | `briefing.md` | the warm start for humans and the next run |

In a small deployment, one agent plays every role in sequence. The roles still matter, because each one has a different contract.

## Process

### Ingest (the same procedure whether the entry is new or late)

1. **Ingest** the chunk into `ledger/` at its knowledge-axis position.
2. **Extract** claims. Propose new entities and aliases (weak evidence produces a *proposal*, not a note). If new aliases were accepted, re-resolve existing claims. That step is mechanical.
3. **Plan.** The initial dirty set is every entity with a new claim. If any claim has `reframes: true`, consider full re-synthesis.
4. **Synthesize to a fixpoint** as of everything (see [§5](#5-reframing-propagates-to-a-fixpoint)).
5. **Chronicle.** Write the new chunk's commentary as of itself. If the entry is **late** on the knowledge axis, regenerate the commentary of every later chunk that touches a note whose interpretation changed, each as of its own chunk, in order.
6. **Brief**: regenerate `briefing.md`.
7. **Commit** once per ingest. The diff is the audit log of what the system changed its mind about. Late inserts and reframes are called out in the message.

The new-vs-late split collapses into one procedure. "Late" only changes how far step 5 reaches.

### What a prequel does, concretely

A book-series wiki has books 1–5 ingested. A prequel (book 0) is published after book 5.

- **Knowledge axis:** book 0 sits *after* book 5, because that's when readers learned it. The commentary for books 1–5 stays as it was, since nobody reading in publication order knew it yet. A profile that defines the knowledge axis as *chronological reading order* puts book 0 first instead, and then every later chunk's commentary cascades. The profile chooses.
- **Narrative axis:** book 0's events sit *before* book 1's in every timeline.
- **Final notes:** synthesized as of everything. The reframing propagates until it stops, so the wiki reflects the total work.

## Invariants

1. Ledger entries are immutable. Corrections, retcons and edits at the source are new entries that reference what they supersede.
2. Every note and chunk commentary can be regenerated from `ledger/` alone.
3. Nothing synthesized as of T uses a claim with `known_at` > T.
4. **Ingestion order does not change the output**, up to claim-equivalence: the same claims, timelines, status lines and open-question set. Prose may differ.
5. Every non-trivial statement in a note traces to at least one claim.
6. Contradictions are resolved in synthesis by profile rules, never silently in extraction.

## Verifying it

The **shuffle test**: ingest a set of chunks in order, then again in a random order with some held back and inserted late. Compare the claim sets, timelines, `status` lines and open questions. They should match. Prose differences are expected.

The **reframe test**: ingest a set, then add one entry that reframes a central entity (a prequel, a retraction, a reveal). Check that every note whose interpretation depended on the old framing changed, and that unrelated notes did not.

## Consequences

**Benefits**
- Late, backfilled, corrected and retconned data become routine.
- `as_of` views come free: spoiler-safe wikis, "what did we know in March".
- Interpretation can be revised wholesale without losing provenance.
- Extraction and synthesis both parallelize per entry and per entity.

**Costs**
- Notes are never hand-edited. Humans correct through annotations.
- LLM synthesis is nondeterministic, so regeneration produces prose churn. Minimal-edit prompting against the current note keeps diffs readable.
- A reframe can rewrite much of the graph. That's the point, but it costs a synthesis pass per affected note.
- High-volume streams (news, Reddit) need batching and summarized commentary periods to stay affordable.

## Failure modes

| Failure | Symptom | Guard |
|---|---|---|
| Entity resolution drift | a phantom entity is created from a misheard name, or two entities are merged | proposals before creation; a reviewable `aliases.md`; annotations to split or merge |
| Hindsight leak | early commentary "knows" a later reveal | enforce `as_of` filtering in code, not just in the prompt |
| Under-revision | a stale framing survives a reframe | the fixpoint loop; the explicit reinterpretation step; the reframe test |
| Over-revision / oscillation | prose rewritten every run, or two notes flip each other | minimal-edit synthesis; the fixpoint depth cap escalates to a human |
| Silent contradiction resolution | one source's version wins without a trace | contradictions resolved only in synthesis, by profile rules, and cited |
| Duplicate observations | the same news story from five outlets counted as five events | the extractor proposes "same event as" links; synthesis treats them as corroboration |
| Mutable sources | a Reddit edit or news update silently changes evidence | re-fetches become new ledger entries; nothing is overwritten |
| Silent misordering | an undated chunk is guessed into the wrong place | require `known_at`; record its confidence; never guess silently |
| Noisy chunk | chaotic input yields confident nonsense | `confidence` per claim; the chunk records its own reliability |

## Decisions

1. **Late data may reframe anything, and the system follows through.** Notes are re-synthesized to a fixpoint as of everything. Chunk commentary is strict: commentary after the late entry's knowledge-axis position is regenerated wherever it touches a reinterpreted note.
2. **Humans edit through the ledger.** A correction is an annotation entry and is processed like any other. Derived notes are never hand-edited. A hand edit is treated as a missing annotation.

## Related patterns

- **Event sourcing.** The ledger is the event log, and notes are projections.
- **Bitemporal data.** The knowledge axis is transaction time from the viewpoint's perspective, and the narrative axis is valid time.
- **Materialized views with incremental maintenance.** The dirty set and the fixpoint loop are view maintenance.
- **Zettelkasten / literature → permanent notes.** The same two-layer split, done by hand.

## Known uses

- [Cael Stormcrow campaign vault](reference-implementation.md): D&D 5e session notes into an Obsidian entity graph. In-order only today.
- [youtube-synthesizer / article-synthesizer](https://github.com/devonjones/devon-claude-skills/blob/main/docs/youtube-wiki-design.md) (devon-claude-skills): implements the **ledger and extraction half**. `sources/<type>/<ingested-date>-<title>/` is an immutable, source-faithful ledger. The frontmatter already separates `source_published_date` (knowledge axis) from `ingested_date`. "Source-faithful, do not interpret" is the extractor contract. The vault root, hands-off for skills, is the synthesis layer, which a human does today. Its planned Phase 4 cross-source linker and MOC maintenance are the first steps toward automated synthesis. Note that it names folders by **ingested** date, which is fine for a reading log but is not this pattern's ordering.
