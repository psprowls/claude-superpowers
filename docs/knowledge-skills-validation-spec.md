# Knowledge Skills Pipeline — Full E2E Validation Spec

**Purpose:** End-to-end validation of the Knowledge Skills pipeline. Exercises every link added in the four 2026-04-30 commits:

1. `writing-plans` recommends skills per task at plan-creation time
2. `subagent-driven-development` reads plan-supplied skills, validates them, dispatches all three subagents
3. The `code-reviewer` agent (whose system prompt was edited) honors the slot
4. Real curated knowledge plugins (`react`, `react-native`, `expo`) provide skills the slots can reference

**Predecessor:** the n=1 validation on 2026-04-30 (`docs/subagent-knowledge-skills.md` "Validation Results"). That run confirmed Shape A works for one task, one plugin (`expo`), one dispatch type (implementer). This run scales to multiple tasks, multiple plugins, all three dispatches.

---

## Where to run

`/Users/pat/Personal/mono-repo` — the same Expo+monorepo used in the n=1 run. The `apps/app-expo-ts` Expo app already has the About screen the n=1 added; this spec extends that pattern.

If your mono-repo also has a Next.js app, an optional 4th task exercises the `nextjs` plugin (see "Optional extension" below). If not, skip it.

---

## Pre-flight

Run all of these in the target repo before opening the test session.

```bash
# 1. Confirm the parent marketplace + new plugin dirs exist.
ls /Users/pat/Personal/claude-plugins/{react,react-native,nextjs}/.claude-plugin/plugin.json

# 2. Confirm the symlinks resolve to real SKILL.md files.
ls -L /Users/pat/Personal/claude-plugins/{react,react-native,nextjs}/skills/*/SKILL.md

# 3. Confirm the PostToolUse hook from the n=1 run is still wired.
#    Should print a JSON object containing PostToolUse → Skill matcher.
jq '.hooks.PostToolUse' /Users/pat/Personal/mono-repo/.claude/settings.json

# 4. Wipe the prior hook log so this run's output is clean.
: > /tmp/skill-invocations.log

# 5. (Optional) Confirm OTel is still pointing at the local SigNoz instance
#    if you want a second source of truth. Not required.
env | grep OTEL
```

Then **open a fresh Claude Code session** at `/Users/pat/Personal/mono-repo` and run, in that session:

```
/plugin marketplace reload psprowls-plugins
/plugin install react@psprowls-plugins
/plugin install react-native@psprowls-plugins
/plugin install nextjs@psprowls-plugins
```

After install, ask Claude: **"List the available skills you can see whose name starts with `react:`, `react-native:`, `nextjs:`, or `expo:`."** You should see 7 + 11 = **18 skills** across the four plugins. If you see fewer, something didn't install — fix before proceeding.

---

## The feature (paste this into the session)

> I want to add a "What's New" screen to the Expo app's settings stack at `apps/app-expo-ts`. The screen displays a collapsible release-notes list. Each release entry has a date, a title, and a list of bullet-point changes. Sections are collapsed by default and expand on tap.
>
> **Requirements:**
>
> - Build a generic `<Collapsible>` component in the shared UI package (`packages/shared-ui-native-ts`). It should use a composition pattern (compound components or render props) so the consumer can pass in whatever header and body content they want — not a boolean-prop API. The component must meet the Web Interface Guidelines for keyboard/screen-reader accessibility (where mobile equivalents apply: proper `accessibilityRole`, `accessibilityState`, hit area).
> - Add a static `release-notes.json` file under `apps/app-expo-ts/assets/data/` containing 3 mock release entries.
> - Add a small data hook or fetch utility that loads `release-notes.json` and returns the parsed entries. Follow whatever fetch/loading pattern this codebase already uses.
> - Add a `whats-new.tsx` screen under `apps/app-expo-ts/app/(settings)/` (or wherever the existing settings stack lives) that renders the release notes using the `<Collapsible>` component. Register it in the settings stack `_layout.tsx` and add a navigation row from the settings index screen.
>
> **Style and stack constraints:**
>
> - Imports must come from `@psprowls/shared-ui-native-ts`, not `react-native` directly.
> - Tailwind classes must use the custom NativeWind tokens (`bg-surface`, `text-primary`, `text-text-secondary`, etc.) — not hex colors or generic Tailwind palette names.
> - TypeScript strict; no `any`.
>
> **Verification:** `pnpm check-types` and `pnpm lint:check` must both pass at exit 0 for every commit.
>
> Use `claude-superpowers:writing-plans` to write the plan. Plan-creation should detect the stack and fill `**Knowledge Skills:**` per task. Then use `claude-superpowers:subagent-driven-development` to execute.

The plan that comes out should have **3 tasks** roughly along these lines:

| Task | Scope | Expected `Knowledge Skills` slot |
|---|---|---|
| 1 | `<Collapsible>` component in shared UI package, with tests | `react:vercel-composition-patterns`, `react:web-design-guidelines`, `claude-superpowers:test-driven-development` |
| 2 | Release-notes JSON + data hook in Expo app | `expo:native-data-fetching`, `claude-superpowers:test-driven-development` |
| 3 | `whats-new.tsx` screen + stack registration + settings row | `expo:building-native-ui`, `react:web-design-guidelines` |

If `writing-plans` produces a different decomposition, that's fine — what matters is that **each task's `Knowledge Skills` field is filled with a sensible per-task selection** (not a copy-pasted plan-level default), and that the slots reference real skills from the installed plugins.

### Optional extension (skip if no Next.js app)

If the mono-repo also has a Next.js app, append this to the feature spec:

> Mirror the same "What's New" feature in the Next.js app at `apps/app-web` (or wherever): a route segment (`app/(settings)/whats-new/page.tsx`) that fetches the same release-notes data via a route handler and renders it with collapsible sections. This task should use `nextjs:next-best-practices` and `nextjs:next-cache-components` (since release notes are perfect cache fodder).

---

## What to watch during the run

The controller (this Claude Code session) should announce each dispatch and show the slot fill. Watch for:

1. **Plan-time slot fills.** When `writing-plans` finishes, open the plan markdown and confirm each task has a `**Knowledge Skills:**` field. The fields should differ across tasks — not the same list copy-pasted three times.

2. **Dispatch-time consistency.** When `subagent-driven-development` dispatches the implementer, then the spec reviewer, then the code-quality reviewer for one task, the slot fills should **match** across all three. If they drift, the SDD skill failed.

3. **Skill validation.** If the plan happens to recommend a skill that isn't installed (typo, plugin uninstalled), SDD should strip it and proceed — not fabricate a substitute and not refuse to dispatch. (You can manually break this by editing the plan to reference `react:nonexistent-skill`, but it's a side experiment.)

4. **Live hook log.** Tail `/tmp/skill-invocations.log` during the run. Each dispatch's first tool call should be `Skill` with one of the listed names.

```bash
tail -f /tmp/skill-invocations.log
```

---

## Post-flight validation queries

After the run finishes (all 3 tasks complete, final reviewer approved, you're back at the prompt), run these queries.

### 1. Skill invocation count by dispatch type

```bash
# Total Skill calls — expect ≥9 for 3 tasks × 3 dispatches × ≥1 skill each.
wc -l /tmp/skill-invocations.log

# Group by skill name — confirms which plugins fired.
jq -r '.skill' /tmp/skill-invocations.log | sort | uniq -c | sort -rn
```

**Pass criterion:** every skill that appeared in any task's slot fill shows up at least once in the log. The Vercel + Next + Expo skills you expected are all represented.

### 2. Cross-dispatch consistency for one task

Pick the largest task (probably Task 1 — the Collapsible component, since it has 3 skills in the slot). Find its three dispatches in the log:

```bash
# Get the latest session's transcript path.
PROJECT_SLUG=$(echo /Users/pat/Personal/mono-repo | sed 's|/|-|g')
LATEST=$(ls -t ~/.claude/projects/${PROJECT_SLUG}/*.jsonl | head -1)
echo "Inspecting: $LATEST"

# Pull all Task-tool dispatches with their tool stats.
jq -c 'select(.toolUseResult.type == "task")
       | {desc: (.message.tool_use_input.description // ""),
          subagent: (.message.tool_use_input.subagent_type // ""),
          stats: .toolUseResult.toolStats}' "$LATEST"
```

**Pass criterion:** for each task, three dispatches (`Implement Task N`, `Review spec compliance for Task N`, `Code quality review for Task N`) appear, all with `toolStats.otherToolCount > 0`. If the code-quality reviewer's count is 0 while the others are non-zero, the agent system prompt edit didn't take — back it out per the back-out plan below.

### 3. Code correctness sanity check

```bash
cd /Users/pat/Personal/mono-repo
pnpm check-types
pnpm lint:check
git log --oneline -10
```

**Pass criterion:** type-check and lint exit 0; the commit log shows three feature commits authored by the implementer subagents in sequence.

### 4. Pattern-application check (the actual point)

The whole reason for this pipeline is to get subagents to follow stack-specific patterns they wouldn't apply unprompted. Verify:

- `<Collapsible>` uses a compound-component or render-prop API — **not** boolean props (`isOpen`, `showHeader`, etc.). If it uses boolean props, the composition-patterns skill didn't shape behavior.
- Imports across all new files come from `@psprowls/shared-ui-native-ts`, not `react-native`. Custom NativeWind tokens used.
- `accessibilityRole`/`accessibilityState`/proper hit-area on the Collapsible header. If missing, the web-design-guidelines skill didn't shape behavior.
- Data hook uses the existing fetch/Query pattern (whatever the codebase has) — not a fresh `useEffect`+`fetch` if that's not the convention.

If any of these miss, the skill was loaded but the subagent didn't apply it — that's a different (and worse) failure than not loading the skill at all. Note which one and which skill, since it tells us whether to invest in stronger "you must follow this skill's guidance" framing or to accept that some skills are noisier than others.

---

## Success criteria summary

The run is a pass if **all** of these hold:

- [ ] Plan markdown contains `**Knowledge Skills:**` for every task; selections differ per task.
- [ ] Hook log has ≥9 `Skill` invocations across the run.
- [ ] All three dispatch types per task show non-zero `otherToolCount` (implementer + spec reviewer + code-quality reviewer all invoked skills).
- [ ] `pnpm check-types` and `pnpm lint:check` exit 0 on every commit.
- [ ] The four pattern-application checks above pass on visual inspection.

The run is a partial pass (extension worth shipping; one component needs revision) if 4 of 5 hold and the failing one is identifiable.

---

## Back-out plan

If something regresses, the changes are individually revertable:

| If this regresses | Revert this commit | Effect |
|---|---|---|
| Code-quality reviewer skips its job, gets stuck on skill loading, or applies skills out of scope | `bacb803` (the `agents/code-reviewer.md` sentence) — revert just that file's change, keep the dispatch-side `KNOWLEDGE_SKILLS` field | Reviewer falls back to dispatch-only instruction; less belt-and-suspenders but should still work |
| Plan-time recommendations are noisy or wrong | `ed61321` | SDD reverts to detecting at dispatch time (the original Shape A); writing-plans no longer recommends skills |
| Subagents stop invoking skills entirely | `12ffe90` (Shape A original) | Whole pattern reverts; back to inline-knowledge prompts |

Each commit is small and independent — partial revert is safe.

---

## Reproducibility

After the run, append a "Validation Results — n=many" section to `docs/subagent-knowledge-skills.md` mirroring the n=1 results format. Capture: which tasks ran, slot fills used, hook-log skill counts, pattern-application check results, and any unexpected behaviors. That keeps the doc a living record of the pipeline's empirical track record.
