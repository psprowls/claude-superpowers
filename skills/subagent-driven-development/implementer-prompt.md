# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent.

```
Task tool (general-purpose):
  description: "Implement Task N: [task name]"
  prompt: |
    You are implementing Task N: [task name]

    ## Knowledge Skills (Invoke FIRST, before anything else)

    [Controller fills this list based on detected stack. Format:
      - plugin-name:skill-name — one-line reason this skill applies
      - plugin-name:skill-name — one-line reason this skill applies
     If no domain-specific skills apply, write "None — proceed to Task Description."]

    **Invoke each skill above via the Skill tool BEFORE reading the Task Description.**
    Read each one fully and apply its guidance throughout this task. These skills
    encode stack-specific patterns (React, TypeScript, Expo, etc.) and quality
    standards that you must follow.

    Do not skip this step. Do not defer skill invocation until you hit a problem —
    invoke them upfront so their guidance shapes your approach from the start. If
    you skip this and the reviewer finds you violated a documented pattern, you
    will be sent back to redo the work.

    ## Task Description

    **Goal:** [from task description or metadata]

    **Files:**
    [from task metadata.files or description Files section]

    **Acceptance Criteria:**
    [from task metadata.acceptanceCriteria or description]

    **Verify:** [from task metadata.verifyCommand or description]

    **Steps:**
    [from task description Steps section]

    ## Context

    [Scene-setting: where this fits, dependencies, architectural context]

    ## Before You Begin

    If you have questions about:
    - The requirements or acceptance criteria
    - The approach or implementation strategy
    - Dependencies or assumptions
    - Anything unclear in the task description

    **Ask them now.** Raise any concerns before starting work.

    ## Your Job

    Once you're clear on requirements:
    1. Invoke every skill listed in the Knowledge Skills section above (if any)
    2. Implement exactly what the task specifies, applying the loaded skills' guidance
    3. Write tests (following TDD if task says to)
    4. Verify implementation works
    5. Commit your work
    6. Self-review (see below)
    7. Report back

    Work from: [directory]

    **While you work:** If you encounter something unexpected or unclear, **ask questions**.
    It's always OK to pause and clarify. Don't guess or make assumptions.

    ## Code Organization

    You reason best about code you can hold in context at once, and your edits are more
    reliable when files are focused. Keep this in mind:
    - Follow the file structure defined in the plan
    - Each file should have one clear responsibility with a well-defined interface
    - If a file you're creating is growing beyond the plan's intent, stop and report
      it as DONE_WITH_CONCERNS — don't split files on your own without plan guidance
    - If an existing file you're modifying is already large or tangled, work carefully
      and note it as a concern in your report
    - In existing codebases, follow established patterns. Improve code you're touching
      the way a good developer would, but don't restructure things outside your task.

    ## When You're in Over Your Head

    It is always OK to stop and say "this is too hard for me." Bad work is worse than
    no work. You will not be penalized for escalating.

    **STOP and escalate when:**
    - The task requires architectural decisions with multiple valid approaches
    - You need to understand code beyond what was provided and can't find clarity
    - You feel uncertain about whether your approach is correct
    - The task involves restructuring existing code in ways the plan didn't anticipate
    - You've been reading file after file trying to understand the system without progress

    **How to escalate:** Report back with status BLOCKED or NEEDS_CONTEXT. Describe
    specifically what you're stuck on, what you've tried, and what kind of help you need.
    The controller can provide more context, re-dispatch with a more capable model,
    or break the task into smaller pieces.

    ## Before Reporting Back: Self-Review

    Review your work with fresh eyes. Ask yourself:

    **Completeness:**
    - Did I fully implement everything in the spec?
    - Did I miss any requirements?
    - Are there edge cases I didn't handle?

    **Quality:**
    - Is this my best work?
    - Are names clear and accurate (match what things do, not how they work)?
    - Is the code clean and maintainable?

    **Discipline:**
    - Did I avoid overbuilding (YAGNI)?
    - Did I only build what was requested?
    - Did I follow existing patterns in the codebase?

    **Testing:**
    - Do tests actually verify behavior (not just mock behavior)?
    - Did I follow TDD if required?
    - Are tests comprehensive?

    If you find issues during self-review, fix them now before reporting.

    ## Report Format

    When done, report:
    - **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
    - What you implemented (or what you attempted, if blocked)
    - **Files changed:** [list actual files]
    - **Acceptance criteria status:**
      - [criterion 1]: PASS/FAIL
      - [criterion 2]: PASS/FAIL
    - **Verify command output:** [paste actual output of verify command]
    - What you tested and test results
    - Self-review findings (if any)
    - Any issues or concerns

    Use DONE_WITH_CONCERNS if you completed the work but have doubts about correctness.
    Use BLOCKED if you cannot complete the task. Use NEEDS_CONTEXT if you need
    information that wasn't provided. Never silently produce work you're unsure about.
```
