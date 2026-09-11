# hexcore-skill

An agent skill for [HexCore](https://pypi.org/project/hexcore/) **9.x** — the Python framework
for hexagonal architecture, DDD, CQRS, background workers, event sourcing and identity.

It does two things a documentation dump cannot.

**It reads the installed package instead of remembering it.** The previous version of this
skill carried a hand-written table of import paths, and that table went stale across seven
majors — it was still teaching `SQLAlchemyCommonImplementationsRepo`, deleted in 7.0. There is
no import registry in this repository any more. `scripts/hexcore_surface.py` parses the
facades' `_EXPORTS` dicts with `ast` and answers from whatever version is actually installed.

**It audits code, not just describes it.** HexCore's documentation marks around twenty traps
with a warning, and they share a property: almost none of them raise.
`scripts/hexcore_audit.py` finds them statically, along with every name removed in 7.0 or
deprecated in 9.0.

---

## Install

Copy this directory into your project's skills folder:

```bash
git clone https://github.com/Indroic/hexcore-skill .claude/skills/hexcore
```

The scripts need only the standard library, and they do **not** import `hexcore` — so they
work on a bare install with zero extras. Run them with the interpreter of the environment
HexCore is installed in.

---

## The tools, standalone

They are useful outside an agent session.

```bash
# What is installed, and whether this skill's prose covers it
python scripts/hexcore_surface.py --version

# Where a symbol lives: facade, canonical long path, required extra
python scripts/hexcore_surface.py --find SqlAlchemyRepository

# Everything one facade exports
python scripts/hexcore_surface.py --facade cqrs

# What dies in the next major, read from the package
python scripts/hexcore_surface.py --deprecated

# Verify every `from hexcore… import …` in a file actually resolves
python scripts/hexcore_surface.py --check src/ docs/

# The whole import registry, as markdown
python scripts/hexcore_surface.py --registry
```

```bash
# Audit a project: removed API, plus the silent failure modes
python scripts/hexcore_audit.py src/

# In CI
python scripts/hexcore_audit.py src/ --fail-on high
python scripts/hexcore_audit.py src/ --json | jq '.[] | select(.severity=="critical")'
```

`--check` and `--audit` are worth wiring into a project's own CI. The single most valuable
finding is `alembic-framework-models`: an `env.py` missing
`ensure_framework_models_loaded()` produces a migration that generates cleanly and drops a
table with data in it when applied.

---

## What is in here

| Path | What it is |
| :-- | :-- |
| `SKILL.md` | The router: version gate, the five facades, the non-negotiables, where to go next |
| `scripts/hexcore_surface.py` | Reads the real API surface out of the installed package |
| `scripts/hexcore_audit.py` | Finds removed API and the framework's silent failure modes |
| `references/core.md` | Entities, events, repositories, queries, UoW, SQL, Alembic, config |
| `references/fastapi.md` | `create_app`, lifespan, health, rate limiting, streaming, routers |
| `references/cqrs-workers-cron.md` | Buses, middleware, Smart Routing, queues, workers, cron |
| `references/event-sourcing.md` | Event store, aggregates, projections, outbox — and its limits |
| `references/darwin.md` | Identity: config, routes, storage, plugins |
| `references/testing.md` | Doubles, fixtures, overrides |
| `references/failure-modes.md` | Indexed by symptom, because the causes are invisible |
| `references/review-rubric.md` | How to review, in the order that finds expensive things first |
| `references/workflows.md` | Ordered recipes per task |
| `references/removed-api.md` | 7.0 removals, 9.0 deprecations, and what changed silently |
| `assets/templates/` | Correct-by-construction starting files |

---

## Keeping it current

When HexCore releases a major, the factual half updates itself — the scripts read the new
package. What needs a human pass is the prose.

1. `python scripts/hexcore_surface.py --version` will say the skill's documented major no
   longer matches. Bump `DOCUMENTED_MAJOR` in `scripts/hexcore_surface.py`.
2. `python scripts/hexcore_surface.py --check SKILL.md references/ assets/templates/` — every
   symbol that was renamed or removed shows up with its file and line.
3. `python scripts/hexcore_surface.py --deprecated` gives the new deprecation table for
   `references/removed-api.md`.
4. Add the new major's silent behaviour changes to `references/removed-api.md` and
   `references/failure-modes.md`. Those are the part no tool can derive: they are semantics,
   not names.

Step 2 is the gate. It is the same mechanism HexCore runs over its own documentation in
`tests/test_documentation_examples.py`, and it is the difference between believing an import
exists and knowing it does.

---

## Licence

MIT. See [LICENSE](./LICENSE).
