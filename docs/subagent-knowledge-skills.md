# Subagent Knowledge Skills — Design Notes

**Status:** Shape A implemented (2026-04-30). Awaiting validation dispatch.
**Owner:** pcvelz fork.

---

## Problem

We want subagents dispatched by `subagent-driven-development` to apply stack-specific expertise (React, TypeScript, Expo, Next.js, etc.) on top of the existing workflow skills (TDD, debugging, review). The original idea was to define new agents per stack — `react-implementer`, `expo-reviewer`, etc. Investigation showed that's the wrong shape.

## Why not "more agents"

Agents in CC are a fixed system prompt + tool allowlist. They:
- Don't compose (skills do)
- Multiply combinatorially (role × stack)
- Are CC-only, but we don't care about cross-harness lock-in on this fork
- Don't auto-bootstrap superpowers skills (the `<SUBAGENT-STOP>` marker in `skills/using-superpowers/SKILL.md` deliberately stops the bootstrap from re-applying inside subagents)

The architecture's center of gravity is **skills as the unit of knowledge, controller as the orchestrator**. We extend that, not invert it.

## Empirical evidence: subagents don't reach for skills voluntarily

Across **38 real-world dispatches** of `claude-superpowers:code-reviewer` spanning 5 projects (deep-wiki-agent, local-agent, mono-repo, mono-repo-domains-shared, plugins) and months of use:

| Metric | Value |
|---|---|
| `toolUseResult.toolStats.otherToolCount > 0` | **0 / 38** |
| Sum of `otherToolCount` (would include `Skill`) | **0** |
| `readCount` per dispatch | 0–49 |
| `bashCount` per dispatch | 2–40 |

**The Skill tool was never invoked in any subagent dispatch.** The agent's system prompt is self-contained (six numbered review sections that duplicate `requesting-code-review`'s methodology), so it has no incentive to reach. The same shape exists in `implementer-prompt.md` — TDD discipline encoded inline, the actual `test-driven-development` skill never named.

**Implication:** if you want a subagent to apply a skill, **you must name the skill explicitly in its dispatch prompt.** Hope-based composition does not work.

Methodology: `grep -h '"subagent_type":"[^"]*"' ~/.claude/projects/*/*.jsonl | sort | uniq -c` then parse `toolUseResult` from each matching line. Reproducible.

## Design: Shape A (chosen)

Add a `## Knowledge Skills` slot at the top of subagent prompt templates. Controller detects stack from `package.json` + file extensions and fills the slot with relevant skill names + one-line reasons. Subagent is instructed (with strong language) to invoke each skill via the `Skill` tool BEFORE doing anything else.

**Alternatives considered:**

- **Shape B — subagent self-detects:** "look at files, invoke matching skills." Rejected: gives discretion to a subagent we know doesn't reach for skills voluntarily. Same failure mode as today.
- **Shape C — stack-specific prompt files (`implementer-prompt-react.md`):** matches the existing inline-knowledge pattern; no skill invocation needed. Rejected as the *first* move because it forks the template (drift risk) and forecloses the skill-composition path. Will become the right answer if Shape A's validation dispatch shows subagents ignore explicit "invoke X" instructions too.
- **Agents-with-baked-in-skill-references:** `react-implementer.md` agent whose system prompt always invokes specific skills. Real option for narrow high-reliability surfaces (reviewers, security audits) but multiplies file count and inverts the architecture.

**Why Shape A first:** additive, testable, doesn't touch tuned content, and the validation result tells us whether to scale this pattern (Shape A wins) or pivot to Shape C (Shape A failed — explicit instruction doesn't help).

## What changed (2026-04-30)

Three additive edits, no tuned content modified:

### `skills/subagent-driven-development/implementer-prompt.md`
- New `## Knowledge Skills (Invoke FIRST, before anything else)` section between the task name (line 9) and `## Task Description`.
- Slot format: `- plugin:skill-name — one-line reason this applies to this task`. Empty case is `None — proceed to Task Description.`
- "Your Job" step list grew from 6 to 7 steps. New step 1 is "Invoke every skill listed in the Knowledge Skills section above (if any)." Original steps 1–6 became 2–7.

### `skills/subagent-driven-development/spec-reviewer-prompt.md`
- Same `## Knowledge Skills` slot, inserted between "You are reviewing whether..." and `## What Was Requested`.
- Rationale: reviewer needs the same domain context as the implementer to verify documented patterns were followed.

### `skills/subagent-driven-development/SKILL.md`
- New `## Stack Detection & Knowledge Skill Selection` section under "Dispatching with Metadata."
- Documents: the 38/0 empirical finding (so future readers know *why* the slot is required), the detection heuristic (package.json, file extensions, language manifests), the rules (≤5 skills per dispatch, reasons mandatory, reuse selection across implementer + reviewers, "None" when nothing matches), and an example slot fill.
- New Red Flag bullet: "Dispatch an implementer with the `## Knowledge Skills` section blank or unset" — blank slot is a bug, not a default.
- One example-workflow line (Task 1) demonstrates a filled slot.

## Deliberately not done

### `code-quality-reviewer-prompt.md` and the `code-reviewer` agent
The code-quality reviewer dispatches `claude-superpowers:code-reviewer` — the agent with the 38/0 empirical record. Adding a Knowledge Skills slot to the dispatch fields won't work unless the agent's *system prompt* (in `agents/code-reviewer.md`) also instructs it to invoke skills. That's a larger, riskier edit because agent system prompts are exactly the "carefully-tuned" content the contributor policy warns against modifying without eval evidence. Defer until Shape A is validated for the implementer + spec reviewer.

### Knowledge skill plugins themselves
Domain-specific skills don't belong in superpowers core (contributor policy explicitly rejects this). They belong in separate plugins. Reference packs already downloaded at:

- `/Users/pat/Personal/skills/expo-skills/plugins/expo/skills/` — Expo (native-data-fetching, building-native-ui, expo-deployment, expo-tailwind-setup, expo-cicd-workflows, etc.)
- `/Users/pat/Personal/skills/next-skills/skills/` — Next.js (next-best-practices, next-cache-components, next-upgrade)
- `/Users/pat/Personal/skills/agent-skills/skills/react-best-practices/SKILL.md` — Vercel React perf guidelines (`vercel-react-best-practices`)
- `/Users/pat/Personal/skills/agent-skills/skills/composition-patterns/`, `react-native-skills/`, `web-design-guidelines/`
- `/Users/pat/Personal/skills/alirezarezvani/claude-skills/engineering/` — large pack (~30 engineering skills)

These are the candidate inputs for next-move (a).

### Validation dispatch
Can't run a real subagent dispatch from a planning conversation. Required to confirm Shape A actually causes skill invocation. See next-move (b).

## Next moves

### (a) Install one knowledge-skill pack so the slot has something to fill

**Goal:** Wire one external skill pack into the CC environment so the new `## Knowledge Skills` slot can reference real skill names.

**Concrete install path (verified 2026-04-30):** the user already has a local marketplace `psprowls-plugins` registered at `/Users/pat/Personal/claude-plugins/` (see `~/.claude/plugins/known_marketplaces.json`), with a `marketplace.json` listing one plugin (`llm-code-wiki`). The Expo pack at `/Users/pat/Personal/skills/expo-skills/plugins/expo` already has a valid `.claude-plugin/plugin.json` (name: `expo`, 11 skills). To make it installable via the existing marketplace:

1. Add `expo` (and optionally `next`, `react-best-practices`) to `/Users/pat/Personal/claude-plugins/.claude-plugin/marketplace.json`. Each entry needs `name`, `source` (path to the plugin dir, can be absolute or relative), `description`. The `expo` entry's `source` would be `/Users/pat/Personal/skills/expo-skills/plugins/expo` (or a symlink under `claude-plugins/` to keep the marketplace dir self-contained).
2. Reload the marketplace in CC (`/plugin marketplace reload psprowls-plugins` or restart session).
3. Install: `/plugin install expo@psprowls-plugins`.
4. Verify in a fresh session: skills with names like `expo:native-data-fetching`, `expo:building-native-ui` should appear in the available-skills system reminder.

**Recommended first pack:** `expo` — it's the largest of the candidate packs (11 skills covering UI, data fetching, deployment, CI/CD, upgrades) and matches the user's mono-repo Expo work, so the validation dispatch in step (b) will have natural skill matches.

**Cheaper alternative:** copy a single SKILL.md (e.g. `agent-skills/skills/react-best-practices/SKILL.md`) into the user's global `~/.claude/skills/` (CC auto-discovers user-level skills without a plugin manifest). One file, no marketplace edit. Tradeoff: doesn't exercise the plugin-install path.

**Success criterion:** at least one skill name from a non-superpowers plugin appears in CC's available-skills list and can be invoked via the `Skill` tool from the main session.

**Caveat:** modifying `/Users/pat/Personal/claude-plugins/.claude-plugin/marketplace.json` is outside this repo. Either edit it manually or have a CC session that has access to that path do it.

### (b) Validation dispatch — does Shape A actually cause skill invocation?

**Goal:** Confirm the empirical question. Across 38 prior dispatches, subagents invoked Skill 0 times. Does an explicit "invoke this skill via the Skill tool BEFORE doing anything else" instruction change that?

**Steps:**
1. In a real Expo/React project (suggest one of the `mono-repo` apps under `~/.claude/projects/`-pattern), open a CC session.
2. Use `claude-superpowers:writing-plans` to create a small one-task plan that touches React/Expo code (e.g., "add a new screen with a fetch call"). The task should naturally benefit from one of the installed knowledge skills.
3. Use `claude-superpowers:subagent-driven-development` to execute. The controller (this CC session) should now follow the new SKILL.md guidance: detect stack, select skills, fill the `## Knowledge Skills` slot before dispatching the implementer.
4. After the dispatch returns, locate the transcript at `~/.claude/projects/<project-slug>/<session-id>.jsonl`.
5. Find the implementer dispatch's `toolUseResult` and check `toolStats.otherToolCount` — and ideally the full subagent transcript if CC stores it separately.

**Inspection recipe (jq):**

```bash
# Replace with the project slug for the validation session.
PROJECT_SLUG="-Users-pat-Personal-mono-repo-apps-myapp"

# Per-dispatch otherToolCount across the most recent session for that project.
LATEST=$(ls -t ~/.claude/projects/$PROJECT_SLUG/*.jsonl 2>/dev/null | head -1)
echo "Inspecting: $LATEST"

# All Task-tool dispatches in that session, with otherToolCount.
jq -c 'select(.toolUseResult.type == "task")
       | {desc: .message.content[0].text? // .toolUseResult.totalDurationMs,
          subagent_type: .message.tool_use_input.subagent_type? // null,
          stats: .toolUseResult.toolStats}' "$LATEST"

# Drill into a single dispatch to see actual tool names called by the subagent.
# CC stores the subagent's tool calls inline in toolUseResult.usage / .content;
# if not present, look for a sibling subagent transcript file in the same dir.
jq 'select(.toolUseResult.type == "task") | .toolUseResult' "$LATEST" | head -200
```

If `toolStats.otherToolCount` is non-zero, search for actual `Skill` invocations — `TodoWrite`, `WebFetch`, etc. all bucket into `otherToolCount` and would be false positives. The definitive check is the raw tool-call list in the subagent's content stream.

**Success criterion:** `toolStats.otherToolCount > 0` for the implementer dispatch, with the increment attributable to `Skill` invocations (not `TodoWrite` or other "other" tools — may need to inspect the subagent's actual tool calls, not just stats).

**Failure modes to watch for:**
- Slot filled but `otherToolCount` still 0 → explicit instruction insufficient. Pivot to Shape C (stack-specific prompt files with inline knowledge).
- `otherToolCount > 0` but the subagent invoked unrelated tools (`TodoWrite`, etc.) → need finer-grained data. Look at the raw tool call list, not just the `toolStats` summary.
- Controller (this CC session) didn't follow the new SKILL.md guidance and dispatched with empty slot → SKILL.md instructions need stronger language, not a Shape A architecture failure.

**Reproducibility:** record what was dispatched, what was in the slot, and the resulting `toolStats`. Add to a follow-up section in this doc (or a new `validation-results.md`).

## Decision flow after validation

```
otherToolCount > 0 with Skill invocations
  └─ Shape A works. Proceed to:
     - Extend pattern to code-reviewer agent (riskier; eval first)
     - Build superpowers-knowledge-react / -expo plugins (your own curated packs)
     - Update writing-plans to suggest stack-relevant skills at plan-creation time

otherToolCount still 0 even with explicit instruction
  └─ Subagents truly don't invoke skills. Pivot to:
     - Shape C: stack-specific prompt files with inline knowledge
     - OR: agents-with-baked-in-skill-references for narrow, high-reliability surfaces
     - Either way: knowledge lives inline in the prompt, not behind a Skill tool call
```

## Files referenced

- `skills/subagent-driven-development/SKILL.md` — controller skill (modified)
- `skills/subagent-driven-development/implementer-prompt.md` — implementer dispatch template (modified)
- `skills/subagent-driven-development/spec-reviewer-prompt.md` — spec reviewer dispatch template (modified)
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md` — code quality reviewer dispatch template (NOT modified; deferred)
- `agents/code-reviewer.md` — code-reviewer agent system prompt (NOT modified; deferred)
- `skills/using-superpowers/SKILL.md` — see `<SUBAGENT-STOP>` marker (line 6) for why subagents don't auto-bootstrap
- `CLAUDE.md` — contributor policy. Domain skills must live in separate plugins, not core.
