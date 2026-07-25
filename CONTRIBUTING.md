# Contributing

Thanks for your interest in improving Playwright Guardrails. Most contributions are
changes to the **canonical rules** — the heart of the plugin.

For bug reports or larger changes (a new rule category, an agent behavior change),
please [open an issue](https://github.com/harigovindarajan/playwright-guardrails/issues)
first so the approach can be agreed before you invest in a PR. By contributing, you
agree that your contributions are licensed under the [MIT license](LICENSE).

## The one thing to know

Anything two or more agents share lives in [`skills/`](skills/) — never inside one of the
agents. There is no second copy to keep in sync.

- [`skills/rules/`](skills/rules/) — the framework rules and the review checklist. The
  probe, writer, and reviewer all consume them.
- [`skills/locator-map/`](skills/locator-map/) — the shape of the locator map. The probe
  writes maps in it, the writer reads them against it. The reviewer does **not** preload
  it and must not: it stays blind to the map by design.

Agents pick these up through their frontmatter `skills:` field, which injects the skill's
content at startup — so a shared source is never passed in as a path and never copied into
an agent. See [`DESIGN.md`](DESIGN.md) for why the architecture is shaped this way.

## Extending the rules

- **`skills/rules/playwright-framework-rules.md`** — the canonical rules. Each rule
  is numbered; the number is its stable identity. Add new rules at the end rather
  than renumbering existing ones, so checklist references and any downstream tooling
  stay valid.
- **`skills/rules/playwright-framework-review-checklist.md`** — the review checklist.
  It **references rule numbers** rather than restating the rules. When you add or
  change a rule, update the checklist to point at it — don't duplicate the rule text.
- **`skills/rules/SKILL.md`** — the manifest the agents preload. Update it only if you
  add or rename a canonical file.

Keep the rules thin and expressive: a rule should read as guidance a reviewer can
apply, and the framework it produces should read as business journeys.

To smoke-test a rule change, dispatch `playwright-test-reviewer` against a spec you
know is compliant (with its contract) and confirm the change doesn't introduce
spurious findings — and, for a new rule, that a spec violating it now produces the
expected finding.

## Changing an agent

The three agents live in [`agents/`](agents/). Each is a single-role sub-agent that
loads the canonical rules itself via its frontmatter `skills:` field — don't pass rule
paths in as inputs, and keep each agent to its one role (the probe never generates,
the writer never verdicts, the reviewer never edits). Per-role reasoning `effort` is
pinned in frontmatter for a reason ([`DESIGN.md`](DESIGN.md)); change it only with a
clear rationale.

Don't restate a shared source inside an agent — not field names, not enum values, not rule
text. An agent describes how it *produces* or *consumes* the artifact; the skill describes
what the artifact *is*. When you change the locator map's shape, change
[`skills/locator-map/SKILL.md`](skills/locator-map/SKILL.md) and check that both the probe
and the writer still hold up against it.

## Before you open a PR

1. **Validate the plugin:**
   ```bash
   claude plugin validate .
   ```
2. **Smoke-load it** and confirm the agents register:
   ```bash
   claude --plugin-dir .
   ```
   Check that the three agents appear in `/context` under Custom Agents. The
   `rules` skill is preload-only (`user-invocable: false`) — the agents load it
   themselves, so it won't show in the slash-command menu.
3. Use clear, conventional commit messages (`feat:`, `fix:`, `docs:`, `chore:`) and
   open the PR against `main`.

## Versioning

The plugin follows [Semantic Versioning](https://semver.org/). Bump `version` in
[`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) and add a matching
[`CHANGELOG.md`](CHANGELOG.md) entry when your change is user-facing.
