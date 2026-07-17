# Contributing

Thanks for your interest in improving Playwright Guardrails. Most contributions are
changes to the **canonical rules** — the heart of the plugin.

## The one thing to know

The framework rules and the review checklist live in exactly one place —
[`skills/rules/`](skills/rules/) — and are the single source the probe, writer, and
reviewer all consume. There is no second copy to keep in sync. See
[`DESIGN.md`](DESIGN.md) for why the architecture is shaped this way.

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

## Changing an agent

The three agents live in [`agents/`](agents/). Each is a single-role sub-agent that
loads the canonical rules itself via its frontmatter `skills:` field — don't pass rule
paths in as inputs, and keep each agent to its one role (the probe never generates,
the writer never verdicts, the reviewer never edits). Per-role reasoning `effort` is
pinned in frontmatter for a reason ([`DESIGN.md`](DESIGN.md)); change it only with a
clear rationale.

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
