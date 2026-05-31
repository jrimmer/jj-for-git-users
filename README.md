# jj-for-git-users

A practical Jujutsu (`jj`) guide for people who already know Git, plus a reusable coding-agent skill that teaches agents to use idiomatic jj instead of translating Git commands mechanically.

Start here:

- [Jujutsu (`jj`) for Git Users: the Dense Guide](jj_for_git_users.md)
- [Idiomatic jj coding-agent skill](skills/idiomatic-jj/SKILL.md)

The guide focuses on the Git-to-jj mental model shift, daily workflows, bookmarks, remotes, PRs, revsets, conflicts, and common gotchas.

The skill is for AI coding assistants. It tells them how to recognize jj repositories, prefer jj-native commands, avoid Git-index assumptions, use bookmarks correctly, inspect revsets before mutating history, and recover safely with the operation log.

## What is in this repo?

```text
README.md                         # This installation and usage overview
jj_for_git_users.md                # Human-readable dense guide
skills/idiomatic-jj/SKILL.md       # Agent Skills-compatible package
```

## Using the human guide

Open or link to [`jj_for_git_users.md`](jj_for_git_users.md) when you want a concise Git-to-jj reference. It covers:

- the working-copy commit (`@`) and parent (`@-`)
- no staging area / no Git index
- stable change IDs vs Git commit IDs
- bookmarks vs branches
- daily local workflows
- stacked work
- history editing and undo
- conflicts as first-class commit data
- revsets
- PR/push workflows

## Using the `idiomatic-jj` agent skill

The skill lives at [`skills/idiomatic-jj/SKILL.md`](skills/idiomatic-jj/SKILL.md). It is a standard `SKILL.md` directory package, so tools that support Agent Skills can discover it directly.

Ask your agent something like:

```text
Use the idiomatic-jj skill. Inspect the current jj repo and tell me the safest way to update my PR.
```

or:

```text
When working in this repo, follow skills/idiomatic-jj/SKILL.md for jj commands and Git-to-jj translations.
```

The skill teaches agents to:

- start by checking `jj status` and `jj log`
- treat `@` as the working-copy commit and remember that completed work is often `@-`
- use `jj split`, `jj squash -i`, and `jj restore` instead of pretending there is a staging area
- treat bookmarks as publication handles, not checked-out branches
- use `jj git fetch` plus explicit `jj rebase` instead of a Git-style `pull` reflex
- inspect revsets with `jj log -r '<revset>'` before mutating history
- resolve conflicts by editing conflicted commits, not by running `--continue`
- recover with `jj undo`, `jj op log`, and `jj op restore`

## Install the skill in the top 5 coding harnesses

These examples assume you cloned this repo and are running commands from its root. Replace `/path/to/project` with the project where you want the agent to use the skill.

Covered here: **Pi**, **Zed**, **Claude Code**, **OpenAI Codex CLI**, and **Cursor**.

### Pi

Pi loads skills from `~/.pi/agent/skills/`, project `.pi/skills/`, and project `.agents/skills/`.

Global install:

```bash
mkdir -p ~/.pi/agent/skills
cp -R skills/idiomatic-jj ~/.pi/agent/skills/
```

Project-local install:

```bash
mkdir -p /path/to/project/.pi/skills
cp -R skills/idiomatic-jj /path/to/project/.pi/skills/
```

Use:

```text
/skill:idiomatic-jj
```

or ask a jj question; the skill description is written so Pi can load it when relevant.

### Zed

Zed supports Agent Skills from `~/.agents/skills/` globally and `<worktree>/.agents/skills/` project-locally.

Global install:

```bash
mkdir -p ~/.agents/skills
cp -R skills/idiomatic-jj ~/.agents/skills/
```

Project-local install:

```bash
mkdir -p /path/to/project/.agents/skills
cp -R skills/idiomatic-jj /path/to/project/.agents/skills/
```

Use from the Agent Panel with `/idiomatic-jj` or by @-mentioning the skill. Project-local skills require the worktree to be trusted.

### Claude Code

Claude Code supports personal and project skills.

Personal/global install:

```bash
mkdir -p ~/.claude/skills
cp -R skills/idiomatic-jj ~/.claude/skills/
```

Project-local install:

```bash
mkdir -p /path/to/project/.claude/skills
cp -R skills/idiomatic-jj /path/to/project/.claude/skills/
```

Use:

```text
/idiomatic-jj
```

or ask Claude Code to use the `idiomatic-jj` skill while working in a jj repository.

### OpenAI Codex CLI

Codex reads `AGENTS.md` instructions. Keep the skill in the project and point Codex at it.

Project-local install:

```bash
mkdir -p /path/to/project/skills
cp -R skills/idiomatic-jj /path/to/project/skills/
cat >> /path/to/project/AGENTS.md <<'EOF'

## Idiomatic jj

When operating in a Jujutsu (`jj`) repository, read and follow `skills/idiomatic-jj/SKILL.md`. Prefer jj-native workflows over Git command translations, especially around `@`, `@-`, bookmarks, revsets, conflicts, and the operation log.
EOF
```

Personal/global install:

```bash
mkdir -p ~/.codex/skills
cp -R skills/idiomatic-jj ~/.codex/skills/
cat >> ~/.codex/AGENTS.md <<'EOF'

## Idiomatic jj

When operating in a Jujutsu (`jj`) repository, follow `~/.codex/skills/idiomatic-jj/SKILL.md` for jj-native command choices and Git-to-jj translations.
EOF
```

Use by asking Codex to inspect the repo and follow the `Idiomatic jj` instructions.

### Cursor

Cursor project rules live in `.cursor/rules/*.mdc`. Convert the skill body into a project rule:

```bash
mkdir -p /path/to/project/.cursor/rules
{
  printf '%s\n' '---'
  printf '%s\n' 'description: Use when working in Jujutsu (jj) repositories or translating Git workflows to jj.'
  printf '%s\n' 'alwaysApply: false'
  printf '%s\n\n' '---'
  tail -n +6 skills/idiomatic-jj/SKILL.md
} > /path/to/project/.cursor/rules/idiomatic-jj.mdc
```

Set `alwaysApply: true` if nearly all work in that project should assume jj. Otherwise, keep it requestable and mention the rule when needed:

```text
Use the idiomatic-jj Cursor rule for this jj workflow.
```

## Keeping the skill up to date

If you installed by copying, update by copying the folder again:

```bash
rm -rf ~/.agents/skills/idiomatic-jj
cp -R skills/idiomatic-jj ~/.agents/skills/
```

Adjust the destination for Pi, Claude Code, Codex, or a project-local install.

## License / reuse

This repository is intended as reusable educational material. If you adapt the skill for another harness, keep the core jj guidance intact and adjust only the packaging/frontmatter required by your tool.
