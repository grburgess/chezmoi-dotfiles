---
name: chezmoi-dotfiles
description: Use when editing, adding, or modifying any dotfile or config file in the user's home directory (~/.zshrc, ~/.zshenv, ~/.gitconfig, ~/.vimrc, ~/.tmux.conf, ~/.config/**, ~/.ssh/config, etc.) — these may be managed by chezmoi and direct edits are destroyed on the next `chezmoi apply`. Always resolve the chezmoi source path first and edit the source, never the target.
---

# Editing chezmoi-managed dotfiles

## Overview

Files in `~/` starting with `.` (and most things in `~/.config/`) are likely managed by chezmoi on this machine. **Direct edits to those targets are silently overwritten** on the next `chezmoi apply`. The source of truth lives at `~/.local/share/chezmoi/` (run `chezmoi source-path` to confirm). Edit the source, then apply.

## The Iron Rule

**Before any `Edit` or `Write` on a path under `~/` that starts with `.`, run:**

```bash
chezmoi source-path <target>
```

- **Exits 0 with a path** → file is managed. Edit THAT source path. Then `chezmoi apply <target>`.
- **Exits non-zero (`not managed`)** → file is unmanaged. Edit the target directly is fine.

No exceptions. Don't skip the check because "it's a small change" or "the target is simpler" — the diff gets wiped.

## Workflow

```bash
# 1. Resolve source
src=$(chezmoi source-path ~/.zshenv)
# → /Users/burgessj/.local/share/chezmoi/dot_zshenv

# 2. Edit $src with Read + Edit tools (NOT the target)

# 3. Preview, then apply
chezmoi diff ~/.zshenv
chezmoi apply ~/.zshenv
```

After step 3, verify the change landed in the target.

## Source filename conventions

`chezmoi source-path` returns paths like `dot_zshenv`, `private_dot_ssh/config`, `executable_dot_scripts/foo`, `dot_gitconfig.tmpl`. **Trust the returned path verbatim** — never construct source paths by guessing prefixes.

| Source name pattern | Target |
|---------------------|--------|
| `dot_X` | `~/.X` |
| `dot_config/Y` | `~/.config/Y` |
| `private_X` | `~/X` (mode 0600) |
| `executable_X` | `~/X` (mode +x) |
| `readonly_X` | `~/X` (mode 0444) |
| `X.tmpl` | rendered Go template |
| `encrypted_X` / `X.age` / `X.gpg` | decrypted on apply |

## Templates (`.tmpl` suffix)

If the source ends in `.tmpl`, you are editing a **Go template**, not the rendered file. Preserve all `{{ ... }}` tokens, e.g. `{{ .email }}`, `{{- if eq .chezmoi.os "darwin" -}}`. Validate before applying:

```bash
chezmoi execute-template < "$src"   # render and inspect
chezmoi apply --dry-run <target>    # check for errors
```

## Encrypted files

If the source name contains `encrypted_` or the file ends in `.age`/`.gpg`, **do not Read/Edit it directly** — it's ciphertext. Ask the user to run:

```bash
chezmoi edit --apply ~/.authinfo
```

This opens the plaintext in their `$EDITOR`. It's interactive; don't try to spawn an editor from a tool call.

## Creating a new dotfile

```bash
# Option A: write the file at the live target, then track it
chezmoi add ~/.newrc

# Option B: create it directly inside the source dir with proper prefix
#   (only if you already know the conventions)
```

Prefer Option A — `chezmoi add` picks the right prefix and mode automatically.

## Source dir is usually a git repo

`~/.local/share/chezmoi/` is typically version-controlled. After editing the source:

- Show `git status` / `git diff` in the source dir so the user sees what's staged for their dotfiles repo.
- **Do not commit or push** in the chezmoi source dir unless explicitly asked — the user owns that history.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Edit `~/.zshenv` directly | Edit `$(chezmoi source-path ~/.zshenv)` then `chezmoi apply ~/.zshenv` |
| Guess source path (`dot_zshenv`) without checking | Always call `chezmoi source-path <target>` |
| Skip `chezmoi apply` after editing source | Target stays stale until you apply |
| Edit a `.tmpl` source as if it were plaintext | Preserve `{{ ... }}` tokens; validate with `chezmoi execute-template` |
| Read/Edit an `encrypted_` source | Ask user to run `chezmoi edit --apply <target>` |
| `git commit` in chezmoi source dir unprompted | Show diff, let user commit |

## Red flags — STOP and run `chezmoi source-path`

- About to `Edit`/`Write` a path matching `~/.*` or `~/.config/*`
- "Just appending one line, no need to check"
- "The source path prefix looks ugly, I'll edit the target"
- Asked to "add an alias to my zshrc / fix my gitconfig / change a config in ~/.config/..."

**All of these → check `chezmoi source-path` first. No exceptions.**
