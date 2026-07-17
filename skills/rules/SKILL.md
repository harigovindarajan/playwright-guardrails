---
name: rules
description: Manifest of the canonical Playwright review sources. Names the two authoritative files — the framework rules and the review checklist — that the reviewer (and future writer) agent must read in full before reviewing or generating any Playwright test. Preload into those agents via their skills: field; not for direct human use.
user-invocable: false
---

# Playwright Guardrails — Rules Manifest

This skill is a **manifest**, not the rules themselves. It names the canonical
Playwright review sources and mandates reading them. It carries no review logic —
the rules live in the files it points to.

## Authoritative review sources

Two files in **this skill's directory** are the ONLY authoritative sources for any
Playwright review or test generation:

1. `playwright-framework-rules.md` — the normative framework rules (locators, Page
   Object Model, `test.step`, assertion placement, reliability, data setup, scaling).
2. `playwright-framework-review-checklist.md` — the adversarial review checklist that
   maps to those rules.

Both paths are relative to this skill's base directory, which is injected into your
context when this skill is preloaded via a sub-agent's `skills:` field.

## What you MUST do

- **You MUST read both files in full**, from this skill's directory, **before**
  performing any review or generating any test. They are the only authoritative
  review sources — reviewer preferences beyond what these files state are not grounds
  for a finding.
- **If either file is missing, empty, or cannot be read, STOP and return `FAIL`**
  with a blocking finding that names the missing source. Do not proceed with a
  partial, remembered, or assumed ruleset.

## Why

A Playwright review is only valid against the canonical rules. A review that runs
without them silently passes violations — the exact failure mode this plugin exists
to prevent. Reading the files from the injected base directory (rather than guessing
paths or relying on training memory) is what makes the rules load reliably on every
run, regardless of where the agent is spawned.
