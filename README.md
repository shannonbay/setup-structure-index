# setup-structure-index

A [Claude Code skill](https://code.claude.com/docs/en/skills) that sets up a lightweight codebase structure index for any project.

## What It Does

Installs a system where Claude maintains `.claude/structure/*.yaml` files — living YAML documents that describe every source file, exported function/class/type, and intra-project dependency in your codebase.

This gives Claude instant structural orientation at the start of each session (one file read vs. dozens of grep searches), and a `TaskCompleted` hook ensures the index stays in sync with code changes.

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
2. Add a "read structure first" instruction to `CLAUDE.md`
3. Add a `TaskCompleted` hook to `.claude/settings.json`
4. Generate initial structure YAML file(s) using a Haiku agent for cost efficiency

## What Gets Created

```
your-project/
├── .claude/
│   ├── settings.json          # TaskCompleted hook added
│   └── structure/
│       └── your-module.yaml   # Structure index
└── CLAUDE.md                  # Instruction added
```

## How It Works

**Two reinforcement points keep the index accurate:**

1. **Session start (soft)** — `CLAUDE.md` instruction tells Claude to read `.claude/structure/*.yaml` before exploring code
2. **Task completion (hard)** — A `TaskCompleted` hook uses a Haiku prompt to evaluate whether completed tasks involved code changes; if so, it blocks completion until the structure file is updated

```
New session
  → Claude reads CLAUDE.md (automatic)
  → Reads structure file(s) — instant codebase orientation
  → Works on tasks, modifies code
  → Tries to mark task complete
  → Hook fires: "did this task change code?"
  → If yes: blocked until structure updated
  → Updates structure, task completes
```

## Structure File Format

No rigid schema — format is self-documenting. Example:

```yaml
module: my-backend
root: src
description: Express.js API server

files:
  controllers/auth.ts:
    description: Authentication route handlers
    exports:
      - name: login
        kind: function
        params: [{name: req, type: Request}, {name: res, type: Response}]
        returns: Promise<void>
    imports:
      - from: services/auth.service
        items: [authService]

  services/auth.service.ts:
    description: Auth business logic
    exports:
      - name: verifyPassword
        kind: function
        params: [{name: email, type: string}, {name: password, type: string}]
        returns: Promise<User>
```

## Cost

- **Initial generation**: ~100K tokens per module on Haiku (~$0.03)
- **Hook evaluation**: ~1K tokens per task completion (~$0.0001)
- **Reading structure**: free (file read)

## Design Principles

1. **Repo is source of truth** — structure files are an index, not an authority
2. **Read before explore** — replaces grep-based orientation, not implementation
3. **Update after modify** — hook enforces keeping the index in sync
4. **No schema enforcement** — consistency emerges from the initial generation
5. **Minimal infrastructure** — just files, a CLAUDE.md line, and one hook

## License

MIT
