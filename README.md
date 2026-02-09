# setup-structure-index

A [Claude Code skill](https://code.claude.com/docs/en/skills) that sets up a two-tier codebase structure index for any project.

## What It Does

Installs a system where Claude maintains a **compact file map** in CLAUDE.md (zero-cost, always loaded) and **detailed per-directory YAML files** in `.claude/structure/` (read on demand). A `TaskCompleted` hook ensures the index stays in sync when structural changes happen.

### Why Two Tiers?

Most exploration savings come from knowing *where things are*, not their exact method signatures. The file map in CLAUDE.md answers "where is X?" at zero extra token cost (it's already in the system prompt). Detailed YAML files are only read when you need to understand a specific directory's exports.

**Old approach:** Read ~2000 lines of YAML every session, even for a one-file fix.
**New approach:** Zero extra reads most sessions; ~100-300 lines on demand for deep exploration.

## Installation

Clone into your Claude Code user skills directory:

```bash
# macOS/Linux
git clone https://github.com/shannonbay/setup-structure-index ~/.claude/skills/setup-structure-index

# Windows
git clone https://github.com/shannonbay/setup-structure-index %USERPROFILE%\.claude\skills\setup-structure-index
```

The skill will be available in all projects.

## Usage

In any project, tell Claude:

```
/setup-structure-index
```

Or just ask Claude to "set up structure index" or "add codebase structure tracking".

Claude will:
1. Create `.claude/structure/` directory
2. Generate a compact file map and add it to `CLAUDE.md`
3. Add a `TaskCompleted` hook to `.claude/settings.json`
4. Generate detailed per-directory YAML files using Haiku agents

## What Gets Created

```
your-project/
├── .claude/
│   ├── settings.json              # TaskCompleted hook added
│   └── structure/
│       ├── backend-controllers.yaml  # Detailed exports per directory
│       ├── backend-services.yaml
│       └── app-fragments.yaml
└── CLAUDE.md                      # File map + instruction added
```

## How It Works

**Two tiers + one enforcement point:**

1. **Tier 1 — File map in CLAUDE.md** (always loaded, zero cost): One line per source file with a brief description. Answers "where is X?" instantly.

2. **Tier 2 — Per-directory YAML** (read on demand): Full export signatures, params, return types, and intra-project imports. Read only when working in that area.

3. **TaskCompleted hook** (structural changes only): Blocks completion only when files/classes/functions are added, removed, or renamed. Bug fixes and implementation changes pass through without requiring index updates.

```
New session
  → CLAUDE.md loaded (includes file map) — instant orientation
  → Need to work in controllers/ → read backend-controllers.yaml
  → Add a new endpoint → hook blocks until index updated
  → Fix a bug in existing function → hook passes through
```

## Tier 1 Example (in CLAUDE.md)

```
# Controllers
src/controllers/auth.ts - POST register, login, google, refresh, logout, password reset
src/controllers/users.ts - GET/PATCH/DELETE user profile, avatar upload/download
src/controllers/groups.ts - Group CRUD, members, invites, activity feed, exports

# Services
src/services/authorization.ts - Role checks: requireGroupMember, requireGroupAdmin, etc.
src/services/subscription.service.ts - Tier logic, group limits, downgrade handling
```

## Tier 2 Example (per-directory YAML)

```yaml
module: my-backend
directory: controllers/
files:
  auth.ts:
    description: Authentication endpoints
    exports:
      - name: register
        kind: function
        params: [{name: req, type: AuthRequest}, {name: res, type: Response}]
        description: POST /api/auth/register
    imports:
      - from: services/auth.service
        items: [verifyPassword, createUser]
```

## Cost

- **Tier 1 reading**: 0 extra tokens (embedded in CLAUDE.md)
- **Tier 2 reading**: ~100-300 lines per directory, only when needed
- **Initial generation**: ~50K tokens per module on Haiku (~$0.01-0.03)
- **Hook evaluation**: ~1K tokens per task completion (~$0.0001)
- **Updates**: only on structural changes, not every code edit

## Design Principles

1. **Zero-cost orientation** — file map in CLAUDE.md, always available
2. **Read on demand** — detailed YAML only when exploring unfamiliar areas
3. **Structural changes only** — hook doesn't trigger for bug fixes or body changes
4. **Repo is source of truth** — index is a convenience, not an authority
5. **Split by directory** — never read 1000+ lines to find one function
6. **Minimal infrastructure** — just files, a CLAUDE.md section, and one hook

## License

MIT
