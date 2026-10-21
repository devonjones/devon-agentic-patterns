# Domain profiles

Starting `PROFILE.md` sketches for the domains this pattern targets. An agent applying the [playbook](playbook.md) should begin from the closest one, then confirm or adjust it with the user.

Each profile names what's distinctive about its domain. Anything not mentioned follows the defaults in the playbook.

---

## YouTube series

- **Chunk:** one video (transcript, kept frames, description). The [youtube-synthesizer](https://github.com/devonjones/devon-claude-skills) literature note *is* the ledger entry
- **Knowledge axis:** video publish date. Tiebreak: playlist position
- **Narrative axis:** same as the knowledge axis, unless the series covers a subject with its own chronology (a history channel), in which case it's the subject's dates
- **Viewpoint:** a viewer who has watched up to `as_of`
- **Entity types:** People (presenters, guests, subjects), Concepts, Tools/Products, Works (books, papers or videos cited), Series. **Narratives:** recurring arguments, project builds that span episodes, positions that evolve
- **Alias noise:** auto-caption mishearings, presenter shorthand
- **Authority:** the presenter's own statements > the video description > auto-captions. Corrections pinned in comments count as source corrections
- **Late arrival:** backfilling the back catalog after following the channel for a while. Expect the presenter's *earlier* positions to land late and reveal how their views evolved
- **Commentary period:** per video

## RPG setting from sourcebooks

- **Chunk:** one chapter or section of a sourcebook
- **Knowledge axis:** book publication date, then page order. Edition is recorded as a source attribute
- **Narrative axis:** the in-world calendar (e.g. Dale Reckoning). Often fuzzy ("during the Time of Troubles")
- **Viewpoint:** omniscient canon as of `as_of`. Optionally a separate player-knowledge view
- **Entity types:** People, Places, Organizations, Deities, Items, Creatures. **Narratives:** historical events, eras, ongoing conflicts, metaplot
- **Alias noise:** spelling changes between editions, titles vs. names, in-world vs. out-of-world names
- **Authority:** **this domain's defining problem.** The precedence rule must be explicit: "newer edition supersedes older for the same fact" or "keep each edition's version, tagged". A retcon is a contradiction resolved by profile, never silently
- **Late arrival:** reading an older or regional sourcebook after the core books. Reframes are common (a lore book redefining a deity)
- **Commentary period:** per book rather than per chapter

## Local news via RSS

- **Chunk:** one article, ingested in batches (hourly or daily)
- **Knowledge axis:** publication timestamp. Updates and corrections are new entries with their own timestamps
- **Narrative axis:** event time stated in the article, which can differ for retrospectives or court cases about old events
- **Viewpoint:** a well-informed resident as of `as_of`
- **Entity types:** People (officials, residents), Organizations (council, businesses, agencies), Places (neighborhoods, sites), Issues. **Narratives:** stories (a zoning fight, a trial, a development)
- **Alias noise:** full names vs. surnames vs. titles; agency acronyms
- **Authority:** primary documents and on-record quotes > reporting > opinion pieces. A correction supersedes the claim it corrects. **Multiple outlets on one event are corroboration, not separate events**: the extractor proposes same-event links
- **Late arrival:** archive backfill when a new feed is added; corrections days later
- **Volume:** high. Commentary is per **day or week**, not per article. Synthesis is batched per period
- **Commentary period:** daily digest

## Reddit community

- **Chunk:** a thread snapshot (post plus comments at fetch time), plus **comment batches** for threads fetched again later
- **Knowledge axis:** post or comment timestamp
- **Narrative axis:** same as the knowledge axis
- **Viewpoint:** a long-time lurker as of `as_of`
- **Entity types:** Users (regulars, mods), Topics, Projects/Products, Memes and in-jokes, external entities the sub discusses. **Narratives:** controversies, rule changes, recurring debates, community events
- **Alias noise:** username changes, nicknames, abbreviations specific to the sub
- **Authority:** mod announcements > high-consensus comments > individual comments. **Claims are attributed** ("u/x asserts…"). The community's beliefs are an entity attribute, not facts about the world
- **Mutable sources:** edits and deletions are new ledger entries (`retraction` claims for deletions). Never overwrite the original snapshot
- **Late arrival:** comments on old threads; backfilling the sub's history
- **Volume:** high. Filter by score or engagement at ingest. Commentary is per week
- **Commentary period:** weekly

## Book-series wiki

- **Chunk:** one chapter
- **Knowledge axis:** **choose explicitly.** Publication order (the default: a reader's experience) or a chronological reading order. This decides whether a prequel's chapter commentary comes first or last
- **Narrative axis:** in-universe chronology. A prequel's events go before book 1's in every timeline
- **Viewpoint:** a reader who has read up to `as_of`. **The final wiki is as of everything.** Spoiler-safe wikis are `as_of` a book boundary
- **Entity types:** Characters, Places, Factions, Objects, Concepts (magic systems). **Narratives:** arcs, mysteries, prophecies, relationships
- **Alias noise:** epithets, titles, secret identities (**a reveal that two entities are one is an alias merge plus a reframe**)
- **Authority:** the text itself; flagged unreliable narrators; author statements outside the text, if the profile allows them
- **Late arrival:** prequels, novellas and side stories. **The canonical reframe case.** Expect full re-synthesis
- **Commentary period:** per chapter, with a per-book roll-up

## Tabletop campaign

- **Chunk:** one session's notes
- **Knowledge axis:** session date, then session number
- **Narrative axis:** session date, except for in-world revelations (visions, lore) that describe the distant past
- **Viewpoint:** one player character's knowledge. GM secrets are excluded
- **Entity types:** People, Places, Organizations, Items. **Narratives:** Plots
- **Alias noise:** phonetic note-taking ("Almondrill" → Ahmendril)
- **Authority:** what the GM narrated > what an NPC claimed > what a player speculated
- **Late arrival:** a missing session's notes found later; another player's notes on an old session
- **Commentary period:** per session, plus a read-aloud briefing for the next one
- See [reference-implementation.md](reference-implementation.md)
