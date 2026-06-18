---
name: safe-delete
description: >
  Never hard-delete files. Move them to a project-local .trash/ folder
  (a recycle bin for the repo) so any deletion is recoverable.
  Trigger: when about to delete, remove, rm, Remove-Item, or clean up
  any file or folder in the project.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- The user asks to **delete / remove / borrar / limpiar** a file or folder.
- You are about to run `rm`, `rm -rf`, `Remove-Item`, `del`, or any destructive cleanup.
- A refactor or move makes a file obsolete and you'd otherwise delete it.

Load this skill **before** the deletion, not after.

## Critical Patterns

1. **NEVER permanently delete.** No `rm -rf`, no `Remove-Item -Recurse`, no wildcards
   on real files. The only exception is emptying the trash (see below), which is
   the one destructive op and ALWAYS needs explicit user confirmation.
2. **Move, don't remove.** Send every deletion to `.trash/<timestamp>/<original/relative/path>`.
   Preserving the relative path inside a timestamped folder means restoring is a single
   move — no log file, no guessing where it came from.
3. **One timestamp per deletion batch.** Delete 5 files in one request → one timestamp
   folder holding all 5 at their relative paths.
4. **Confirm scope before deleting more than one item.** If the user says "delete the image"
   and there are several, ASK which one. Mass deletion is exactly the failure this prevents.
5. **`.trash/` is gitignored** and lives at the project root.

## Code Examples

### Move to trash (the safe delete)

```powershell
# PowerShell (Windows)
$ts   = Get-Date -Format "yyyyMMdd-HHmmss"
$src  = "output/post-1.png"            # file or folder to "delete"
$dest = Join-Path ".trash/$ts" $src
New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
Move-Item -LiteralPath $src -Destination $dest
```

```bash
# bash (macOS / Linux)
ts="$(date +%Y%m%d-%H%M%S)"
src="output/post-1.png"
dest=".trash/$ts/$src"
mkdir -p "$(dirname "$dest")"
mv "$src" "$dest"
```

### Restore (undo a deletion)

```powershell
# Move it back to where it came from — the relative path is preserved inside the timestamp folder
Move-Item -LiteralPath ".trash/20260618-101500/output/post-1.png" -Destination "output/post-1.png"
```

```bash
mv ".trash/20260618-101500/output/post-1.png" "output/post-1.png"
```

### List what's in the trash

```powershell
Get-ChildItem -Recurse .trash | Select-Object FullName, LastWriteTime
```

### Empty the trash — DESTRUCTIVE, confirm first

```powershell
# Only after the user explicitly confirms. This cannot be undone.
Remove-Item -Recurse -Force .trash/*
```

## Commands

```bash
# One-time setup: ignore the trash in git
echo ".trash/" >> .gitignore
```

## Resources

- **Stronger enforcement**: a skill is advisory — Claude reads it and *should* obey.
  For a guarantee the harness enforces every time, add a `PreToolUse` hook on `Bash`
  that blocks `rm -rf` / `Remove-Item -Recurse` and redirects to the move-to-trash flow above.
