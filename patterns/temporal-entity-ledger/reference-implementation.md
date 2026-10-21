# Reference implementation: Cael Stormcrow campaign vault

A D&D 5e campaign tracked in an Obsidian vault from one player's point of view. Another player takes notes at the table. After each session, an agent turns them into a session write-up and propagates entities into a note graph. This is where the pattern came from. The vault predates the formal pattern and implements the **in-order** half of it.

## Mapping

| Pattern concept | In the vault |
|---|---|
| Chunk | one session |
| Observed-at | session date (`date:` in frontmatter) and session number |
| Ledger entry | **merged into** `Sessions/Session NN - <Title>.md`: raw notes appear as bold lines inside the write-up |
| Chunk commentary | the same session file: blockquote analysis, *Key Discoveries*, *Open Questions*, *Active Plot Threads* |
| Notes | `Notes/{People,Places,Organizations,Items,Plots}/*.md` |
| Status line | `status:` frontmatter (plus `priority:` on plots) |
| Timeline | a *Session Appearances* section with one `### [[Session NN]]` per session |
| Aliases | a per-session "naming notes" block ("Almondrill" → Ahmendril, "Iron Fist" → Order of the Fist). No central registry |
| Point of view | the player character's knowledge. GM-only backstory is kept in a separate file and never leaks into the notes |
| Briefing | `Sessions/Session NN+1 - Read This First.md`: a one-minute read-aloud recap in the party's voice. It gets renamed into the next session's file, and its content becomes that file's *Previously* section |
| Commit | one commit per batch of sessions. The message has a paragraph per session plus the list of new notes |
| Narrative entities | `Notes/Plots/`, e.g. a devil's debt or a stolen keystone, with status and priority |
| Annotation | done ad hoc as a commit, e.g. *"Remove fictional General Marcus -- note-taking error for the unnamed Wolf captain"* |

## What it does well

- **Observation vs. interpretation** is visually consistent: bold for what happened, blockquotes for analysis, ⚠️ callouts for things the party missed.
- **Reinterpretation happens.** Later sessions rewrite earlier conclusions in the notes. For example, once an ally was shown to be magically compromised, the earlier decisions made on that ally's advice were relabeled as likely mistakes.
- **Narratives are first-class entities.**
- **The briefing works as a warm start** for the humans at the table and for the agent's next run.

## Where it falls short of the pattern

| Gap | Consequence |
|---|---|
| The ledger is merged into the commentary | the raw notes can't be cleanly separated out to replay them |
| No claim layer | each ingest rereads sessions and notes. Late data would need a manual rewrite |
| Notes are edited state, not derived | a late session can't be slotted in. Hand edits and agent edits are indistinguishable |
| No `as_of` | nothing prevents a later reveal from leaking into an earlier session's commentary |
| Aliases live per session | the same misspelling gets re-resolved every time. A phantom entity (General Marcus) once got created |
| Annotations are commits, not ledger entries | a replay would resurrect the corrected errors |

## Migration sketch

1. Pull the bold raw-note lines out of each session file into `ledger/<date>--session-NN.md`. Where the original notes exist, use those instead.
2. Build `aliases.md` from every session's naming-notes block, plus annotations for past corrections.
3. Extract claims per session, synthesize every note, and diff against the current `Notes/`. Review the diff: anything in the old notes with no claim behind it is either missing evidence (write an annotation) or invention (drop it).
4. Regenerate the session commentary as of each session, and compare it with the originals.
