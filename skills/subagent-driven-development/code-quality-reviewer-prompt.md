# Code Quality Reviewer Prompt Template

Use this template when dispatching a code quality reviewer subagent.

**Purpose:** Verify implementation is well-built (clean, tested, maintainable)

**Only dispatch after spec compliance review passes.**

```
Task tool (claude-superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  KNOWLEDGE_SKILLS: |
    [SAME slot fill used for the implementer and spec reviewer dispatches.
     Format:
       - plugin-name:skill-name — one-line reason this skill applies
     If no domain-specific skills apply, write "None — proceed to review."]
  WHAT_WAS_IMPLEMENTED: [from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
  DESCRIPTION: [task summary]
```

**Knowledge Skills must match what implementer + spec reviewer received.** A
reviewer without the same domain context cannot verify whether documented
patterns were followed — they will either miss real violations or flag
correct code as wrong. See `claude-superpowers:subagent-driven-development`
"Stack Detection & Knowledge Skill Selection" for selection rules.

**In addition to standard code quality concerns, the reviewer should check:**
- Does each file have one clear responsibility with a well-defined interface?
- Are units decomposed so they can be understood and tested independently?
- Is the implementation following the file structure from the plan?
- Did this implementation create new files that are already large, or significantly grow existing files? (Don't flag pre-existing file sizes — focus on what this change contributed.)

**Code reviewer returns:** Strengths, Issues (Critical/Important/Minor), Assessment
