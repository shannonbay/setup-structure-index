---
name: setup-structure-index
description: This skill should be used when the user asks to "set up structure index", "add codebase structure tracking", "create structure files", "set up code structure YAML", or needs to set up the codebase structure index system in a project. It creates a two-tier structure index (compact file map in CLAUDE.md + detailed YAML files) and configures a TaskCompleted hook to enforce updates.
---

# Codebase Structure Index — Setup Guide

A two-tier system for maintaining a living structural index of your codebase. A compact file map in CLAUDE.md provides instant zero-cost orientation every session, while detailed per-directory YAML files are read on demand when deeper exploration is needed.

## Why Two Tiers?

Most of the exploration savings come from knowing *where things are*, not their exact method signatures. A compact file map (~100-200 lines) embedded in CLAUDE.md captures ~70% of the navigation value at zero extra token cost (CLAUDE.md is already loaded into the system prompt). Detailed YAML files with full signatures are only needed when working in an unfamiliar area of the codebase.

**Token economics of the old single-tier approach:**
- Reading a full structure YAML (~1000-2000 lines per module) every session
- Most sessions only touch 1-2 directories, wasting tokens on irrelevant detail
- Method signatures are low value — you still need to read source before editing

**Token economics of the two-tier approach:**
- Tier 1 (file map in CLAUDE.md): 0 extra tokens (already in system prompt)
- Tier 2 (detailed YAML): read only when needed, split by directory for targeted reads
- Net result: most sessions read 0 extra lines; deep exploration reads ~100-300 lines

## What It Creates

### Tier 1: Compact File Map (in CLAUDE.md)

A concise listing of every source file with a one-line description, embedded directly in CLAUDE.md. This answers "where is X?" — the most common exploration question. Since CLAUDE.md is automatically loaded into the system prompt, this costs zero additional tokens.

### Tier 2: Detailed Structure Files (in .claude/structure/)

Per-directory YAML files with full export signatures, method params/returns, and intra-project imports. These answer "what does X export?" and are only read when working in that area. Split by directory so you never read more than you need.

### TaskCompleted Hook

A hook that evaluates whether a completed task added, removed, or renamed files/classes. Only triggers updates for structural changes, not every code edit.

## Setup Steps

### 1. Create the structure directory

```
mkdir .claude/structure
```

### 2. Add the file map and instruction to CLAUDE.md

Add these sections to your project's `CLAUDE.md` (create one if it doesn't exist). The file map goes in the project structure area; the instruction goes near the top.

**Instruction section** (add near the top):

```markdown
## Codebase Structure Index

The file map below provides instant orientation. For detailed export signatures and dependencies, read the relevant `.claude/structure/*.yaml` file for the directory you're working in.

After adding, removing, or renaming source files or public classes/functions, update both the file map below and the relevant structure YAML file.
```

**File map section** (add after the instruction, or integrate into an existing project structure section):

```markdown
### File Map

<!-- One line per source file: relative path - brief description -->
src/controllers/auth.ts - Authentication endpoints (register, login, OAuth, refresh)
src/controllers/users.ts - User profile, avatar, subscription endpoints
src/services/auth.service.ts - Auth business logic (password verification, token generation)
src/services/subscription.service.ts - Tier logic, group limits, downgrade handling
src/middleware/authenticate.ts - JWT bearer token verification
src/middleware/errorHandler.ts - Global error handler (ApplicationError, Zod, Multer)
src/database/types.ts - Kysely table interfaces and type utilities
src/utils/jwt.ts - JWT generation and verification
...
```

Also add `.claude/structure/` to any project tree listing:

```
└── .claude/structure/    # Detailed structure index (per-directory YAML)
```

**Guidelines for the file map:**
- One line per source file: `relative/path.ext - Brief description`
- Keep descriptions to ~10 words. For API handlers, include HTTP methods/routes.
- Group by directory with blank lines between groups
- Skip test files, config files, and generated files
- Target ~100-200 lines total. If the project is larger, group minor utility files.

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
            "prompt": "A task is being completed: $ARGUMENTS. Check if the task involved adding, removing, or renaming source files, public classes, or public functions. If so, respond {\"ok\": false, \"reason\": \"Update the CLAUDE.md file map and .claude/structure/ YAML to reflect added, removed, or renamed files/classes/functions.\"}. If the task only modified existing function bodies, fixed bugs without structural changes, or was non-code work, respond {\"ok\": true}."
          }
        ]
      }
    ]
  }
}
```

This hook is more selective than a blanket "did code change?" check. It only blocks for **structural changes** (new/removed/renamed files, classes, functions), not for every bug fix or implementation change. This avoids wasting tokens updating the index when a one-line fix doesn't change the structure.

### 4. Generate the initial file map for CLAUDE.md

Use a Haiku agent (Task tool with `model: "haiku"`) to scan the source tree and produce the compact file map.

**Agent prompt template:**

> Generate a compact file map for the [MODULE_NAME] module to embed in CLAUDE.md.
>
> **What to do:**
> 1. List every source file in `[MODULE_PATH]/`
> 2. For each file, write one line: `relative/path.ext - Brief description (~10 words)`
> 3. For API handlers, include HTTP method + route in the description
> 4. Group by directory with blank lines between groups
> 5. Skip test files, config files, and generated files
> 6. Return the result as plain text (not YAML) ready to paste into CLAUDE.md
>
> **Target:** ~100-200 lines total. Be concise.

After the agent returns the file map, add it to CLAUDE.md under the instruction section.

### 5. Generate the initial detailed structure files

For each major directory (not module — directory), generate a focused YAML file. Use a Haiku agent per directory. Split along natural boundaries: `controllers/`, `services/`, `ui/fragments/`, etc.

**Agent prompt template:**

> Generate a detailed structure YAML file for the `[DIRECTORY]` directory of the [MODULE_NAME] module.
>
> **What to do:**
> 1. For each source file in `[MODULE_PATH]/[DIRECTORY]/`, identify:
>    - Public classes/functions/types and their signatures (params + return types)
>    - Key intra-project imports (skip external packages)
> 2. Write the result as YAML to `.claude/structure/[module]-[directory].yaml`
>
> **Guidelines:**
> - Only include PUBLIC interfaces, skip private/internal
> - Keep signatures concise — param names and types, return type
> - Include a one-line description for each file and each export
> - For API handlers, include HTTP method + route
> - Don't include function bodies or implementation details
> - For obvious signatures (like Android lifecycle methods), omit params
> - Skip imports that are obvious from context

**Naming convention:** `[module]-[directory].yaml`, e.g.:
- `backend-controllers.yaml`
- `backend-services.yaml`
- `app-fragments.yaml`
- `app-network.yaml`

For small directories (< 5 files), combine related ones into a single file.

## Tier 1: File Map Format

Plain text, one line per file. Designed for minimal token footprint while answering "where is X?"

```
# Controllers
src/controllers/auth.ts - POST register, login, google, refresh, logout, password reset
src/controllers/users.ts - GET/PATCH/DELETE user profile, avatar upload/download
src/controllers/groups.ts - Group CRUD, members, invites, activity feed, exports

# Services
src/services/authorization.ts - Role checks: requireGroupMember, requireGroupAdmin, etc.
src/services/subscription.service.ts - Tier logic, group limits, downgrade handling
src/services/fcm.service.ts - Firebase push notifications and topic messaging

# Middleware
src/middleware/authenticate.ts - JWT bearer token verification
src/middleware/errorHandler.ts - ApplicationError class + global error handler
src/middleware/rateLimiter.ts - Rate limiters for API, auth, password reset
```

## Tier 2: Detailed YAML Format

Self-documenting, per-directory. Only read when working in that area.

```yaml
# backend-controllers structure detail
module: my-backend
directory: controllers/
description: Express route handlers

files:
  auth.ts:
    description: Authentication endpoints
    exports:
      - name: register
        kind: function
        params: [{name: req, type: AuthRequest}, {name: res, type: Response}]
        returns: Promise<void>
        description: POST /api/auth/register - Create user with email/password
      - name: login
        kind: function
        description: POST /api/auth/login - Authenticate with email/password
      - name: googleAuth
        kind: function
        description: POST /api/auth/google - Sign in/register with Google OAuth
    imports:
      - from: services/auth.service
        items: [verifyPassword, createUser]
      - from: utils/jwt
        items: [generateAccessToken, generateRefreshToken]
```

### Format Conventions

- **`kind`**: `function`, `class`, `interface`, `type`, `object`, `export`
- **`params`**: Array of `{name, type}`. Omit for obvious signatures (lifecycle methods, simple getters).
- **`returns`**: Return type. Omit for void or when obvious.
- **`description`**: One line. Include HTTP method + route for API handlers.
- **`imports`**: Only intra-project dependencies. Skip external packages.

### What to Include

- All public functions, classes, interfaces, types, constants
- Method signatures (params + return types) for non-obvious interfaces
- Intra-project imports
- One-line descriptions

### What to Exclude

- Private/internal functions and properties
- Function bodies or implementation details
- External package imports
- Obvious lifecycle method signatures (onCreate, onViewCreated, etc.)
- Parameter types when obvious from name (e.g., `context: Context`)
- Test files, config files

## How It Works

The system has two tiers and one enforcement point:

1. **Tier 1 — File map in CLAUDE.md (always loaded, zero cost)**: Answers "where is X?" without any file reads. Loaded automatically in the system prompt every session.

2. **Tier 2 — Per-directory YAML (loaded on demand)**: Answers "what does X export?" when you need to understand a specific area. Read only the relevant directory file, not the entire codebase.

3. **TaskCompleted hook (structural changes only)**: Blocks task completion when files/classes/functions are added, removed, or renamed. Does NOT block for bug fixes or implementation changes within existing functions.

### Workflow

```
New session
  → Claude reads CLAUDE.md (automatic, includes file map)
  → Instant orientation: knows where everything is
  → Needs to work in controllers/ → reads .claude/structure/backend-controllers.yaml
  → Works on tasks, modifies code
  → Adds a new controller function
  → Tries to mark task complete
  → Hook fires: "were files/classes/functions added/removed/renamed?"
  → Yes: blocked until file map + structure YAML updated
  → Updates both, task completes

  → Next task: fix bug in existing service function
  → Tries to mark task complete
  → Hook fires: only a body change, no structural change
  → Passes through, no update needed
```

## Maintenance

- **File map is the primary index.** Keep it accurate — it's what gets read every session.
- **Structure YAMLs are secondary detail.** Update when structural changes happen, but don't obsess over keeping method signatures perfectly current.
- **Git-tracked.** Commit both the file map (in CLAUDE.md) and structure files alongside code changes.
- **Periodic validation.** If you suspect drift, re-run the Haiku generation and diff.
- **Split threshold.** A single YAML file works well up to ~300 lines. Beyond that, split further by subdirectory.

## Estimated Costs

- **Tier 1 reading** (per session): 0 extra tokens (embedded in CLAUDE.md, already loaded)
- **Tier 2 reading** (per deep exploration): ~100-300 lines per directory file
- **Initial generation** (Haiku agents): ~50K tokens per module, ~$0.01-0.03
- **Hook evaluation**: ~1K tokens per task completion, fraction of a cent
- **Structure updates**: only on structural changes, not every code edit

## Design Principles

1. **Zero-cost orientation.** The file map in CLAUDE.md means every session starts with full codebase awareness at no extra token cost.
2. **Read on demand, not up front.** Detailed YAML files are only read when working in that area. Most sessions read 0-1 detail files.
3. **Structural changes only.** The hook and update obligation only apply when the codebase structure changes (new/removed/renamed items), not for every line of code.
4. **The repo is the source of truth.** Structure files are an index, not an authority. If there's a conflict, the code wins.
5. **Split by directory, not by module.** Keeps individual reads small and targeted. Never read 1000+ lines to find one function.
6. **Minimal infrastructure.** Files, a CLAUDE.md section, and one hook. No databases, no servers, no build steps.
