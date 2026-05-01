# Subagent Knowledge Skills — Design Notes

**Status:** Shape A validated (2026-04-30). `writing-plans` baked recommendations into plans, code-reviewer slot extended, and curated knowledge plugins (`react`, `react-native`, `nextjs`) wired into the marketplace (all 2026-04-30). Awaiting full end-to-end test exercising the multi-skill pipeline.
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

## Validation Results — 2026-04-30

**Outcome: Shape A works.** One real-world dispatch produced a `Skill` invocation by the implementer subagent. The "explicit instruction overrides default behavior" hypothesis holds in the n=1 case; pre-existing 38/0 baseline now reads 38/0 + 1/1.

### Setup

- **Repo under test:** `/Users/pat/Personal/mono-repo` (pnpm + Turborepo monorepo with an Expo SDK 54 app at `apps/app-expo-ts`).
- **Knowledge skill plugin installed via marketplace path:** `expo@psprowls-plugins` (11 skills, including `expo:building-native-ui`). Step (a) of "Next moves" satisfied via marketplace install rather than the cheaper single-file alternative.
- **Instrumentation added for evidence capture:** `PostToolUse` hook with matcher `Skill` writing one JSON line per invocation to `/tmp/skill-invocations.log`. Configured in `<repo>/.claude/settings.json`. Captures `ts`, `session_id`, `skill`, `args`. Cross-referenced with `OTEL_LOG_TOOL_DETAILS=1` already exporting to a local SigNoz instance — but the hook log alone was sufficient.

### Dispatch

- **Plan:** one-task plan (`docs/superpowers/plans/2026-04-30-validation-about-screen.md` in the consumer repo) — add a static About screen to the Expo app's settings stack. Three file edits: create `about.tsx`, register `<Stack.Screen>` in `_layout.tsx`, add a `List.Item` row in `index.tsx`.
- **Knowledge Skills slot fill:**
  ```
  - expo:building-native-ui — implementer is creating a new Expo Router file-based screen,
    registering it in the Stack navigator, and adding a navigation row from a List screen.
    The skill covers screen patterns, Stack.Screen options, file-based routing
    conventions, and Link/router.push usage that this task uses end-to-end.
  ```
- **Subagent type/model:** `general-purpose` / `sonnet`. Reinforcing language added to the dispatch prompt: "THIS IS NOT OPTIONAL. The very first tool call you make in this task must be the Skill tool with skill='expo:building-native-ui'. Do not read files, do not run bash, do not edit anything before that Skill call."

### Evidence

Hook log after the run, three entries, all under the same `session_id`:

```json
{"ts":"...:38:07Z","session":"f679...","skill":"claude-superpowers:writing-plans","args":"..."}
{"ts":"...:42:45Z","session":"f679...","skill":"claude-superpowers:subagent-driven-development","args":"..."}
{"ts":"...:44:02Z","session":"f679...","skill":"expo:building-native-ui","args":null}
```

Lines 1–2 are controller-side calls. **Line 3 is the implementer subagent's call** — the validation answer. The subagent's self-report ("invoked `expo:building-native-ui` as the very first tool call") matches the timestamp evidence.

The implementation also passed both `pnpm check-types` and `pnpm lint:check` (exit 0), and produced a clean single-feature commit. Imports came from `@psprowls/shared-ui-native-ts` (not `react-native`) and used custom NativeWind tokens (`bg-surface`, `text-primary`, `text-text-secondary`) — exactly the patterns the package's CLAUDE.md mandates and the kind of stack-specific guidance the skill is supposed to inject. Without the Knowledge Skills slot, the n=1 prediction is the subagent would have used `react-native` imports and standard Tailwind palette names (or hex colors). With it: clean fit on first try.

### Architectural findings worth recording

1. **Subagent tool calls share the parent's `session_id` and fire the parent's `PostToolUse` hooks.** The doc's earlier hedge ("if not present, look for a sibling subagent transcript file") can be retired — there is no separate subagent JSONL on this version of CC. The parent transcript is the single source of truth; a hook on the parent process catches subagent tool use too.
2. **A 3-line `PostToolUse` hook + `OTEL_LOG_TOOL_DETAILS=1` is overkill for this validation.** Either alone is sufficient. Future runs can drop the OTel cross-check and rely on the hook log, which is faster to consume.
3. **IDE diagnostics false-positive:** the editor's TypeScript server flagged `Property 'className' does not exist on type 'ScrollViewProps'` immediately after the implementer's writes, but `tsc --noEmit` passed. Root cause is the LSP not loading `nativewind-env.d.ts` augmentations in time. This matters for future review-pipeline patterns: any naive "scrape IDE diagnostics" reviewer would have rejected a correct implementation. The authoritative signal is `tsc`, not the editor's diagnostic stream.
4. **Subagent honesty held up.** The dispatched agent's claim "check-types passed" was true even when the IDE diagnostics suggested otherwise. Trust-but-verify still warranted, but the agent did not fabricate a result.

### Recommended next moves (now unblocked)

Shape A green path applies:

- **(High value, low risk)** Update `writing-plans` to suggest stack-relevant skills at plan-creation time, so the controller doesn't have to detect stack at dispatch time. The plan would carry the recommended Knowledge Skills list per task.
- **(High value, medium risk)** Build curated `superpowers-knowledge-react` / `-expo` / `-typescript` plugins under your fork. Each is a small SKILL.md set that captures stack patterns the implementer should default to. Only proceed once you've used existing 3rd-party packs (expo, next, vercel-react) for a few weeks and have a sense of what overlaps and what's missing.
- **(Riskier, eval-gated)** Extend the pattern to `code-quality-reviewer-prompt.md` and the underlying `code-reviewer` agent. Adding a slot to the dispatch fields is cheap; modifying `agents/code-reviewer.md`'s system prompt is the part the contributor policy warns about. Run a small eval set first — same code change, with vs. without slot, observe whether spec compliance and code quality improve.
- **(Reproducibility)** Capture future validation runs in this same doc (or a sibling `validation-results.md`). The hook + OTel + JSONL stack is now a proven recipe; new runs should add to a results table rather than rediscover the methodology.

### Open questions for follow-up validation

- **n=1 is not n=many.** This run used `sonnet` and a relatively distinctive skill (`expo:building-native-ui`) on a task with strong fit. Does the pattern hold for `haiku` (cheaper) and for less distinctive skills? Worth a sweep.
- **Multiple skills in one slot:** when 3+ skills are listed, does the subagent invoke all of them? Or does it cherry-pick? Untested.
- **Slot drift:** if the controller fills the slot with skills that don't exist (typo, plugin uninstalled), does the subagent silently no-op or error visibly? Untested.

## Plan-time recommendation (shipped 2026-04-30)

Implements next-move (a) from the prior section: shift Knowledge Skill selection from dispatch time (controller-side) to plan-creation time (plan-author-side).

**Edits:**

- `skills/writing-plans/SKILL.md` — new `## Knowledge Skill Recommendations` section explaining the 38/0 finding (by reference, not duplicated), detection heuristic, and per-task selection rules. Task Structure template grew a `**Knowledge Skills:**` field between `**Verify:**` and `**Steps:**`. Self-Review checklist gained item 4 (verify the field is filled on every task). Native Task metadata example and Task Persistence example now include a `knowledgeSkills` array.
- `skills/shared/task-format-reference.md` — `knowledgeSkills` added to the metadata schema table as an optional `string[]`.
- `skills/subagent-driven-development/SKILL.md` — Stack Detection section now reads "Prefer plan-supplied recommendations" first, with detection-at-dispatch-time documented as the fallback for plans authored without writing-plans. Includes a validation step: strip listed skills that aren't in the current available-skills system reminder rather than fabricating substitutes.

**Why this shape:**

- The plan author has full context — they're inspecting the working directory, looking at file extensions for tasks they're actively writing, and seeing the available-skills list at plan-creation time. Asking the controller to redo this work at dispatch time wastes effort and risks drift if the dispatching session has different installed plugins (the SDD validation step handles that case explicitly).
- Per-task remains the right granularity. Different tasks in the same plan may touch different stacks; baking a single plan-level default would lose that.
- Empty arrays are explicit (`None — proceed without skill loading.`) rather than missing. Missing is a plan bug; explicit empty is a valid choice for trivial tasks (config edits, doc changes).

**Deliberately not changed:**

- `executing-plans` — that skill executes plans in a session where the user/main agent is the worker, not via subagent dispatch. The `Skill` tool is natively available there; no slot is needed. If executing-plans starts dispatching subagents in the future, mirror the SDD changes.
- `brainstorming` — the brainstorming skill produces specs, not implementation plans. Knowledge Skill selection happens one layer down (writing-plans).

**What to verify on next live use:**

- Open a fresh CC session, run `claude-superpowers:writing-plans` against a real task in `mono-repo`. Confirm the plan written to disk contains `**Knowledge Skills:**` per task and `knowledgeSkills` in the embedded metadata JSON.
- Then dispatch via `subagent-driven-development`. Confirm the controller reads the plan-supplied list rather than re-detecting (you should see the slot filled identically to what the plan said). Hook log at `/tmp/skill-invocations.log` should still show the implementer invoking the listed skills.
- If a plan-supplied skill is uninstalled, confirm SDD strips it instead of dispatching with a broken name. Probably worth manually testing this edge case once.

## Code-reviewer slot extension (shipped 2026-04-30)

Implements the second next-move from the validation results section: extend the Knowledge Skills pattern to the code-quality reviewer dispatch. This was the eval-gated edit because it touches the `code-reviewer` agent system prompt — "carefully-tuned" content per the contributor policy. We're shipping to the fork now and validating in the same end-to-end test that exercises the writing-plans changes.

**Edits:**

- `skills/requesting-code-review/code-reviewer.md` — new `## Knowledge Skills (Invoke FIRST, before reviewing)` section near the top with a `{KNOWLEDGE_SKILLS}` placeholder. Strong "invoke first, before reading any code or running git commands" instruction; explicit empty case (`None — proceed to review.`).
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md` — `KNOWLEDGE_SKILLS:` template field added to the dispatch fields list. Note instructing the controller to use the same slot fill as implementer + spec reviewer.
- `agents/code-reviewer.md` — one additive sentence after the opening "Senior Code Reviewer" line: "If your dispatch prompt contains a `## Knowledge Skills` section listing one or more skills, invoke each via the Skill tool before any other action..." Does not restructure the six review sections; does not change the senior-reviewer voice.
- `skills/subagent-driven-development/SKILL.md` — Stack Detection section now lists all three dispatch templates and notes the slot fill must match across them. Red Flag bullet generalized from "implementer" to "any of the three subagents".

**Why this is lower-risk than feared:**

The validation dispatch (2026-04-30) showed the implementer subagent (`general-purpose` agent type, no custom system prompt) honored an "invoke skill X first" instruction in its dispatch prompt. The code-reviewer agent has a custom system prompt but it doesn't conflict with the instruction — it tells the agent what review sections to produce, not what to do first. Adding a sentence to the system prompt that says "honor the slot if present" is belt-and-suspenders; the dispatch-side instruction alone is probably sufficient. Either way, the change is additive — nothing is removed or restructured.

**Eval still required:**

Per the original deferral note, the contributor policy bar for behavior-shaping changes is real. The full end-to-end test (next move: install curated knowledge plugins, run a multi-task plan through writing-plans → subagent-driven-development) will surface any regression. If the code-reviewer agent starts misbehaving (skipping reviews, getting stuck on skill loading, or applying skill guidance to areas it shouldn't), back this change out and pivot to dispatch-side instruction only.

**What to verify in the full test:**

- The code-quality reviewer subagent's tool log shows `Skill` invocations matching the slot fill, before any `Read` / `Bash` / git commands.
- The reviewer produces its standard output structure (Strengths / Issues / Recommendations / Assessment) — slot loading didn't displace the review.
- Reviewer flags a stack-specific violation it would have missed without the skills (the eval-positive case). Or, if no violation exists, confirm review duration / token usage didn't blow up.

## Curated knowledge plugins (shipped 2026-04-30)

Three new plugins wired into the `psprowls-plugins` marketplace at `/Users/pat/Personal/claude-plugins/.claude-plugin/marketplace.json`. Each plugin is a thin wrapper (plugin.json + symlinked skills/) around upstream skill repos already checked out under `~/Personal/skills/`. Symlinks (not copies) so upstream `git pull` updates flow without rebundling.

| Plugin | Skills | Upstream |
|---|---|---|
| `react` | `vercel-react-best-practices`, `vercel-composition-patterns`, `web-design-guidelines` | `vercel/agent-skills` (`~/Personal/skills/agent-skills/skills/`) |
| `react-native` | `vercel-react-native-skills` | `vercel/agent-skills` |
| `nextjs` | `next-best-practices`, `next-cache-components`, `next-upgrade` | `vercel/next-skills` (`~/Personal/skills/next-skills/skills/`) |

**Plugin layout:**

```
/Users/pat/Personal/claude-plugins/{react,react-native,nextjs}/
├── .claude-plugin/plugin.json
├── README.md
└── skills/
    └── <skill-name>/        ← symlink to ~/Personal/skills/.../<skill>/
```

**Naming notes:**
- The Vercel skills retain their `vercel-` frontmatter prefix to preserve attribution. Slot fills read as `react:vercel-react-best-practices` (slightly redundant but faithful to upstream).
- `web-design-guidelines` is framework-agnostic (UI/accessibility review) but lives in `react` because that's the most likely consumer for this fork's monorepo work. If a future plan touches non-React UI, the slot fill can still reference it directly.

**Install (manual, run in CC):**

```
/plugin marketplace reload psprowls-plugins
/plugin install react@psprowls-plugins
/plugin install react-native@psprowls-plugins
/plugin install nextjs@psprowls-plugins
```

After install, a new session's available-skills system reminder should list 7 new skills across the three plugins.

**Failure modes to watch for:**
- Symlink target moves/deletes — plugin shows empty skill list. Re-clone upstream at expected path.
- Two plugins with the same skill `name` — CC's namespace-by-plugin should prevent collisions (e.g., `nextjs:next-best-practices` vs `react:next-best-practices` if a future skill duplicates the name). Untested in this setup; only worth resolving if it happens.
- The `vercel-` prefix annoyance — if it becomes a real problem, fork the SKILL.md frontmatter under our plugin dir (replace the symlink with a copy) and rename. Not doing this preemptively.

## Files referenced

- `skills/subagent-driven-development/SKILL.md` — controller skill (modified Shape A + plan-supplied preference + all-three-dispatches generalization)
- `skills/subagent-driven-development/implementer-prompt.md` — implementer dispatch template (modified)
- `skills/subagent-driven-development/spec-reviewer-prompt.md` — spec reviewer dispatch template (modified)
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md` — code quality reviewer dispatch template (modified: KNOWLEDGE_SKILLS field added)
- `skills/requesting-code-review/code-reviewer.md` — code review prompt template (modified: Knowledge Skills section + placeholder)
- `skills/writing-plans/SKILL.md` — plan author skill (modified to recommend skills per task at plan-creation time)
- `skills/shared/task-format-reference.md` — task metadata schema (added `knowledgeSkills`)
- `agents/code-reviewer.md` — code-reviewer agent system prompt (modified: one additive sentence to honor the slot)
- `skills/using-superpowers/SKILL.md` — see `<SUBAGENT-STOP>` marker (line 6) for why subagents don't auto-bootstrap
- `CLAUDE.md` — contributor policy. Domain skills must live in separate plugins, not core.
