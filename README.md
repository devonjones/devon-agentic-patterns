# Devon's Agentic Patterns

Design patterns for work done by LLM agents: shapes that keep recurring across projects, written down once so the next agent (or human) can start from the pattern instead of rediscovering it.

A pattern belongs here once the same shape has shown up in more than one real project. Each one lives in its own folder under `patterns/`.

## Patterns

| Pattern | One line | Status |
|---|---|---|
| [Temporal Entity Ledger](patterns/temporal-entity-ledger/) | Turn a time-ordered stream of messy observations into a living graph of entity and narrative notes, and stay correct when observations arrive late. | Draft (decisions settled) |

## Anatomy of a pattern folder

```
patterns/<name>/
  README.md                    # the pattern: intent, structure, process, trade-offs, open decisions
  playbook.md                  # how an agent applies it to a new domain, step by step
  reference-implementation.md  # a real project that uses it, and where it falls short of the pattern
  domain-profiles.md           # optional: starting configurations for the domains it targets
```

`README.md` is for understanding the pattern. `playbook.md` is what you hand to an agent that has to build or run an instance of it.

## Using a pattern with an agent

Point the agent at the folder:

> Read `patterns/temporal-entity-ledger/` in devon-agentic-patterns, then apply the playbook to `<my project>`.

The playbook begins with a **domain profile**: a short list of questions the agent answers (or asks you) before it writes anything.
