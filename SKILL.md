---
name: setup-structure-index
description: This skill should be used when the user asks to "set up structure index", "add codebase structure tracking", "create structure files", "set up code structure YAML", or needs to set up the codebase structure index system in a project. It creates .claude/structure/ YAML files, adds a CLAUDE.md instruction, and configures a TaskCompleted hook to enforce structure updates.
---

# Codebase Structure Index — Setup Guide

A lightweight system for maintaining a living structural index of your codebase as YAML files. Claude reads these before exploring code (saving exploration rounds) and updates them after modifying code (keeping them in sync).

## What It Is

A `.claude/structure/` directory containing one YAML file per module. Each file describes every source file, its exported functions/classes/types, their signatures, and intra-project imports. Think of it as a flat-file graph of your public interfaces.

This replaces expensive grep-based exploration with a single file read at the start of each session.

## Setup Steps

### 1. Create the structure directory

```
mkdir .claude/structure
```

### 2. Add the CLAUDE.md instruction

Add this section to your project's `CLAUDE.md` (create one if it doesn't exist), ideally near the top before any project structure section:

```markdown
## Codebase Structure Index

Before exploring code, read the relevant `.claude/structure/*.yaml` files for a structural overview of each module. These contain every file, exported function/class, and key relationships (imports, calls). Read these first to orient yourself before using grep/glob to find specific code.

After modifying source code, update the relevant structure file to reflect any added, modified, or removed files, classes, or methods.
```

Also add `.claude/structure/` to any project tree listing in CLAUDE.md:

```
└── .claude/structure/    # Codebase structure index (YAML)
```

### 3. Add the TaskCompleted hook

Add this to `.claude/settings.json` (create if it doesn't exist). If the file already has content, merge the `hooks` key into the existing object:

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "A task is being completed: $ARGUMENTS. Check if the task_subject or task_description suggests source code was modified (new files, classes, methods, refactoring, bug fixes in code). If code was likely changed, respond {\"ok\": false, \"reason\": \"Update .claude/structure/ to reflect any added, modified, or removed files, classes, or methods before completing this task.\"}. If the task is purely research, documentation, discussion, or non-code, respond {\"ok\": true}."
          }
        ]
      }
    ]
  }
}
```

This uses a Haiku prompt to evaluate each task completion. If the task involved code changes, it blocks completion until the structure file is updated. Non-code tasks pass through.

### 4. Generate the initial structure file(s)

For each module/directory in your project, generate a structure YAML file. Use a Haiku agent (Task tool with `model: "haiku"`) for cost efficiency. One agent per module.

**Agent prompt template:**

> Generate a YAML structure file for the [MODULE_NAME] module. This is a codebase structure index that describes files, their exports (functions, classes, types), and key relationships (imports, calls).
>
> **What to do:**
> 1. Explore `[MODULE_PATH]/` to understand the directory structure
> 2. For each source file, identify:
>    - Exported functions/classes/types and their signatures (params + return types)
>    - Key imports (which other project modules it depends on)
> 3. Write the result as a YAML file to `.claude/structure/[module-name].yaml`
>
> **Guidelines:**
> - Only include PUBLIC interfaces (exported items), skip internal/private details
> - Keep method signatures concise — param names and types, return type
> - For imports, only track intra-project dependencies (skip node_modules/external packages)
> - Group by directory
> - Include a brief one-line description for each file's purpose
> - Don't include function bodies or implementation details

For monorepos, create one file per module. For single-module projects, one file is sufficient.

## Structure File Format

There is no rigid schema — the format is self-documenting. Here's a representative example showing the conventions:

```yaml
# module-name structure index
module: module-name
root: module-name/src
description: Brief description of the module

directories:
  - name: controllers/
    description: Route handlers
  - name: services/
    description: Business logic

files:
  controllers/auth.ts:
    description: Authentication route handlers
    exports:
      - name: login
        kind: function
        params: [{name: req, type: Request}, {name: res, type: Response}]
        returns: Promise<void>
        description: POST /api/auth/login - Authenticate with email/password
      - name: register
        kind: function
        params: [{name: req, type: Request}, {name: res, type: Response}]
        returns: Promise<void>
    imports:
      - from: services/auth.service
        items: [authService]
      - from: utils/jwt
        items: [generateAccessToken, verifyToken]

  services/auth.service.ts:
    description: Authentication business logic
    exports:
      - name: verifyPassword
        kind: function
        params: [{name: email, type: string}, {name: password, type: string}]
        returns: Promise<User>
      - name: AuthError
        kind: class
        description: Custom error for auth failures
    imports:
      - from: database/index
        items: [db]

  models/user.ts:
    description: User type definitions
    exports:
      - name: User
        kind: interface
      - name: NewUser
        kind: type
      - name: UserUpdate
        kind: type

# Optional: document key architectural patterns
patterns:
  authentication:
    - Bearer token JWT in Authorization header
    - Access tokens (1h), refresh tokens (30d) hashed in DB
  error_handling:
    - Custom ApplicationError class with status code
    - Global error middleware catches all errors
```

### Format Conventions

- **`kind`**: `function`, `class`, `interface`, `type`, `export` (for re-exports or constants)
- **`params`**: Array of `{name, type}` objects. Omit for simple getters or when obvious.
- **`returns`**: Return type. Omit for void or when obvious.
- **`description`**: One line. Include HTTP method + route for API handlers.
- **`imports`**: Only intra-project dependencies. Group by source module.
- **`patterns`** (optional): Key architectural conventions worth surfacing.

### What to Include

- All exported functions, classes, interfaces, types, constants
- Function signatures (params + return types)
- Intra-project imports
- One-line descriptions

### What to Exclude

- Private/internal functions
- Function bodies or implementation details
- External package imports (express, lodash, etc.)
- Test files (unless they have shared test utilities)
- Config files (tsconfig, package.json, etc.)

## How It Works

The system has two reinforcement points:

1. **Session start (soft)**: CLAUDE.md instruction tells Claude to read `.claude/structure/*.yaml` before exploring code. This is loaded into the system prompt every session.

2. **Task completion (hard)**: The TaskCompleted hook uses a Haiku prompt to evaluate whether the completed task involved code changes. If so, it blocks completion until the structure file is updated.

### Workflow

```
New session
  → Claude reads CLAUDE.md (automatic)
  → CLAUDE.md says "read .claude/structure/*.yaml first"
  → Claude reads structure file(s) — instant codebase orientation
  → Claude works on tasks, modifying code
  → Claude tries to mark task complete
  → TaskCompleted hook fires (Haiku evaluates)
  → If code was changed: blocked until structure updated
  → Claude updates structure file
  → Task completes
```

## Maintenance

- **Structure files are living documents.** They evolve with the codebase.
- **No version numbers needed.** The file itself is the current version.
- **Git-tracked.** Commit structure files alongside code changes so the team benefits.
- **Periodic validation.** If you suspect drift, re-run the Haiku generation prompt and diff against the existing file.
- **Per-module splitting.** For large projects, maintain separate files per module. One file works well up to ~1000 lines of YAML. Beyond that, split.

## Estimated Costs

- **Initial generation** (Haiku agent): ~100K tokens per module, a few cents
- **TaskCompleted hook** (Haiku prompt): ~1K tokens per evaluation, fraction of a cent
- **Reading structure file** (per session): free (just a file read)

## Design Principles

1. **The repo is the source of truth.** Structure files are an index, not an authority. If there's a conflict, the code wins.
2. **Read before explore.** Structure files replace grep-based orientation, not grep-based implementation. Always read actual source before modifying it.
3. **Update after modify.** Keep the index in sync. The hook enforces this.
4. **No schema enforcement.** The format is self-documenting and flexible. Consistency emerges from the initial generation setting the pattern.
5. **Minimal infrastructure.** Just files, a CLAUDE.md instruction, and one hook. No databases, no servers, no build steps.
