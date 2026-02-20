# Spec-Kitty Workflow Intelligence

> Lessons learned from the kasmos planning lifecycle (2026-02-12 through 2026-02-20).
> Ported to kas-crew as universally applicable spec-driven development knowledge.

## The Workflow Pipeline

```
specify -> clarify -> plan -> tasks -> analyze -> [implement]
  (req)    (opt)     (req)   (req)    (opt)       (req)
```

Each phase produces artifacts that downstream phases consume. Skipping optional
phases is fine for simple features but costly for complex ones.

## Phase-by-Phase Learnings

### 1. /specify - Discovery & Requirements

**What it produces**: `spec.md` with user stories, functional requirements, NFRs,
edge cases, success criteria.

**Key lessons**:
- Gherkin-style acceptance scenarios make intent unambiguous
- FRs with MUST language give clear implementation targets
- Edge cases section catches integration failures early
- Key Entities in spec should be considered provisional -- the data model is the authority
- Every action/capability mentioned in clarifications should be traced back to an FR

### 2. /clarify - Ambiguity Resolution

**When to use**: When the feature involves multiple systems interacting or UX
decisions with multiple valid answers.

**Key lessons**:
- Each clarification typically prevents 1-2 days of implementation rework
- Skip only for spikes or truly simple features
- Document the skip decision explicitly

### 3. /plan - Architecture & Research

**Key lessons**:
- **Run research before writing architecture decisions (ADs)**. Research may
  invalidate assumptions.
- Always re-read the plan summary after writing the body -- summaries are the
  most-read, least-updated section
- Constitution check is mandatory. If the agent can't find it, search harder, don't skip.

### 4. /tasks - Work Package Decomposition

**Key lessons**:
- Reading actual source files during task generation produces much higher quality
  prompts than working from the plan alone
- Include: current code, target code, reusable patterns, exact import paths
- WP prompt files exist on disk, not in git (by design -- prevents merge conflicts)
- Mark parallel opportunities at both WP and subtask level

### 5. /analyze - Cross-Artifact Consistency

**Key lessons**:
- Catches drift between artifacts that accumulates across phases
- Most valuable finding categories (in order):
  1. Inconsistency (same concept described differently across files)
  2. Coverage gaps (requirement with no task, or task with no requirement)
  3. Underspecification (mentioned but not detailed enough to implement)
- Even well-executed workflows produce 10+ findings. Never skip.

## Workflow Anti-Patterns to Avoid

1. **Writing ADs before research**: Research may invalidate architectural assumptions.
2. **Summarizing from memory, not from the body**: Always re-read before summarizing.
3. **Trusting entity lists across artifacts**: The data model is the authority.
4. **Assuming clarification answers propagate to FRs**: Every new capability needs an FR.
5. **Skipping analyze for "simple" features**: The cost of analyze is low; the cost
   of implementing against inconsistent specs is high.

## Session Architecture Notes

Planning across multiple agent sessions requires handoff summaries that carry:
- Key design decisions and their rationale
- Codebase discoveries (file paths, existing patterns, dependency versions)
- Clarification outcomes
- What was done vs what remains

**Lesson**: Handoff summaries should include **discoveries about the codebase**
(not just decisions). The session that reads source files and discovers the structure
saves the next session from re-reading them.

## WP Lifecycle Protocol

### Rule: Update WP frontmatter lane on completion

When a work package is **completed** (code written, tests passing, committed),
the lane MUST be updated in the WP frontmatter before or as part of the commit.

**Lane values**: `planned` -> `doing` -> `for_review` -> `done`

### Checklist (per WP completion)

1. Verify the WP's code changes build and pass tests
2. Update the WP frontmatter: set `lane: done`
3. Append a history entry with timestamp, lane, actor, and action
4. Include the frontmatter update in the same commit as the code
5. Verify downstream WPs are unblocked (their deps are now all `done`)

### Why this matters

When kas-crew gains spec-kitty integration, it will read the lane field to determine
task state and dependency resolution. Stale lanes will block downstream WPs from
being spawned.

### Anti-pattern: Completing WPs without updating lanes

When WPs are implemented by external agents or manual coding, the lane state
diverges from reality. Always reconcile lane state as part of the completion
workflow, not as an afterthought.
