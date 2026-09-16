# hexcore-skill — moved

This skill now lives inside the HexCore monorepo:

**https://github.com/Indroic/HexCore/tree/master/skills/hexcore**

This repository is archived and no longer updated.

## Why it moved

The skill's whole premise is that it does not remember HexCore's API, it reads it — because the
hand-written import table its first version carried went stale across seven majors and was
still teaching `SQLAlchemyCommonImplementationsRepo`, deleted in 7.0.

Living in a separate repository reproduced that failure one level up: nothing contrasted the
skill's *prose* against the framework it documents. Inside the monorepo it is covered by the
same CI gate as the official documentation — `packages/hexcore/tests/test_documentation_examples.py`
resolves every `from hexcore… import …` it teaches against the real package, and every
`import { … } from "@hexcore-js/darwin-client"` against the real TypeScript client. A rename in
either package now turns the build red instead of quietly teaching an import that does not
exist.

It also gained coverage of `@hexcore-js/darwin-client`, Darwin's official TypeScript client,
which did not exist when this repository was written.

## Installing it

```bash
npx skills add Indroic/HexCore -s hexcore        # -> .claude/skills/hexcore
npx skills add Indroic/HexCore -s hexcore -g     # -> ~/.claude/skills/hexcore
```

`-s hexcore` matters: HexCore is a monorepo and the CLI walks `skills/` looking for every
`SKILL.md`. If you pass `-a`, the agent id is `claude-code`.

By hand, if you would rather not use the CLI:

```bash
git clone --depth 1 https://github.com/Indroic/HexCore /tmp/hexcore
cp -r /tmp/hexcore/skills/hexcore .claude/skills/hexcore
```

Working inside the monorepo itself, the repository root is a Claude Code plugin: run
`/plugin marketplace add .` once and the skill loads on its own, as `/hexcore:hexcore`.

## Licence

MIT, as before. See [LICENSE](./LICENSE).
