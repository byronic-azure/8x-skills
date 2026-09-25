---
name: paean-skills-update
description: Update the local 8x-skills repository and reinstall Paean skills for Claude Code, Codex, or Zero CLI. Use when the user asks to update skills, refresh Paean skills, pull the latest skill instructions, or sync paean-publish / paean-remix / paean-zero-setup skill changes.
---

# Paean Skills Update (Codex)

Update the local `8x-skills` checkout and refresh Paean skill instructions. Use this when the
user asks to update skills, refresh Paean skills, pull the latest skill instructions, or sync
`paean-publish` / `paean-remix` / `paean-zero-setup` changes.

> **Using this skill in Codex.** Reference this file explicitly — add a pointer in your
> project `AGENTS.md` ("To update Paean skills, follow
> `8x-skills/codex/paean-skills-update/SKILL.md`.") or name the skill in your prompt.

## Claude Code plugin install (no checkout needed)

If Claude Code has the skills installed as the plugin `paean@8x-skills` (check with
`claude plugin list`), update them through the plugin system. This refreshes the Claude Code
side without Git or a checkout:

```bash
claude plugin marketplace update 8x-skills
claude plugin update paean@8x-skills
```

The update applies to the next Claude Code session; in a session that is already open, run
`/reload-plugins`. If manual copies from an earlier install still exist
(`~/.claude/skills/paean-*` or a project's `.claude/skills/paean-*`), they load alongside
the plugin's namespaced skills — offer to remove them and delete them only with the user's
approval.

If the plugin is not installed yet and the user wants it, install it instead of copying
files (for a fork, substitute the fork's `owner/8x-skills`):

```bash
claude plugin marketplace add paean-ai/8x-skills
claude plugin install paean@8x-skills
```

The plugin update covers Claude Code only. If the user also maintains a local `8x-skills`
checkout — for Codex pointers, Zero CLI copies, or a vendored copy — continue with the
checkout-based steps below so those stay current, skipping only the Claude Code manual-copy
section. Stop here only when the plugin is the sole installation.

## Locate the skills repo

Prefer the current repo if it contains `claude-code/`, `codex/`, and `zero/`. Otherwise look
for the user's checkout (ask where it lives, or check common spots such as
`~/Zero/opensource/8x-skills` or `~/a8e/paean-opensource/8x-skills`). If there is no
checkout, clone it:

```bash
git clone https://github.com/paean-ai/8x-skills
cd 8x-skills
```

Check status before pulling. Do not discard local changes.

```bash
git status --short
git remote -v
```

If there are local changes, report them first. Pull only when the user wants the remote update
and the changes do not create an obvious conflict.

```bash
git pull --ff-only
```

If `--ff-only` fails, stop and report the conflict/divergence; do not reset or overwrite.

## Reinstall for Claude Code (manual copies)

Skip this when the plugin is installed (see above) — copied skills would load twice. Otherwise
copy each Claude Code skill directory into the global skills folder:

```bash
mkdir -p ~/.claude/skills
cp -R claude-code/paean-publish ~/.claude/skills/
cp -R claude-code/paean-remix ~/.claude/skills/
cp -R claude-code/paean-sdk ~/.claude/skills/
cp -R claude-code/paean-zero-setup ~/.claude/skills/
cp -R claude-code/paean-skills-update ~/.claude/skills/
```

If a project uses `.claude/skills/`, copy there instead or in addition.

## Refresh Codex pointers

Codex can use this repo in place. Ensure the project `AGENTS.md` points at the current files:

```markdown
## Skills
- To update Paean skills, follow `8x-skills/codex/paean-skills-update/SKILL.md`.
- To install Zero CLI or log in to Paean for publishing, follow `8x-skills/codex/paean-zero-setup/SKILL.md`.
- To publish to Paean Apps Square, follow `8x-skills/codex/paean-publish/SKILL.md`.
- To remix Paean Apps Square games, follow `8x-skills/codex/paean-remix/SKILL.md`.
- To add cloud save or a shared leaderboard via the Paean Web SDK, follow `8x-skills/codex/paean-sdk/SKILL.md`.
```

If the project keeps a vendored copy of `8x-skills/`, update that copy from this checkout with
the user's approval.

## Reinstall for Zero CLI

Zero CLI discovers skills from a `skills/` directory (project `.zero/skills/` or the global
config dir). Copy each Zero skill directory in:

```bash
mkdir -p ~/.zero/skills
cp -R zero/paean-publish ~/.zero/skills/
cp -R zero/paean-remix ~/.zero/skills/
cp -R zero/paean-zero-setup ~/.zero/skills/
cp -R zero/paean-sdk ~/.zero/skills/
cp -R zero/paean-skills-update ~/.zero/skills/
```

If a project uses `.zero/skills/`, copy there instead or in addition.

## Verify

```bash
claude plugin validate .
find claude-code codex zero -maxdepth 2 -name SKILL.md | sort
node --check codex/paean-publish/scripts/publish.mjs
node --check codex/paean-remix/scripts/remix.mjs
node --check zero/paean-publish/scripts/publish.mjs
node --check zero/paean-remix/scripts/remix.mjs
```

Report the current commit hash and any files that remain modified.
