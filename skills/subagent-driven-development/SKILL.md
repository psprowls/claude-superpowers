---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with two-stage review after each: spec compliance review first, then code quality review.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh subagent per task (no context pollution)
- Two-stage review after each task: spec compliance first, then code quality
- Faster iteration (no human-in-loop between tasks)

## The Process

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Dispatch implementer subagent (skills/subagent-driven-development/implementer-prompt.md)" [shape=box];
        "Implementer subagent asks questions?" [shape=diamond];
        "Answer questions, provide context" [shape=box];
        "Implementer subagent implements, tests, commits, self-reviews" [shape=box];
        "Dispatch spec reviewer subagent (skills/subagent-driven-development/spec-reviewer-prompt.md)" [shape=box];
        "Spec reviewer subagent confirms code matches spec?" [shape=diamond];
        "Implementer subagent fixes spec gaps" [shape=box];
        "Dispatch code quality reviewer subagent (skills/subagent-driven-development/code-quality-reviewer-prompt.md)" [shape=box];
        "Code quality reviewer subagent approves?" [shape=diamond];
        "Implementer subagent fixes quality issues" [shape=box];
        "TaskUpdate: mark task completed" [shape=box];
    }

    "Read plan, extract tasks, TaskCreate for each with full text" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Dispatch final code reviewer subagent for entire implementation" [shape=box];
    "Use claude-superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract tasks, TaskCreate for each with full text" -> "Dispatch implementer subagent (skills/subagent-driven-development/implementer-prompt.md)";
    "Dispatch implementer subagent (skills/subagent-driven-development/implementer-prompt.md)" -> "Implementer subagent asks questions?";
    "Implementer subagent asks questions?" -> "Answer questions, provide context" [label="yes"];
    "Answer questions, provide context" -> "Dispatch implementer subagent (skills/subagent-driven-development/implementer-prompt.md)";
    "Implementer subagent asks questions?" -> "Implementer subagent implements, tests, commits, self-reviews" [label="no"];
    "Implementer subagent implements, tests, commits, self-reviews" -> "Dispatch spec reviewer subagent (skills/subagent-driven-development/spec-reviewer-prompt.md)";
    "Dispatch spec reviewer subagent (skills/subagent-driven-development/spec-reviewer-prompt.md)" -> "Spec reviewer subagent confirms code matches spec?";
    "Spec reviewer subagent confirms code matches spec?" -> "Implementer subagent fixes spec gaps" [label="no"];
    "Implementer subagent fixes spec gaps" -> "Dispatch spec reviewer subagent (skills/subagent-driven-development/spec-reviewer-prompt.md)" [label="re-review"];
    "Spec reviewer subagent confirms code matches spec?" -> "Dispatch code quality reviewer subagent (skills/subagent-driven-development/code-quality-reviewer-prompt.md)" [label="yes"];
    "Dispatch code quality reviewer subagent (skills/subagent-driven-development/code-quality-reviewer-prompt.md)" -> "Code quality reviewer subagent approves?";
    "Code quality reviewer subagent approves?" -> "Implementer subagent fixes quality issues" [label="no"];
    "Implementer subagent fixes quality issues" -> "Dispatch code quality reviewer subagent (skills/subagent-driven-development/code-quality-reviewer-prompt.md)" [label="re-review"];
    "Code quality reviewer subagent approves?" -> "TaskUpdate: mark task completed" [label="yes"];
    "TaskUpdate: mark task completed" -> "More tasks remain?";
    "More tasks remain?" -> "Dispatch implementer subagent (skills/subagent-driven-development/implementer-prompt.md)" [label="yes"];
    "More tasks remain?" -> "Dispatch final code reviewer subagent for entire implementation" [label="no"];
    "Dispatch final code reviewer subagent for entire implementation" -> "Use claude-superpowers:finishing-a-development-branch";
}
```

## Dispatching with Metadata

When dispatching an implementer subagent:
1. Read the task's description via TaskGet — metadata is embedded as a `json:metadata` code fence at the end
2. Parse the metadata JSON and map fields (files, acceptanceCriteria, verifyCommand) to the implementer prompt sections
3. The implementer should receive ALL structured data — don't make them parse it from prose
4. Detect the task's tech stack and fill the implementer prompt's `## Knowledge Skills` section (see below)

## Stack Detection & Knowledge Skill Selection

Subagents do not auto-discover skills. Empirical data: across 38 real-world dispatches of `claude-superpowers:code-reviewer`, `toolStats.otherToolCount` was 0 in every case — the Skill tool was never invoked. If you want a subagent to apply domain knowledge from a skill, you MUST name the skill explicitly in its dispatch prompt.

Before dispatching each implementer subagent, fill the `## Knowledge Skills` section of `implementer-prompt.md` with the relevant domain skills.

**How to detect the stack:**
1. Inspect the task's `metadata.files` and the working directory's `package.json` (or equivalent for non-JS stacks)
2. Match dependencies and file extensions to available skills:
   - `react` / `react-dom` / `.tsx` / `.jsx` → React knowledge skills
   - `next` → Next.js knowledge skills
   - `expo` / `react-native` / `app.json` / `eas.json` → Expo knowledge skills
   - `typescript` / `tsconfig.json` / `.ts` → TypeScript knowledge skills
   - language-specific: `pyproject.toml` (Python), `Cargo.toml` (Rust), `go.mod` (Go), etc.
3. Look at the available skills list in your environment. Pick the ones whose `description` matches the task's stack and intent.

**What to write into the slot:**
- Up to ~5 skills per dispatch — more than that and the subagent loses focus
- Each line: `- plugin:skill-name — one-line reason this applies to this task`
- Reasons are required. They tell the subagent why each skill is relevant and shape how it applies the guidance.
- If no domain skills apply, write `None — proceed to Task Description.` Do not invent skills.

**Don't:**
- List every available skill — relevance > coverage
- Inject the same skills into every dispatch by default — pick per-task based on the files being touched
- List skills the subagent has already loaded in a previous dispatch — each dispatch is fresh context, but list only what's needed for THIS task
- Reference a skill that isn't installed in the environment

**Do:**
- Reuse the same selection across implementer + spec reviewer + code quality reviewer dispatches for one task — all three need the same domain context
- Treat this as a per-task decision, not a per-plan decision — different tasks in the same plan may touch different stacks

**Example slot fill:**

```
## Knowledge Skills (Invoke FIRST, before anything else)

  - expo:native-data-fetching — Task adds a network request; this skill defines the fetch/React Query patterns we use.
  - expo:building-native-ui — Task creates a new screen component; this skill defines layout and styling conventions.
  - claude-superpowers:test-driven-development — TDD discipline for the test-first workflow this task requires.
```

## Model Selection

Use the least powerful model that can handle each role to conserve cost and increase speed.

**Mechanical implementation tasks** (isolated functions, clear specs, 1-2 files): use a fast, cheap model. Most implementation tasks are mechanical when the plan is well-specified.

**Integration and judgment tasks** (multi-file coordination, pattern matching, debugging): use a standard model.

**Architecture, design, and review tasks**: use the most capable available model.

**Task complexity signals:**
- Touches 1-2 files with a complete spec → cheap model
- Touches multiple files with integration concerns → standard model
- Requires design judgment or broad codebase understanding → most capable model

## Handling Implementer Status

Implementer subagents report one of four statuses. Handle each appropriately:

**DONE:** Proceed to spec compliance review.

**DONE_WITH_CONCERNS:** The implementer completed the work but flagged doubts. Read the concerns before proceeding. If the concerns are about correctness or scope, address them before review. If they're observations (e.g., "this file is getting large"), note them and proceed to review.

**NEEDS_CONTEXT:** The implementer needs information that wasn't provided. Provide the missing context and re-dispatch.

**BLOCKED:** The implementer cannot complete the task. Assess the blocker:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, escalate to the human

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.

## Prompt Templates

- `skills/subagent-driven-development/implementer-prompt.md` - Dispatch implementer subagent
- `skills/subagent-driven-development/spec-reviewer-prompt.md` - Dispatch spec compliance reviewer subagent
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md` - Dispatch code quality reviewer subagent

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/superpowers/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[TaskCreate for each task with full description]

Task 1: Hook installation script

[Get Task 1 text and context (already extracted)]
[Detect stack: bash + node — no domain skill matches; TDD always applies]
[Knowledge Skills slot: claude-superpowers:test-driven-development — TDD discipline for test-first workflow]
[Dispatch implementation subagent with full task text + context + filled Knowledge Skills slot]

Implementer: "Before I begin - should the hook be installed at user or system level?"

You: "User level (~/.config/superpowers/hooks/)"

Implementer: "Got it. Implementing now..."
[Later] Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ✅ Spec compliant - all requirements met, nothing extra

[Get git SHAs, dispatch code quality reviewer]
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.

[Mark Task 1 complete]

Task 2: Recovery modes

[Get Task 2 text and context (already extracted)]
[Dispatch implementation subagent with full task text + context]

Implementer: [No questions, proceeds]
Implementer:
  - Added verify/repair modes
  - 8/8 tests passing
  - Self-review: All good
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ❌ Issues:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)

[Implementer fixes issues]
Implementer: Removed --json flag, added progress reporting

[Spec reviewer reviews again]
Spec reviewer: ✅ Spec compliant now

[Dispatch code quality reviewer]
Code reviewer: Strengths: Solid. Issues (Important): Magic number (100)

[Implementer fixes]
Implementer: Extracted PROGRESS_INTERVAL constant

[Code reviewer reviews again]
Code reviewer: ✅ Approved

[Mark Task 2 complete]

...

[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

Done!
```

## Advantages

**vs. Manual execution:**
- Subagents follow TDD naturally
- Fresh context per task (no confusion)
- Parallel-safe (subagents don't interfere)
- Subagent can ask questions (before AND during work)

**vs. Executing Plans:**
- Same session (no handoff)
- Continuous progress (no waiting)
- Review checkpoints automatic

**Efficiency gains:**
- No file reading overhead (controller provides full text)
- Controller curates exactly what context is needed
- Subagent gets complete information upfront
- Questions surfaced before work begins (not after)

**Quality gates:**
- Self-review catches issues before handoff
- Two-stage review: spec compliance, then code quality
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

**Cost:**
- More subagent invocations (implementer + 2 reviewers per task)
- Controller does more prep work (extracting all tasks upfront)
- Review loops add iterations
- But catches issues early (cheaper than debugging later)

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance (spec reviewer found issues = not done)
- Skip review loops (reviewer found issues = implementer fixes = review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is ✅** (wrong order)
- Move to next task while either review has open issues
- **Dispatch an implementer with the `## Knowledge Skills` section blank or unset.** Either fill it with the relevant skills (with one-line reasons) or explicitly write "None — proceed to Task Description." A blank slot is a bug, not a default.

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

**If reviewer finds issues:**
- Implementer (same subagent) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

**If subagent fails task:**
- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Task Persistence Sync

After marking each task completed via `TaskUpdate`, update the `.tasks.json` file to stay in sync:

1. Read `<plan-path>.tasks.json`
2. Set the task's `"status"` to `"completed"`
3. Set `"lastUpdated"` to current ISO timestamp
4. Write the file back

This ensures cross-session resume works correctly. Without this, a new session loading `.tasks.json` would see completed tasks as `"pending"`.

## Integration

**Required workflow skills:**
- **claude-superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **claude-superpowers:writing-plans** - Creates the plan this skill executes
- **claude-superpowers:requesting-code-review** - Code review template for reviewer subagents
- **claude-superpowers:finishing-a-development-branch** - Complete development after all tasks

**Subagents should use:**
- **claude-superpowers:test-driven-development** - Subagents follow TDD for each task

**Alternative workflow:**
- **claude-superpowers:executing-plans** - Use for parallel session instead of same-session execution
