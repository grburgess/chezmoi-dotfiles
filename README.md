# chezmoi-dotfiles

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that keeps the agent from clobbering dotfiles managed by [chezmoi](https://www.chezmoi.io/). When the user asks to edit `~/.zshrc`, `~/.gitconfig`, anything under `~/.config/`, or any other home-directory dotfile, the skill activates and forces the agent to resolve the chezmoi source path *before* touching the file, edit the source rather than the target, and run `chezmoi apply` to propagate the change.

## The problem

Dotfiles under chezmoi management are regenerated from a source state on every `chezmoi apply`. An LLM coding agent that has not been told about this will happily `Read` and `Edit` `~/.zshrc` directly, and the edit will silently vanish the next time the user (or their `chezmoi-init` hook) applies. The result appears, to the user, as Claude having lied about completing a task — the diff was real, it just did not survive.

## What this skill does

When loaded, the skill imposes an iron rule on the agent: before any `Edit` or `Write` to a path under `~/` that begins with `.`, it must run `chezmoi source-path <target>`. The command returns the source path if the file is managed, or exits non-zero if it is not. The agent edits the returned path, runs `chezmoi diff` and `chezmoi apply`, and verifies the change landed. Template files (`.tmpl` suffix) are flagged for Go-template preservation; encrypted sources (`encrypted_`, `.age`, `.gpg`) are deferred to interactive `chezmoi edit --apply`, which the user runs themselves.

The skill is loaded automatically by Claude Code's skill-discovery mechanism whenever a request mentions editing a dotfile path that matches the description triggers. No hook configuration is required.

## Install

The skill is a single `SKILL.md` file, plus its parent directory name. Drop it into `~/.claude/skills/`:

```bash
git clone https://github.com/grburgess/chezmoi-dotfiles.git ~/.claude/skills/chezmoi-dotfiles
```

That is sufficient. Claude Code scans `~/.claude/skills/` on startup and picks up the new entry via its frontmatter `description`. To verify, start a Claude Code session and ask it to list available skills; `chezmoi-dotfiles` should appear.

If you prefer to keep the repo checkout elsewhere and symlink (so edits flow back to git without copying), that also works:

```bash
git clone https://github.com/grburgess/chezmoi-dotfiles.git ~/code/chezmoi-dotfiles
ln -s ~/code/chezmoi-dotfiles ~/.claude/skills/chezmoi-dotfiles
```

## What it expects of the environment

The skill assumes `chezmoi` is on `$PATH` and that a source directory is configured (`chezmoi source-path` returns a valid path with no arguments). It does not assume the source dir is a git repository, though it will surface `git status` in the source dir when one is present, so the user can see what is pending. It does not commit or push on the user's behalf.

## Caveats

The skill is a documentation-level constraint, not a hook. It guides the agent's behaviour through the system prompt; it does not block direct edits at the tool layer. An agent that ignores its loaded skills can still write to `~/.zshrc` directly. In practice, with current Claude Code models, compliance has been reliable in baseline testing — but we cannot exclude failure modes under sufficient adversarial pressure, and we leave hook-level enforcement (e.g. a `PreToolUse` hook that intercepts edits to managed paths) for a future revision.

The skill also assumes the user's chezmoi setup is conventional: a single source directory, no exotic externals, no custom data hooks that would invalidate `chezmoi source-path` output. Users with heavily customised setups should treat the skill as a starting point and adapt the workflow section accordingly.

## License

MIT.
