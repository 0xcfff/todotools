# CLI API Specification (Implementation Planning)

## Overview and Design Inputs

This document is an implementation-planning spec for the root CLI of the todotools project. It defines the command surface and sample scenarios only; no code is specified here.

### Purpose

- Define the unified root CLI (`todo`) with subcommands for listing, diff, patch, merge, and config.
- Document a comprehensive sample command set and scenarios for implementers.
- Align with familiar tool conventions so users can transfer mental models from existing tools.

### Design References

- **GNU diff/patch**: Unified diff format; two-file comparison (`diff -u from to`); patch application (`patch -i file` or stdin); strip level (e.g. `patch -p0`) for path prefixes; merge workflows.
- **TODO.sh** (todo.txt-cli): Action-based CLI (`todo.sh action [args]`); `list` with TERM filters; project/context notation (`+project`, `@context`); exclusion via `-TERM`; `listpri`, `listcon`, `listproj`; config via `-d CONFIG_FILE`; global options (force, verbose, plain/color).

### Scope

- **In scope**: Root CLI only; subcommands `list`, `diff`, `patch`, `merge`, `config`.
- **Out of scope**: REST/OpenAPI or other API design; implementation details (libraries, packages); changes to [TODO_DIFF_SPEC.md](../TODO_DIFF_SPEC.md). Diff output format is defined there; this spec covers CLI invocation only.

### Existing Context

- Diff format: [specs/TODO_DIFF_SPEC.md](../TODO_DIFF_SPEC.md) defines the semantic todo-diff format (unified-diff style, no context lines, task matching).
- Current executables: `cmd/todo`, `cmd/tododiff`, `cmd/todopatch` (stubs). This spec assumes a single `todo` binary with subcommands.

---

## Root CLI Syntax

**General form:**

```text
todo [global-opts] subcommand [subcommand-args]
```

**Global options (candidates, aligned with TODO.sh where relevant):**

| Option | Purpose |
|--------|--------|
| `-d FILE`, `--config FILE` | Use configuration file (e.g. overrides default `~/.config/todotools/config`) |
| `--config-dir DIR` | Use directory for config lookup |
| `-f`, `--file FILE` | Override default todo file path for commands that use it |
| `-f`, `--force` | No confirmation / non-interactive (when used for destructive actions) |
| `-v`, `--verbose` | Verbose output |
| `-q`, `--quiet` | Minimal output |
| `-p`, `--plain` | Disable color; `--no-color` alternative |
| `-h`, `--help` | Help for root or subcommand |
| `-V`, `--version` | Version and credits |

Note: `-f` may be used for either "file" or "force" depending on subcommand; exact semantics TBD per subcommand to avoid collision.

**Subcommand aliases:** Some subcommands have short aliases (e.g. `list` can be invoked as `ls`). Samples in this spec use the canonical name; aliases accept the same arguments and behave identically.

---

## Subcommand: list

**Purpose:** List tasks from a todo.txt file with optional filtering. Sorted by priority with line numbers (compatible with TODO.sh `list` behavior). **Alias:** `ls` — `todo ls` is equivalent to `todo list`.

### Argument types

- **Positional TERM(s):** Zero or more terms. Each term can be a word, `+project`, `@context`, or `-TERM` (exclusion). Multiple positive terms = AND; optional OR via `TERM1|TERM2` (document exact syntax for implementers).
- **File:** Default from config or the platform data path (see [Default todo file](#default-todo-file)); override with global `-f`/`--file` or local `--file path/to/todo.txt`.
- **Filters:** Optional priority range, explicit project/context, exclude patterns.

### Sample commands and scenarios

```text
todo list
```

List all tasks from the default todo file.

```text
todo list apple
```

List tasks containing the term "apple".

```text
todo list +prj
```

List tasks in project `+prj` (tasks that contain `+prj`).

```text
todo list @ctx
```

List tasks in context `@ctx` (tasks that contain `@ctx`).

```text
todo list -done
```

Exclude tasks containing "done" (e.g. avoid completed-task text in description).

```text
todo list TERM1 TERM2
```

List tasks that contain both TERM1 and TERM2 (AND).

```text
todo list "TERM1|TERM2"
```

List tasks that contain TERM1 or TERM2 (OR); exact escaping TBD.

```text
todo list --file path/to/todo.txt
todo list -f path/to/todo.txt
```

List from an explicit file path.

```text
todo list --done
todo listall
```

Include done tasks (e.g. from done.txt or same file with completed items). Scenario: "listall"-style; behavior TBD (separate done.txt vs single file with `x` lines).

```text
todo listpri
todo listpri A
todo listpri A-C
```

List by priority: all prioritized tasks; single priority (A); or range (A–C).

```text
todo listcon
todo listcon TERM
```

List all context names (@...) in the file; optionally restricted to tasks containing TERM.

```text
todo listproj
todo listproj TERM
```

List all project names (+...) in the file; optionally restricted to tasks containing TERM.

---

## Subcommand: diff

**Purpose:** Produce a todo-diff (semantic unified diff) between two todo files. Output format follows [TODO_DIFF_SPEC.md](../TODO_DIFF_SPEC.md). Aligns with GNU `diff -u from-file to-file`.

### Argument types

- **FROM / TO paths:** Positional or from config. Order: FROM (original) then TO (modified), so the patch would transform FROM into TO.
- **Output:** Stdout by default; optional `-o FILE` or `--output FILE` to write patch to a file.

### Sample commands and scenarios

```text
todo diff todo.txt todo_changed.txt
```

Produce semantic unified diff from `todo.txt` (original) to `todo_changed.txt` (modified). Output to stdout.

```text
todo diff from.txt to.txt -o changes.patch
todo diff from.txt to.txt --output changes.patch
```

Same as above but write patch to `changes.patch`.

```text
todo diff
```

Use FROM/TO from config or defaults (e.g. default todo file vs backup path, or two configured paths). Exact default behavior TBD.

```text
todo diff -u from.txt to.txt
```

Unified output (default); `-u` may be accepted for compatibility with GNU diff habits.

```text
todo diff --no-semantic from.txt to.txt
todo diff --line-based from.txt to.txt
```

Optional: fall back to line-based diff (standard unified diff, no task matching). Document as scenario for implementers; may be useful for debugging or compatibility.

---

## Subcommand: patch

**Purpose:** Apply a todo-diff patch to a todo file. Analogous to GNU `patch` applying a unified diff. Patch format is as in TODO_DIFF_SPEC; application is semantic (match tasks, apply deltas) where possible.

### Argument types

- **Patch input:** File via `-i FILE` / `--input FILE`, or stdin when no `-i` and no patch file positional.
- **Target file:** Positional path to the todo file to patch, or default from config.
- **Strip level:** Optional `-p N` / `--strip N` to strip N path components from filenames in the patch (GNU patch style).

### Sample commands and scenarios

```text
todo patch -i changes.patch todo.txt
```

Apply `changes.patch` to `todo.txt`. Target file is explicit.

```text
todo patch todo.txt < changes.patch
```

Apply patch from stdin to `todo.txt`.

```text
todo patch -i changes.patch
```

Apply patch to the default todo file (from config or environment).

```text
todo patch --dry-run -i changes.patch todo.txt
todo patch -n -i changes.patch todo.txt
```

Preview: report what would be applied without modifying the file.

```text
todo patch -i changes.patch -p1 todo.txt
```

Apply patch, stripping one path component from paths in the patch (e.g. when patch was created from a subdirectory).

---

## Subcommand: merge

**Purpose:** Merge two or three todo files. Three-way merge: base + ours + theirs → merged result. Two-way: combine two files with a defined strategy (e.g. union vs conflict markers).

### Argument types

- **Three-way:** `base`, `ours`, `theirs` (positional or via options). Output: stdout or `-o FILE`.
- **Two-way:** `ours` and `theirs` (positional). Behavior TBD: union of tasks vs conflict markers for conflicting edits.
- **Conflict handling:** Strategy: abort on conflict, emit conflict markers, or automatic resolution (e.g. prefer ours/theirs). Document for implementers.

### Sample commands and scenarios

```text
todo merge base.txt ours.txt theirs.txt
```

Three-way merge: base, our version, their version; output merged result to stdout.

```text
todo merge -o merged.txt base.txt ours.txt theirs.txt
todo merge base.txt ours.txt theirs.txt -o merged.txt
```

Three-way merge; write result to `merged.txt`.

```text
todo merge ours.txt theirs.txt
```

Two-way merge. Scenario: e.g. union of tasks, or conflict markers where the same task was edited differently; exact behavior TBD.

```text
todo merge --abort-on-conflict base.txt ours.txt theirs.txt
```

Three-way merge; exit with error on conflict (no in-file conflict markers).

```text
todo merge --markers -o merged.txt base.txt ours.txt theirs.txt
```

On conflict, write conflict markers into `merged.txt` for manual resolution.

---

## Subcommand: config

**Purpose:** Show, list, or set configuration (e.g. default todo file path, archive path, behavior flags). Config file location: e.g. `~/.config/todotools/config` or path given by global `-d`/`--config`.

### Argument types

- **Show:** `todo config` — print resolved config (all keys and values in effect).
- **List keys:** `todo config --list` — list config key names.
- **Get:** `todo config get KEY` — print value for KEY.
- **Set:** `todo config set KEY VALUE` or `todo config KEY VALUE` — set KEY to VALUE (and persist to config file if applicable).

### Sample commands and scenarios

```text
todo config
```

Show current config (resolved from file + defaults + environment).

```text
todo config --list
```

List all config keys.

```text
todo config get todo_file
```

Print the configured default todo file path.

```text
todo config set todo_file /path/to/todo.txt
todo config todo_file /path/to/todo.txt
```

Set default todo file path to `/path/to/todo.txt`.

```text
todo config set archive_file /path/to/done.txt
```

Set path for archive/done file (if supported).

```text
todo -d /etc/todotools/config config
```

Show config using an alternate config file (global `-d`).

### Config keys (candidates)

| Key | Purpose |
|-----|--------|
| `todo_file` | Path to todo.txt. If unset, use platform data path (see [Default todo file](#default-todo-file)). |
| `archive_file` | Default done.txt path (optional) |

### Config file location (per OS)

Config file path is resolved in this order: explicit path from global `-d` / `--config`, then the platform default below. The same path is used for reading and writing config (e.g. `todo config set`).

- **Linux:** Follow the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html). Config file: `$XDG_CONFIG_HOME/todotools/config`. If `XDG_CONFIG_HOME` is unset or empty, use `~/.config/todotools/config`. This keeps user config out of the home root and respects overrides (e.g. for containers or custom layouts).

- **macOS:** Use the same path as on Linux: `$XDG_CONFIG_HOME/todotools/config` with fallback to `~/.config/todotools/config`. Many CLI tools use the same XDG-style location on both Linux and macOS for consistency and simpler cross-platform behavior.

- **Windows:** Use `%APPDATA%\todotools\config` (e.g. `C:\Users\<user>\AppData\Roaming\todotools\config`). `APPDATA` (Roaming) is the standard per-user location for application config and data. It’s the Windows analogue of XDG config (user-specific, roams with the profile) and is where both CLI and GUI apps typically store config.

---

## Scenario Summary Tables

### list

| Scenario | Command sample | Arguments | Expected behavior |
|----------|-----------------|-----------|--------------------|
| All tasks | `todo list` | (none) | List all tasks from default file, sorted by priority, with line numbers |
| Filter by term | `todo list apple` | TERM | List tasks containing "apple" |
| Filter by project | `todo list +prj` | +project | List tasks in project +prj |
| Filter by context | `todo list @ctx` | @context | List tasks in context @ctx |
| Exclude term | `todo list -done` | -TERM | List tasks not containing "done" |
| AND terms | `todo list TERM1 TERM2` | TERM TERM | List tasks containing both |
| Explicit file | `todo list --file path/to/todo.txt` | --file PATH | List from given file |
| Include done | `todo list --done` | --done | Include completed tasks (listall-style) |
| By priority | `todo listpri A-C` | [PRIORITIES] | List tasks with priority in range A–C |
| List contexts | `todo listcon` | (none) | List unique @contexts in file |
| List projects | `todo listproj` | (none) | List unique +projects in file |

### diff

| Scenario | Command sample | Arguments | Expected behavior |
|----------|-----------------|-----------|--------------------|
| Two files | `todo diff todo.txt todo_changed.txt` | FROM TO | Semantic unified diff to stdout |
| Output to file | `todo diff from.txt to.txt -o out.patch` | FROM TO -o FILE | Write patch to out.patch |
| Default paths | `todo diff` | (none) | Use configured/default FROM and TO |
| Line-based | `todo diff --line-based a.txt b.txt` | --line-based FROM TO | Standard line-based unified diff |

### patch

| Scenario | Command sample | Arguments | Expected behavior |
|----------|-----------------|-----------|--------------------|
| Apply to file | `todo patch -i changes.patch todo.txt` | -i PATCH FILE | Apply patch to todo.txt |
| Stdin patch | `todo patch todo.txt < changes.patch` | FILE (stdin) | Apply patch from stdin to todo.txt |
| Default target | `todo patch -i changes.patch` | -i PATCH | Apply to default todo file |
| Dry run | `todo patch --dry-run -i changes.patch todo.txt` | -n -i PATCH FILE | Preview only, no write |
| Strip level | `todo patch -p1 -i p.patch todo.txt` | -p N -i PATCH FILE | Strip N path components from patch paths |

### merge

| Scenario | Command sample | Arguments | Expected behavior |
|----------|-----------------|-----------|--------------------|
| Three-way stdout | `todo merge base.txt ours.txt theirs.txt` | BASE OURS THEIRS | Merged result to stdout |
| Three-way to file | `todo merge -o merged.txt base ours theirs` | -o FILE BASE OURS THEIRS | Write merged result to merged.txt |
| Two-way | `todo merge ours.txt theirs.txt` | OURS THEIRS | Two-way merge; strategy TBD |
| Abort on conflict | `todo merge --abort-on-conflict base ours theirs` | --abort-on-conflict BASE OURS THEIRS | Exit with error on conflict |

### config

| Scenario | Command sample | Arguments | Expected behavior |
|----------|-----------------|-----------|--------------------|
| Show all | `todo config` | (none) | Print resolved config |
| List keys | `todo config --list` | --list | Print config key names |
| Get value | `todo config get todo_file` | get KEY | Print value for KEY |
| Set value | `todo config set todo_file /path/to/todo.txt` | set KEY VALUE | Set KEY and persist |

---

## Cross-Cutting Notes

### Default todo file

When `todo_file` is not set in config, the default path comes from the platform application-data directory (XDG “state”/data location), not the current working directory.

- **Linux:** `$XDG_DATA_HOME/todotools/todo.txt`. If `XDG_DATA_HOME` is unset or empty, use `~/.local/share/todotools/todo.txt`. This follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) for user-specific data.
- **macOS:** Same as Linux: `$XDG_DATA_HOME/todotools/todo.txt` with fallback to `~/.local/share/todotools/todo.txt` (many CLI tools use XDG-style paths on macOS too).
- **Windows:** `%APPDATA%\todotools\todo.txt` (e.g. `C:\Users\<user>\AppData\Roaming\todotools\todo.txt`). Application data lives in Roaming; same idea as `XDG_DATA_HOME` for user-specific data.

Override: config key `todo_file`, or global `-f`/`--file`, or environment variable (e.g. `TODO_FILE`) if supported.

### Exit codes

- `0`: Success.
- Non-zero: Error (e.g. file not found, parse error, merge conflict when using abort-on-conflict). Exact codes TBD (e.g. 1 generic, 2 conflict).

### Compatibility

- **diff:** Invocation mirrors `diff -u FROM TO`; output format is todo-diff (TODO_DIFF_SPEC), not raw GNU diff.
- **patch:** Invocation and options (e.g. `-i`, `-p`, `--dry-run`) mirror GNU `patch` where applicable.
- **list:** Filter semantics (TERM, +project, @context, -exclude, listpri/listcon/listproj) align with TODO.sh for familiarity.
