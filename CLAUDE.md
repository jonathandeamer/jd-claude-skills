# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This repo (`jd-claude-skills`) is **Jonathan Deamer's personal Claude Code plugin marketplace** — a catalog of skills, not a Python application. There is no application code to build or run here. Each deliverable is a *skill*: a Markdown instruction file that teaches Claude to do something. Editing this repo means editing prose (instructions Claude follows) and the manifests that package it, not application logic. The marketplace currently ships one skill (`bootstrap-python-repo`) but is structured to hold many.

## Layout

One marketplace catalog at the root, then one self-contained plugin per skill:

- `.claude-plugin/marketplace.json` — the marketplace catalog. Its `plugins` array lists every skill-plugin and points `source` at each plugin directory. Adding a skill = adding one entry here.
- `plugins/<skill>/.claude-plugin/plugin.json` — the plugin manifest for one skill (name, description, author, and — when published — `version`).
- `plugins/<skill>/skills/<skill>/SKILL.md` — **the actual product.** Everything else is packaging around these files.
- `evals/<skill>/evals.json` — that skill's regression suite: prompts plus objective assertions exercising it end-to-end. Lives at the repo root, outside the plugin source, so it never ships to users. When you change a `SKILL.md`, re-check its evals; when you add a behavior, add an eval.

Concretely, the `bootstrap-python-repo` skill lives at `plugins/bootstrap-python-repo/skills/bootstrap-python-repo/SKILL.md`, with evals at `evals/bootstrap-python-repo/evals.json`.

## Adding a new skill

Each skill is an independent plugin, so adding one is purely additive — it never touches existing skills:

1. Create `plugins/<skill>/skills/<skill>/SKILL.md` (the skill itself).
2. Create `plugins/<skill>/.claude-plugin/plugin.json` (`name`, `description`, `author`; omit `version` to keep commit-SHA auto-updates).
3. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`: `name`, `source: "./plugins/<skill>"`, and `description`.
4. Add `evals/<skill>/evals.json` as that skill's regression suite.
5. Validate: `claude plugin validate ./plugins/<skill>` and `claude plugin validate .`.

Users then install it with `/plugin install <skill>@jd-claude-skills`. Build the skill the rigorous way — brainstorm, draft, then eval — via the skill-creator workflow.

## The name/description appears in three places — keep them in sync

For each skill, the plugin name and one-line description are duplicated across that skill's entry in `marketplace.json`, its `plugin.json`, and its `SKILL.md` frontmatter. Change one, change all three (for that skill).

The `description:` in `SKILL.md`'s YAML frontmatter is **not cosmetic** — it is the trigger phrase Claude matches against to decide whether to activate the skill. Editing it changes *when the skill fires*, so treat wording there as behavior, not documentation.

## Versioning (required for a distributed marketplace)

Right now no `version` field is set anywhere, so Claude Code falls back to the **git commit SHA** as the version — every commit is treated as a new release, and installed users update on each push. That is fine for active development.

For a stable, distributed plugin, set an explicit semantic `version` in `plugin.json` (it takes precedence over any version in the marketplace entry). **Once you adopt explicit versioning, you must bump `version` on every change to `SKILL.md` (or any plugin file) — otherwise `/plugin update` reports "already at the latest version" and users never receive your change.** This is the single most important habit for maintaining a published skill: a content change without a version bump is invisible to installed users.

Validate manifests before publishing:

```bash
claude plugin validate ./plugins/bootstrap-python-repo --strict
```

Note: `claude plugin validate` reports unrecognized/misspelled fields as warnings, not errors — read its output, don't just check the exit code.

## Editing the bootstrap-python-repo skill

This guidance is specific to the `bootstrap-python-repo` skill. Two things make its `SKILL.md` correct:

- The bootstrap steps it describes must actually produce a working project. The authoritative check is that a scaffolded project passes `uv run ruff check .` and `uv run pytest` — this is asserted in every eval. If you change the generated `pyproject.toml`, `.gitignore`, or git hooks, mentally (or actually) run a scaffold through to confirm it still passes.
- The skill embeds a *template* `CLAUDE.md` (for the projects it generates). Don't confuse it with this file: this CLAUDE.md governs *this* repo; the one inside `SKILL.md` is emitted into the user's new project. Conventions baked into that template (conventional commits, no AI attribution, type hints required, no `print()` in production) are deliberate — preserve them unless changing them on purpose.

## Commits

This repo follows the same conventions it teaches: **conventional commits** (`feat`, `fix`, `docs`, `chore`, `refactor`, etc.) and **no AI attribution** — no `Co-Authored-By` trailers, commit under the repository's own git identity.
