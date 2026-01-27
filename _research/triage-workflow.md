---
title: Triage Workflow Pattern for AI Agents
problem: Agent doesn't know what to do next or wastes time on decision paralysis
context: Building agent orchestration, task management, multi-step workflows
category: agent-workflows
tags: [agents, workflows, triage, orchestration, task-management]
version: "1.0"
last_updated: 2026-01-27
source: https://github.com/rhea-impact/taskr/blob/main/docs/research/taskr-triage-workflow-case-study.md
---

# Triage Workflow Pattern for AI Agents

> **TL;DR**: Instead of letting agents figure out what to do, give them a structured triage step that returns a concrete workflow. This eliminates decision paralysis and ensures consistent execution.

## Problem Statement

Without triage, agents waste cycles asking:
- "Should I start a session?"
- "What tools should I use?"
- "In what order?"

This creates inconsistent behavior and slow execution.

## Solution: Triage-First Workflow

### The Pattern

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   triage    │────▶│   session   │────▶│   execute   │
│   (plan)    │     │   (start)   │     │  (workflow) │
└─────────────┘     └─────────────┘     └─────────────┘
```

### What Triage Returns

1. **Current State Assessment**: Active sessions, recent work, inferred context
2. **Recommended Workflow**: Ordered phases with specific tools
3. **Actionable Prompt**: Ready-to-execute instructions for the agent

### Example Triage Output

| Phase | Tool | Why |
|-------|------|-----|
| Session Setup | `session_start` | Establishes context, retrieves handoff notes |
| Understand Context | `what_changed`, `search` | See recent activity |
| Check External State | `github_issues`, `project_items` | Don't duplicate work |
| Execute Work | Domain-specific tools | The actual task |
| Document | `devlog_add` | Record decisions immediately |
| End Session | `session_end` | Summarize, leave handoff notes |

## Why This Works

### 1. Eliminates Decision Paralysis
Triage answers "what should I do?" upfront with a concrete workflow. No guessing, no exploration needed.

### 2. Provides Tool-Specific Guidance
Each phase has:
- Specific tool name
- Why to use it
- Example parameters

### 3. Context-Aware Recommendations
Triage notices:
- No active session → "Start one"
- Working on specific repo → Inferred from directory
- Recent files touched → Included in context

### 4. Subagent-Ready Prompts
For complex tasks, triage generates complete prompts that can be handed to subagents, enabling parallelization.

## When to Use Triage

1. **Starting a work session** - Get oriented
2. **Switching repos/projects** - Understand new context
3. **After a break** - Catch up on what changed
4. **Project cleanup** - Reconcile work with tracking

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad |
|--------------|-------------|
| Skipping session start | Loses continuity between runs |
| Not creating devlogs | AI memory gap - future runs won't know what happened |
| Closing issues without comments | No audit trail |
| Not leaving handoff notes | Next session starts cold |

## Implementation

```python
def triage(request: str, working_directory: str, recent_files: list) -> TriageResult:
    """
    Analyze context and return structured workflow.
    
    Returns:
        - current_state: dict with session info, recent activity
        - recommended_workflow: list of phases with tools
        - subagent_prompt: str ready for execution
    """
    # Check for active sessions
    # Infer repo from working directory
    # Analyze recent files for context
    # Generate phase-by-phase workflow
    # Return actionable prompt
```

## Results

In practice, triage-driven workflows complete in **1-2 minutes** what would otherwise take 5-10 minutes of exploration and decision-making.

The key value is **eliminating decision overhead** - the agent doesn't need to figure out what to do, it just executes the workflow.

## References

- [Taskr Triage Case Study](https://github.com/rhea-impact/taskr/blob/main/docs/research/taskr-triage-workflow-case-study.md)
- [MCP Server Pattern](https://modelcontextprotocol.io/)
