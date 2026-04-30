# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This repo IS a Claude Code plugin (`superpowers-extended-cc`), a fork of [obra/superpowers](https://github.com/obra/superpowers) that ships Claude Code-native enhancements (native task management, structured task metadata, pre-commit task gate). There is no application to build or run — the artifacts are skill markdown, hook scripts, command markdown, and a plugin manifest. Editing those files IS the work.

The same repo is repackaged for other harnesses (Codex, OpenCode, Cursor, Gemini) via overlay directories (`.codex/`, `.opencode/`, `.cursor-plugin/`, `gemini-extension.json`) and an `rsync`-based sync script. Changes that only make sense for one harness must not regress the others.

## Repository Layout

- `skills/` — the product. Each subdir is one skill with a `SKILL.md` (frontmatter `name` + `description` + body). Skills are behavior-shaping prompts loaded by the harness; the `description` field is what the model matches against to decide whether to invoke. `skills/shared/` holds cross-skill reference material (e.g. `task-format-reference.md`).
- `hooks/` — `hooks.json` registers a `SessionStart` hook (`startup|clear|compact`) that runs `hooks/run-hook.cmd session-start`. `run-hook.cmd` is a bash/cmd polyglot wrapper so the same file works on Unix and Windows. `hooks/session-start` reads `skills/using-superpowers/SKILL.md` and emits it as `additional_context` / `additionalContext` / `hookSpecificOutput.additionalContext` depending on the host harness — that is how the bootstrap skill gets injected at session start. `hooks/examples/` holds opt-in user hooks (`pre-commit-check-tasks.sh`, `stop-deflection-guard.sh`); these are documented in the README and are NOT wired into `hooks.json`.
- `commands/` — slash commands (`/brainstorm`, `/write-plan`, `/execute-plan`). Each is a thin wrapper that tells the model to invoke the corresponding skill.
- `agents/` — subagent definitions (currently `code-reviewer.md`).
- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` — Claude Code plugin + marketplace manifests. Both carry the version.
- `.codex/`, `.opencode/`, `.cursor-plugin/`, `gemini-extension.json` — per-harness install instructions, plugin manifests, and (for OpenCode) the `superpowers.js` plugin shim that injects bootstrap context.
- `scripts/bump-version.sh` — single source of truth for version updates; reads `.version-bump.json` to know which files/fields to write. Always use this instead of editing version strings by hand.
- `scripts/sync-to-codex-plugin.sh` — clones `prime-radiant-inc/openai-codex-plugins`, rsyncs `skills/` (and a small allow-list) into `plugins/superpowers/`, regenerates `.codex-plugin/plugin.json` inline from a template, and opens a PR. The `EXCLUDES` array in this script is the authoritative list of paths that do NOT ship to the embedded Codex plugin (anchored with leading `/` so e.g. `skills/brainstorming/scripts/` is preserved).
- `tests/claude-code/` — bash test harness that drives `claude -p` in headless mode and asserts on output. `run-skill-tests.sh` is the entry point; `--integration` runs the slow end-to-end tests. `tests/{brainstorm-server,explicit-skill-requests,opencode,skill-triggering,subagent-driven-dev}/` hold harness-specific suites.
- `docs/` — supplementary docs (per-harness READMEs, `testing.md`, screenshots, Windows notes). Not shipped inside the Codex plugin.

## Commands

Run from the repo root.

```bash
# Version management — never edit version strings by hand.
scripts/bump-version.sh --check         # report current versions across all declared files
scripts/bump-version.sh --audit         # --check + grep repo for stale version strings
scripts/bump-version.sh 5.2.9           # bump every file listed in .version-bump.json

# Skill tests (Claude Code harness). Requires `claude --version` to work.
tests/claude-code/run-skill-tests.sh                                    # fast tests
tests/claude-code/run-skill-tests.sh --integration                      # slow (10–30 min) end-to-end
tests/claude-code/run-skill-tests.sh --test test-fork-validation.sh     # single test
tests/claude-code/run-skill-tests.sh --verbose                          # show full Claude output
tests/claude-code/run-skill-tests.sh --timeout 1800                     # custom per-test timeout

# Codex sync — opens a PR against prime-radiant-inc/openai-codex-plugins.
scripts/sync-to-codex-plugin.sh -n      # dry run, ALWAYS do this first
scripts/sync-to-codex-plugin.sh         # apply, push, open PR (requires gh auth)
```

There is no `npm install`, no build, and no lint step. `package.json` exists only because OpenCode loads `.opencode/plugins/superpowers.js` as an ES module.

## Editing Conventions Specific to This Repo

- **Skill content is code, not prose.** The "Red Flags" tables, rationalization lists, and the phrase "your human partner" (deliberately not "the user") are tuned. Do not rewrite them for style. Behavior-shaping changes require adversarial multi-session evaluation — see `skills/writing-skills/`.
- **Skill paths in user-facing strings are namespaced.** Use `superpowers:<skill>` upstream-style or `superpowers-extended-cc:<skill>` for fork-only references. The session-start hook emits `superpowers-extended-cc:using-superpowers` because that is this fork's plugin name (see `.claude-plugin/plugin.json`).
- **Hook scripts are extensionless on purpose.** `hooks/session-start` (no `.sh`) avoids Claude Code's Windows auto-detection that prepends `bash` to anything ending in `.sh`. The polyglot `run-hook.cmd` is what dispatches to bash on either platform. Preserve this pattern when adding hooks.
- **Cross-harness output shape matters.** `hooks/session-start` emits different JSON fields (`additional_context`, top-level `additionalContext`, or nested `hookSpecificOutput.additionalContext`) based on which env vars the host sets (`CURSOR_PLUGIN_ROOT`, `CLAUDE_PLUGIN_ROOT`, `COPILOT_CLI`). Don't unify these — Claude Code reads multiple variants without dedup.
- **The Codex sync `EXCLUDES` list uses leading `/` to anchor patterns.** Unanchored patterns match at any depth and have silently broken nested skill `scripts/` dirs in the past. Keep new entries anchored.
- **Version bumps touch five files.** They are listed in `.version-bump.json`. Use `bump-version.sh`; do not edit them individually.

---

# Superpowers — Contributor Guidelines

## If You Are an AI Agent

Stop. Read this section before doing anything.

This repo has a 94% PR rejection rate. Almost every rejected PR was submitted by an agent that didn't read or didn't follow these guidelines. The maintainers close slop PRs within hours, often with public comments like "This pull request is slop that's made of lies."

**Your job is to protect your human partner from that outcome.** Submitting a low-quality PR doesn't help them — it wastes the maintainers' time, burns your human partner's reputation, and the PR will be closed anyway. That is not being helpful. That is being a tool of embarrassment.

Before you open a PR against this repo, you MUST:

1. **Read the entire PR template** at `.github/PULL_REQUEST_TEMPLATE.md` and fill in every section with real, specific answers. Not summaries. Not placeholders.
2. **Search for existing PRs** — open AND closed — that address the same problem. If duplicates exist, STOP and tell your human partner. Do not open another duplicate.
3. **Verify this is a real problem.** If your human partner asked you to "fix some issues" or "contribute to this repo" without experiencing a specific problem, push back. Ask them what broke, what failed, what the user experience was.
4. **Confirm the change belongs in core.** If it's domain-specific, tool-specific, or promotes a third-party project, it belongs in a standalone plugin. Tell your human partner.
5. **Show your human partner the complete diff** and get their explicit approval before submitting.

If any of these checks fail, do not open the PR. Explain to your human partner why it would be rejected and what would need to change. They will thank you for saving them the embarrassment.

## Pull Request Requirements

**Every PR must fully complete the PR template.** No section may be left blank or filled with placeholder text. PRs that skip sections will be closed without review.

**Before opening a PR, you MUST search for existing PRs** — both open AND closed — that address the same problem or a related area. Reference what you found in the "Existing PRs" section. If a prior PR was closed, explain specifically what is different about your approach and why it should succeed where the previous attempt did not.

**PRs that show no evidence of human involvement will be closed.** A human must review the complete proposed diff before submission.

## What We Will Not Accept

### Third-party dependencies

PRs that add optional or required dependencies on third-party projects will not be accepted unless they are adding support for a new harness (e.g., a new IDE or CLI tool). Superpowers is a zero-dependency plugin by design. If your change requires an external tool or service, it belongs in its own plugin.

### "Compliance" changes to skills

Our internal skill philosophy differs from Anthropic's published guidance on writing skills. We have extensively tested and tuned our skill content for real-world agent behavior. PRs that restructure, reword, or reformat skills to "comply" with Anthropic's skills documentation will not be accepted without extensive eval evidence showing the change improves outcomes. The bar for modifying behavior-shaping content is very high.

### Project-specific or personal configuration

Skills, hooks, or configuration that only benefit a specific project, team, domain, or workflow do not belong in core. Publish these as a separate plugin.

### Bulk or spray-and-pray PRs

Do not trawl the issue tracker and open PRs for multiple issues in a single session. Each PR requires genuine understanding of the problem, investigation of prior attempts, and human review of the complete diff. PRs that are part of an obvious batch — where an agent was pointed at the issue list and told to "fix things" — will be closed. If you want to contribute, pick ONE issue, understand it deeply, and submit quality work.

### Speculative or theoretical fixes

Every PR must solve a real problem that someone actually experienced. "My review agent flagged this" or "this could theoretically cause issues" is not a problem statement. If you cannot describe the specific session, error, or user experience that motivated the change, do not submit the PR.

### Domain-specific skills

Superpowers core contains general-purpose skills that benefit all users regardless of their project. Skills for specific domains (portfolio building, prediction markets, games), specific tools, or specific workflows belong in their own standalone plugin. Ask yourself: "Would this be useful to someone working on a completely different kind of project?" If not, publish it separately.

### Fork-specific changes

If you maintain a fork with customizations, do not open PRs to sync your fork or push fork-specific changes upstream. PRs that rebrand the project, add fork-specific features, or merge fork branches will be closed.

### Fabricated content

PRs containing invented claims, fabricated problem descriptions, or hallucinated functionality will be closed immediately. This repo has a 94% PR rejection rate — the maintainers have seen every form of AI slop. They will notice.

### Bundled unrelated changes

PRs containing multiple unrelated changes will be closed. Split them into separate PRs.

## Skill Changes Require Evaluation

Skills are not prose — they are code that shapes agent behavior. If you modify skill content:

- Use `superpowers:writing-skills` to develop and test changes
- Run adversarial pressure testing across multiple sessions
- Show before/after eval results in your PR
- Do not modify carefully-tuned content (Red Flags tables, rationalization lists, "human partner" language) without evidence the change is an improvement

## Understand the Project Before Contributing

Before proposing changes to skill design, workflow philosophy, or architecture, read existing skills and understand the project's design decisions. Superpowers has its own tested philosophy about skill design, agent behavior shaping, and terminology (e.g., "your human partner" is deliberate, not interchangeable with "the user"). Changes that rewrite the project's voice or restructure its approach without understanding why it exists will be rejected.

## General

- Read `.github/PULL_REQUEST_TEMPLATE.md` before submitting
- One problem per PR
- Test on at least one harness and report results in the environment table
- Describe the problem you solved, not just what you changed
