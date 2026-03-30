---
name: intent
description: Manage tasks using the Intent CLI (nt). Use this skill when the user asks to create, list, update, delete, or organize tasks, or when you need to track work items in a project. Intent is a fast terminal-based task manager that connects to Strayl workspaces.
---

# Intent CLI — Task Management

Intent (`nt`) is a CLI for managing tasks in Strayl workspaces. Use it to track work, assign tasks, and organize projects from the terminal.

## Prerequisites

- Node.js 20+
- A Strayl account (sign up at https://app.strayl.dev)
- Install: `npm install -g @strayl/intent`

## Setup

```bash
nt login        # Authenticate via browser (device OAuth flow)
nt init          # Create or select a workspace (saves to .intent/config.json)
nt whoami        # Verify current user and workspace
```

## Commands

### Create a task

```bash
nt create "Title" [options]
```

Options:
- `-p, --priority <level>` — urgent, high, medium, low, none
- `-s, --status <status>` — backlog, todo, in_progress, done, cancelled
- `-l, --label <label>` — add label (repeatable for multiple)
- `-d, --description <text>` — task description
- `--parent <ref>` — parent task number (for subtasks)
- `--assignee <id>` — assign to user
- `--agent <name>` — assign to AI agent

Example:
```bash
nt create "Fix login bug" -p high -l bug -l auth -d "Login fails after OAuth refactor"
# ✓ Created task #1: Fix login bug
```

### List tasks

```bash
nt ls [options]
```

Filters:
- `--status <s>` — filter by status (comma-separated)
- `--priority <p>` — filter by priority
- `--label <l>` — filter by label
- `--assignee <id>` — filter by assignee
- `--parent <ref>` — filter by parent task
- `-q <text>` — search in title and description
- `--sort <field>` — number, priority, status, created, updated, due
- `--order <dir>` — asc, desc
- `--limit <n>` — max results (1-200)

Example:
```bash
nt ls --status todo,in_progress -p high
```

### Show task details

```bash
nt show <ref>
```

`<ref>` is the task number (e.g., `3`).

### Update a task

```bash
nt update <ref> [options]
```

Same options as create, plus:
- `--title <string>` — change title
- `--due <date>` — set due date

Example:
```bash
nt update 1 --status in_progress
# ✓ Updated #1: Fix login bug [in_progress]
```

### Mark as done

```bash
nt done <ref>
# Shortcut for: nt update <ref> --status done
```

### Delete a task

```bash
nt rm <ref>
# or: nt delete <ref>
```

### Bulk operations

```bash
nt bulk [file]        # from JSON file
cat ops.json | nt bulk  # from stdin
```

JSON format — array of operations:
```json
[
  { "op": "create", "data": { "title": "Task A", "priority": "high" } },
  { "op": "update", "ref": "1", "data": { "status": "done" } },
  { "op": "delete", "ref": "2" }
]
```

Max 100 operations per batch.

## JSON Mode

Add `--json` to any command for machine-readable output:

```bash
nt ls --json              # returns { tasks: [...], hasMore, nextCursor }
nt create "Task" --json   # returns full task object
nt show 1 --json          # returns task with subtasks
nt bulk ops.json --json   # returns { results: [...] }
```

Always use `--json` when parsing output programmatically.

## Status & Priority Reference

**Status values:** backlog, todo, in_progress, done, cancelled

**Priority values:** urgent, high, medium, low, none

## Task Fields

| Field | Type | Description |
|-------|------|-------------|
| number | int | Auto-incremented per workspace |
| title | string | Task title |
| description | string? | Detailed description |
| status | string | Current status |
| priority | string | Priority level |
| labels | string[] | Tags/categories |
| assigneeId | string? | Assigned user |
| assigneeAgent | string? | Assigned AI agent name |
| parentTaskId | string? | Parent task (for subtasks) |
| dependsOn | string[] | Task dependencies |
| dueDate | string? | ISO date |
| metadata | string? | Free-form JSON |

## Best Practices for AI Agents

1. **Always use `--json`** for parsing output reliably
2. **Use bulk operations** when creating/updating multiple tasks at once
3. **Check `nt whoami --json`** first to verify workspace is configured
4. **Use labels** to categorize work (e.g., `bug`, `feature`, `docs`)
5. **Use `--agent`** flag when creating tasks assigned to yourself
6. **Use subtasks** (`--parent`) to break down complex work
7. **Search with `-q`** before creating to avoid duplicates
