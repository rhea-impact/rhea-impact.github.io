---
title: Triage Workflow Pattern for AI Agents
problem: Agent doesn't know what to do next
context: Building agent orchestration, task management, project cleanup
category: agent-workflows
tags: [agents, workflow, triage, task-management, mcp]
version: "1.0"
last_updated: 2026-01-27
---

# Case Study: Using taskr_triage for Project Cleanup

> How the triage workflow enabled rapid project reconciliation in under 2 minutes

## Context

After building the rhea-impact/taskr open-source MCP server (10 commits, ~5000 lines), the project had:
- 8 GitHub issues in the project board
- No devlogs documenting the work
- Unknown state of which issues were actually completed

**Goal**: Clean up the project, close completed work, document everything.

## The Triage Call

```python
taskr_triage(
    request="Clean up and finalize the rhea-impact/taskr open-source rebuild project",
    working_directory="/Users/dshanklinbv/repos/taskr",
    org="rhea-impact",
    repo="taskr",
    recent_files=["docs/research/...", "packages/taskr-mcp/..."]
)
```

## What Triage Returned

### 1. Current State Assessment

```json
{
  "has_active_session": false,
  "recent_devlogs_count": 10,
  "inferred_repo": "rhea-impact/taskr"
}
```

**Insight**: No active session meant I should start one for continuity.

### 2. Recommended Workflow (8 Phases)

| Phase | Tool | Why |
|-------|------|-----|
| Session Setup | `taskr_session_start` | Establishes context, retrieves handoff notes |
| Understand Context | `taskr_what_changed`, `devlog_search` | See recent activity, check for prior work |
| Check GitHub | `mcp__github__search_issues`, `taskr_project_items` | Don't duplicate work |
| Project Cleanup | `skillflow_execute(triage-project-cleanup)` | Reconcile devlogs with issues |
| Select Next Work | `skillflow_execute(work-next)` | Prioritize remaining items |
| Before Coding | `github_create_issue_in_project` or `taskr_claim_work` | Track before starting |
| During Work | `devlog_add` | Record decisions immediately |
| End of Session | `taskr_session_end` | Summarize, leave handoff notes |

### 3. Actionable Subagent Prompt

Triage generated a complete prompt for spawning a subagent:

```
You are helping Claude Code use taskr effectively...
1. Check GitHub state for rhea-impact/taskr
2. Run project cleanup using triage-project-cleanup skillflow
3. Select next work using work-next skillflow
4. Return actionable next steps
```

## Execution (1.5 minutes)

Following the triage workflow:

### Step 1: Start Session
```python
taskr_session_start(
    agent_id="claude-code",
    context="Cleaning up rhea-impact/taskr..."
)
# Returns: session_id, recent_devlogs, handoff_notes from previous session
```

### Step 2: Get Project State
```python
taskr_project_items(org="rhea-impact", project_number=1)
# Returns: 8 items, all status="Todo"
```

### Step 3: Reconcile with Git History
```bash
git log --oneline -20
# Reveals: commit 9501039 "Improve code quality and add linting config"
# This matches Issue #7!
```

### Step 4: Close Completed Work
```python
# Add comment explaining what was done
mcp__github__add_issue_comment(
    owner="rhea-impact", repo="taskr", issue_number=7,
    body="Completed in commit 9501039: ruff.toml, py.typed, mypy config..."
)

# Close the issue
mcp__github__issue_write(
    method="update", owner="rhea-impact", repo="taskr",
    issue_number=7, state="closed", state_reason="completed"
)
```

### Step 5: Document Progress on Open Issues
```python
# Issue #1 (README) - partial progress
mcp__github__add_issue_comment(issue_number=1, body="Progress made: badges, tool list...")

# Issue #8 (hybrid search) - research added
mcp__github__add_issue_comment(issue_number=8, body="Research docs added...")
```

### Step 6: Create Devlog
```python
devlog_add(
    category="feature",
    title="rhea-impact/taskr - Open-source MCP server initial release",
    content="# Summary\nBuilt and released...",
    service_name="taskr",
    tags=["rhea-impact", "open-source", "mcp"]
)
```

### Step 7: End Session
```python
taskr_session_end(
    session_id="4d6b06f5-...",
    summary="Cleaned up project, closed #7, documented work",
    handoff_notes="7 open issues remain. Priority: #2 (tests) or #6 (examples)"
)
```

## Results

| Metric | Value |
|--------|-------|
| Time to complete | 1.5 minutes |
| Issues closed | 1 |
| Issues updated | 2 |
| Devlogs created | 1 |
| Session duration | 1.52 minutes |

## Why Triage Made This Fast

### 1. Eliminated Decision Paralysis

Without triage, the agent would ask:
- "Should I start a session?"
- "What tools should I use?"
- "In what order?"

Triage answered all of these upfront with a concrete workflow.

### 2. Provided Tool-Specific Guidance

Each phase had:
- Specific tool name
- Why to use it
- Example parameters

No guessing, no exploration needed.

### 3. Context-Aware Recommendations

Triage noticed:
- No active session → "Start one"
- Working on rhea-impact/taskr → Inferred from directory
- Recent files touched → Included in context

### 4. Subagent-Ready Prompt

For complex cleanups, triage generates a complete prompt that could be handed to a subagent. This enables parallelization of project triage across multiple repos.

## Pattern: Triage-Driven Project Cleanup

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   triage    │────▶│   session   │────▶│   project   │
│   (plan)    │     │   (start)   │     │   (items)   │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                    ┌─────────────────────────┘
                    ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    git      │────▶│  reconcile  │────▶│   close/    │
│    log      │     │   issues    │     │   comment   │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                    ┌─────────────────────────┘
                    ▼
┌─────────────┐     ┌─────────────┐
│   devlog    │────▶│   session   │
│   (add)     │     │   (end)     │
└─────────────┘     └─────────────┘
```

## Recommendations

### When to Use Triage

1. **Starting a work session** - Get oriented
2. **Switching repos** - Understand new context
3. **After a break** - Catch up on what changed
4. **Project cleanup** - Reconcile work with tracking

### Triage Inputs That Help

| Input | Why |
|-------|-----|
| `request` | Specific task helps triage focus recommendations |
| `working_directory` | Infers repo from path |
| `recent_files` | Shows what you've been working on |
| `org`/`repo` | Explicit GitHub context |

### Anti-Patterns

- **Skipping session start** - Loses continuity
- **Not creating devlogs** - AI memory gap
- **Closing issues without comments** - No audit trail
- **Not leaving handoff notes** - Next session starts cold

---

## Session 2: Continuing the Cleanup

> Demonstrating session continuity with handoff notes

### Context

The next day, after significant work on the README, test coverage, and GitHub integration, the user invoked triage again:

```
"use taskr triage to clean up!!"
```

### Step 1: Triage Orientation

```python
taskr_triage(
    request="Clean up rhea-impact/taskr project",
    working_directory="/Users/dshanklinbv/repos/taskr",
    org="rhea-impact",
    repo="taskr"
)
```

Triage returned:
- **Current state**: No active session
- **Recent devlogs**: 11 entries
- **Recommended workflow**: Same 8-phase pattern

### Step 2: Session Start with Handoff Notes

```python
taskr_session_start(
    agent_id="claude-code",
    context="Cleaning up rhea-impact/taskr after major README/test work"
)
```

**Key result**: Retrieved handoff notes from Session 1:

```
"7 open issues remain. Priority: #2 (tests) or #6 (examples)"
```

This provided immediate context without needing to re-investigate.

### Step 3: GitHub State Check

```python
mcp__github__list_issues(owner="rhea-impact", repo="taskr", state="open")
# Returns: 6 open issues (#1, #2, #4, #5, #6, #8)
```

Note: Down from 8 issues in Session 1. Issue #7 was closed previously.

### Step 4: Reconcile with Today's Work

Reviewed recent activity:
- **README overhaul**: Logo, badges, value proposition, screenshots, learning section
- **Test coverage**: 65+ tests across task, devlog, session, and GitHub tools
- **GitHub integration**: Moved from plugin to core

### Step 5: Close Completed Issues

**Issue #1 (README Overhaul)**:
```python
mcp__github__add_issue_comment(
    owner="rhea-impact", repo="taskr", issue_number=1,
    body="""Completed! The README now includes:
- Logo and badges
- "Why Taskr?" value proposition
- Screenshots showing Taskr building Taskr
- "Taskr Learns Over Time" section with devlog explanation
- Deep Sleep preview (taskr-worker)
- Complete tool reference..."""
)

mcp__github__issue_write(
    method="update", owner="rhea-impact", repo="taskr",
    issue_number=1, state="closed", state_reason="completed"
)
```

**Issue #2 (Test Coverage)**:
```python
mcp__github__add_issue_comment(
    owner="rhea-impact", repo="taskr", issue_number=2,
    body="""Test infrastructure complete with 65+ tests:
- tests/core/test_task_service.py (17 tests)
- tests/core/test_devlog_service.py (16 tests)
- tests/core/test_session_service.py (18 tests)
- tests/mcp/test_github_tools.py (15 mocked tests)..."""
)

mcp__github__issue_write(
    method="update", owner="rhea-impact", repo="taskr",
    issue_number=2, state="closed", state_reason="completed"
)
```

### Step 6: Document Session

```python
devlog_add(
    category="feature",
    title="Session 2 - README polish and test coverage complete",
    content="Continued work on rhea-impact/taskr...",
    service_name="taskr",
    tags=["rhea-impact", "session-2", "cleanup"]
)
```

### Step 7: End Session

```python
taskr_session_end(
    session_id="...",
    summary="Closed #1 (README) and #2 (tests). 4 issues remain.",
    handoff_notes="Remaining: #4 (PyPI), #5 (session tools), #6 (examples), #8 (hybrid search). Priority: #6 examples for user adoption."
)
```

### Results

| Metric | Session 1 | Session 2 |
|--------|-----------|-----------|
| Time to complete | 1.5 min | 1.27 min |
| Issues closed | 1 | 2 |
| Issues updated | 2 | 0 |
| Devlogs created | 1 | 1 |
| Open issues after | 7 | 4 |

### Why Session 2 Was Faster

1. **Handoff notes eliminated investigation**
   - Session 1 ended with "Priority: #2 (tests) or #6 (examples)"
   - Session 2 knew immediately what was important

2. **Pattern was already established**
   - Same 8-phase workflow
   - No decision paralysis

3. **Incremental progress**
   - Each session closes a few issues
   - Project gets cleaner over time

---

## Conclusion

`taskr_triage` transformed a vague "clean up this project" request into a concrete 8-phase workflow that completed in 1.5 minutes. The key value is **eliminating decision overhead** - the agent doesn't need to figure out what to do, it just executes the workflow.

### Session Continuity Pattern

```
Session 1                    Session 2                    Session N
┌─────────────┐              ┌─────────────┐              ┌─────────────┐
│   Work      │──handoff──▶  │   Work      │──handoff──▶  │   Work      │
│   8→7 open  │    notes     │   6→4 open  │    notes     │   ...       │
└─────────────┘              └─────────────┘              └─────────────┘
```

Each session:
- Retrieves previous handoff notes
- Picks up where the last session ended
- Leaves notes for the next session

This creates **persistent memory across context windows**.

### Key Takeaways

For projects with many issues, the triage skillflow (`triage-project-cleanup`) automates the reconciliation between devlogs and GitHub issues, finding:
- Issues that can be closed (work already done)
- Stale issues with no activity
- Untracked work that needs issues

This case study demonstrates that structured workflows (via triage) enable rapid, consistent project management that would otherwise require significant cognitive overhead. The session continuity pattern ensures no work is lost between conversations.
