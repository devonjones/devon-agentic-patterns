# Temporal Entity Ledger

> Status: **Draft**. Two design decisions are still open. See [Open decisions](#open-decisions).

## Intent

Take a stream of time-ordered, messy, partial observations, such as game sessions, chat logs, news, chapters or meeting notes, and keep a **graph of entity notes** up to date from it. The graph covers people, places, organizations and objects, and also **narratives** (plots, storylines, incidents, debates). Each note shows the entity's history and the current understanding of it.

Observations can arrive **out of order**. When an old observation turns up late, the result must be **as if everything had been ingested in temporal order**.

## Also known as

Chronicle → wiki. Incremental knowledge graph. Event-sourced notes.

## Motivation

Raw logs answer "what happened on day N". They are bad at answering "what do we know about X", "what is still unresolved" or "what changed our minds". Entity notes answer those questions, but if an agent maintains them by hand-editing state, three things go wrong:

1. **Interpretation drifts away from evidence.** Nobody can tell which sentences were observed and which were inferred.
2. **Later evidence doesn't revise earlier conclusions.** Agents append readily but rarely go back and change what they already wrote.
3. **Late data has nowhere to go.** An observation from two months ago lands in a state that already "knows the future", so the notes can't be made consistent.

The fix is to make the **ledger** the only source of truth and to treat every note as a **derived view** that can be regenerated from it.

## Applicability

Use it when:

- the input arrives in chunks, each with a meaningful time
- the same nouns recur across chunks, under inconsistent names
- the questions people care about are about entities and storylines, not individual chunks
- later information regularly changes what earlier information means
- chunks can arrive late, be backfilled, or be corrected

Don't use it for one-shot summaries, or when the stream has no recurring entities.

| Domain | Chunk | Entities | Narratives |
|---|---|---|---|
| Tabletop campaign | session notes | NPCs, places, items, factions | plots, mysteries |
| Discord / Slack | day or week of messages | people, projects, decisions | debates, incidents |
| News | article or daily feed | actors, orgs, places | storylines |
| Novel | chapter | characters, settings, objects | arcs, mysteries |
| Research / codebase | paper, PR, meeting | concepts, modules, people | open problems, migrations |

## Core ideas

### 1. Three clocks

| Clock | Meaning | Used for |
|---|---|---|
| **Observed-at** | when the observation was made: the session date, post time or chapter number | **the canonical order.** "Temporal order" always means this |
| **Ingested-at** | when the system processed it | audit only. Never used for ordering |
| **Referent time** | when the described events happened, if different | content. Notes say "learned at T about events at T₀" |

Referent time is not a sort key. Take a Session 15 vision that shows events from a century ago: it is *learned* in Session 15, and the notes must say so. Flashbacks in a novel and retrospective news reporting work the same way.

### 2. Observation vs. interpretation

Every derived document keeps what was observed visibly separate from what is inferred from it. That is the precondition for revising interpretations safely. If the two get blended, you can no longer tell which part to rewrite.

### 3. Narratives are entities

Plots, storylines and incidents get their own notes, with status and priority, just like people do. The abstract nouns are often what readers actually want.

### 4. Two layers of processing

| Layer | Input → output | Depends on order? | Cached? |
|---|---|---|---|
| **Extraction** | one ledger entry + the alias registry → atomic **claims** about entities | no | yes, one file per entry |
| **Synthesis** | all claims about an entity, sorted by observed-at, `as_of` T → that entity's note | **yes** | no, regenerated when dirty |

Almost all of the expensive reading happens in extraction, and extraction is order-independent. Synthesis is the only order-dependent step, and it reads compact claims rather than raw chunks. This split is what makes late arrival cheap: no full replay is needed.

### 5. Append history, rewrite the summary

Inside a note, the **timeline** (one entry per appearance, sorted by observed-at) is mechanically derived from the claims. The **status, overview, analysis and open questions** are the mutable interpretation, and they get rewritten on synthesis.

### 6. Point of view and `as_of`

Every synthesized document is written from a defined viewpoint (a player character, a reader at chapter N, the team at date D) and **as of** a time. Synthesis at `as_of` T may only use claims with observed-at ≤ T. Hindsight must never leak backward. This rule is also what makes "as if ingested in order" well-defined.

## Structure

```
ledger/                        # immutable. One file per observation chunk
  2026-08-13--session-13.md    #   frontmatter: observed_at, ingested_at, source, confidence
  2026-10-20--annotation-07.md #   human corrections are ledger entries too
claims/                        # Layer 1 cache. One file per ledger entry
  2026-08-13--session-13.jsonl
aliases.md                     # canonical name <- variants. Derived from claims and annotations
notes/                         # Layer 2. Generated
  People/  Places/  Organizations/  Items/  Plots/
chunks/                        # per-chunk commentary, generated as_of that chunk
  Session 13.md
briefing.md                    # "read this first": the current state, compressed
```

### Claim schema

One JSON object per line in `claims/<entry>.jsonl`:

```json
{"entry": "2026-08-13--session-13", "observed_at": "2026-08-13", "seq": 14,
 "entity": "Samus", "kind": "event|attribute|relation|status|alias|question",
 "text": "Offered to free Martavis in exchange for the helms of three lieutenants",
 "related": ["General Martavis", "Farraday", "Celine", "Draymon"],
 "referent_time": null, "confidence": "observed|reported|inferred", "quote": "Bring me their helms."}
```

`seq` preserves order within a chunk. `confidence` keeps "the GM said" separate from "a player guessed".

### Note frontmatter

```yaml
type: person | place | organization | item | plot
status: one line, current as of `as_of`
as_of: 2026-09-10--session-16
derived_from: [2026-03-12--session-05, ..., 2026-09-10--session-16]
aliases: [Almondrill]
```

## Participants

| Role | Reads | Writes | Notes |
|---|---|---|---|
| **Ingester** | raw input | `ledger/` | normalizes format, assigns observed-at, never interprets |
| **Extractor** | one ledger entry, `aliases.md`, the list of entity names | `claims/<entry>.jsonl`, new-entity and new-alias proposals | runs once per entry and can run in parallel |
| **Planner** | new claims, existing notes' `derived_from` and links | a **dirty set** | the cheap, deterministic part |
| **Synthesizer** | one entity's sorted claims ≤ `as_of`, its current note | that note | one per dirty entity, in parallel. Minimal-edit against the current note |
| **Chronicler** | one ledger entry, the notes as of that chunk | `chunks/<chunk>.md` | commentary: callbacks, flags, open questions |
| **Briefer** | the latest notes and open questions | `briefing.md` | the compressed warm start for humans and the next agent run |

In a small deployment, one agent plays every role in sequence. The roles still matter, because each one has a different contract.

## Process

### In-order ingest (the common case)

1. **Ingest** the chunk into `ledger/`.
2. **Extract** claims. Propose new entities and aliases. Weak evidence produces a *proposal*, not a note.
3. **Plan** the dirty set: entities with new claims, plus one hop of linked entities whose interpretation may change (see [Reinterpretation](#reinterpretation)).
4. **Synthesize** each dirty note as of the new chunk.
5. **Chronicle** the chunk's commentary.
6. **Brief**: regenerate `briefing.md`.
7. **Commit**, one commit per ingest. The diff is the audit log of what the system changed its mind about.

### Late arrival

The chunk's observed-at T is earlier than the latest already ingested.

1. **Ingest** at its observed-at position. Nothing already in the ledger moves.
2. **Extract** its claims. If it introduces a new alias, re-run alias resolution (cheap and mechanical) over existing claims.
3. **Plan.** The dirty set is every entity with a claim from the new entry, plus linked entities, and all their notes, because notes are always synthesized as of *now* and now includes T.
4. **Synthesize** the dirty notes. The synthesizer sees claims sorted by observed-at, so the late information lands in its correct place in the timeline and the interpretation.
5. **Chronicle cascade**, if [decision 1](#open-decisions) is strict: regenerate the commentary for every chunk after T that touches a dirty entity, each as of its own chunk.
6. **Brief** and **commit**, with the late insert called out in the commit message.

### Reinterpretation

Agents under-revise, so the synthesizer prompt makes revision an explicit step:

> Given the new claims, list which statements in the current note are now **contradicted**, **strengthened** or **reframed**. Rewrite those statements. Leave the rest alone.

Linked entities join the dirty set because reframings spread: "the ally is compromised" changes the notes of everyone the ally vouched for.

## Invariants

1. Ledger entries are immutable. Corrections are new entries (annotations) that reference the entry they correct.
2. Every note and chunk commentary can be regenerated from `ledger/` alone.
3. Nothing synthesized as of T uses a claim with observed-at > T.
4. Order of ingestion does not change the result, up to **claim-equivalence**: the same claims, timeline, status lines and open-question set. Prose may differ.
5. Every non-trivial statement in a note traces to at least one claim.

## Verifying it

The **shuffle test**: ingest a set of chunks in order, then again in a random order with some held back and inserted late. Compare the claim sets, timelines, `status` lines and open questions. They should match. Prose differences are expected.

## Consequences

**Benefits**
- Late, backfilled and corrected data become routine instead of corrupting state.
- "What did we know as of T" comes free.
- Interpretation can be revised without losing provenance.
- Extraction and synthesis both parallelize per entry and per entity.

**Costs**
- Notes are no longer hand-editable. Humans correct through annotations.
- LLM synthesis is nondeterministic, so regeneration produces prose churn. Minimal-edit prompting against the current note keeps diffs readable.
- A strict chronicle cascade can rewrite a lot of commentary for one old chunk.
- The claim layer is extra storage and an extra schema to maintain.

## Failure modes

| Failure | Symptom | Guard |
|---|---|---|
| Entity resolution drift | a phantom entity is created from a misheard name, or two entities are merged | proposals before creation; a human-reviewable `aliases.md`; annotations to split or merge |
| Hindsight leak | early-chunk commentary "knows" a later reveal | enforce `as_of` filtering in code, not just in the prompt |
| Under-revision | stale conclusions survive contradicting evidence | the explicit reinterpretation step; one-hop dirty propagation |
| Over-revision | stable prose rewritten on every run | minimal-edit synthesis from the current note |
| Silent misordering | an undated chunk is guessed into the wrong place | require observed-at; record `confidence` on it; never guess silently |
| Noisy chunk | chaotic input (combat, flame wars) yields confident nonsense | `confidence` per claim; the chunk records its own reliability |

## Open decisions

These are unresolved in the draft. Each instance should decide them in its domain profile.

1. **Strict or frozen chunk commentary.** When a late chunk at T arrives, does commentary for chunks after T get regenerated, so it reads as if written knowing T, or does it stay frozen as a historical artifact? Strict is purer and costs a cascade. Frozen is cheaper, and entity notes are still correct. *Leaning: strict, bounded to chunks that touch dirty entities.*
2. **Human edits to derived notes.** If humans never edit notes, the system stays simple: all corrections go through annotations. If they do, notes need protected regions that synthesis must preserve verbatim. *Leaning: annotations only.*

## Related patterns

- **Event sourcing.** The ledger is the event log, and notes are projections.
- **Bitemporal data.** Observed-at is valid time and ingested-at is transaction time.
- **Materialized views with incremental maintenance.** The dirty set is view maintenance.
- **Zettelkasten / literature → permanent notes.** The same two-layer split, done by hand.

## Known uses

- [Cael Stormcrow campaign vault](reference-implementation.md): D&D 5e session notes into an Obsidian entity graph. In-order only today.
- [youtube-synthesizer / article-synthesizer](https://github.com/devonjones/devon-claude-skills/blob/main/docs/youtube-wiki-design.md) (devon-claude-skills): implements the **ledger and extraction half**. `sources/<type>/<ingested-date>-<title>/` is an immutable, source-faithful ledger. The frontmatter already separates the clocks: `source_published_date` is observed-at and `ingested_date` is ingested-at. "Source-faithful, do not interpret" is the extractor contract. The vault root, hands-off for skills, is the synthesis layer, which a human does today. Its planned Phase 4 cross-source linker and MOC maintenance are the first steps toward automated synthesis. Note that it sorts by **ingested** date, which is right for a reading log and wrong for this pattern's ordering.
