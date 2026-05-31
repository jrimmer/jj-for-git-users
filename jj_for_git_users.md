# Jujutsu (`jj`) for Git Users: the Dense Guide

Generated on **2026-05-30** by **OpenAI Codex GPT-5.5**.

A practical, opinionated intro to [Jujutsu](https://github.com/jj-vcs/jj) for people who already know Git. It condenses Steve Klabnik's [Jujutsu for everyone](https://jj-for-everyone.github.io/) and the official docs into one long page that you can skim, search, and keep open while trying `jj` in a real repo.

Use this as a quick path from "I know Git" to "I can use jj safely in a real repo." It focuses on the parts that usually trip up Git users: the always-committed working copy, no index, stable change IDs, bookmark-vs-branch semantics, PR pushing, revsets, and the operation log.

`jj` is a Git-compatible VCS front end and model. In normal use, commits still live in a Git repo and can be pushed to GitHub/GitLab. The UX and data model are different: your working copy is a real commit, commits evolve via stable change IDs, history editing is ordinary, conflicts are first-class, and every operation is undoable.

This repo also includes an **`idiomatic-jj` coding-agent skill** at [`skills/idiomatic-jj/SKILL.md`](skills/idiomatic-jj/SKILL.md). It packages the jj mental model and best-practice command choices for AI coding assistants, so they are less likely to apply Git habits like staging, current-branch assumptions, or `rebase --continue` workflows to jj repositories. See the [README](README.md#install-the-skill-in-the-top-5-coding-harnesses) for install and usage instructions.

**Version note:** command names and flags in jj have changed over time. This guide was checked against `jj 0.41.0`; if your installed version disagrees, trust `jj help <command>` and the latest official docs.

## Table of contents

- [Start here: the mental model shift](#mental-model)
- [Install and initialize](#install)
- [Agent skill for AI coding assistants](#agent-skill)
- [The 10-minute survival workflow](#survival-workflow)
- [Git-to-jj command map](#command-map)
- [Concept translation: Git words vs jj words](#concept-translation)
- [The working-copy commit, `@`, `@-`, and change IDs](#working-copy)
- [Everyday local workflows](#local-workflows)
- [Bookmarks, branches, remotes, and PRs](#bookmarks-remotes-prs)
- [Stacked work](#stacked-work)
- [History editing without fear](#history-editing)
- [Conflicts](#conflicts)
- [Revsets: jj's query language](#revsets)
- [Suggested config and aliases](#config-aliases)
- [FAQ / gotchas for Git users](#faq)
- [Further reading](#further-reading)

---

<a id="mental-model"></a>

## Start here: the mental model shift

If you remember only five things:

1. **There is no Git index/staging area.** The working copy is automatically snapshotted into a real commit.
2. **`@` is the working-copy commit.** Its parent is `@-`. After `jj commit`, your finished work is usually `@-`, because `@` becomes a new empty working commit.
3. **Commits have two IDs:** a normal Git **commit ID** and a stable jj **change ID**. Editing a commit creates a new commit ID but preserves the change ID.
4. **Branches are bookmarks.** A bookmark is a named pointer. There is **no current branch/bookmark**, and bookmarks do **not** automatically advance when you make commits.
5. **Operations are undoable.** `jj op log`, `jj undo`, and `jj op restore <op>` are the safety net Git users wish reflogs were.

`jj` feels weird for a day because Git teaches you to finish work with `git commit`; `jj` often has you *start* work with a new change, then evolve it until it is ready. That inversion is what lets jj treat rewriting, splitting, moving hunks, and rebasing descendants as normal operations instead of special cleanup rituals.

A useful way to phrase the model is: **Git asks you to decide what is a commit at commit time; jj lets a change become a commit gradually.** You can describe it before writing code, edit it while coding, split it after the fact, fold review fixes into it later, and still talk about "the same change" because the change ID survives those rewrites.

If you are nervous about trying jj in an existing project, start with a personal repository or a throwaway clone. The Git backend means the commits you create are still ordinary Git commits. jj also adds its own `.jj` metadata locally. Your team does not need to know or care unless you push bookmarks/branches in unusual ways.

---

<a id="install"></a>

## Install and initialize

Install from the official instructions for your platform: <https://docs.jj-vcs.dev/latest/install-and-setup/>. The CLI is named `jj`; the project/VCS is named Jujutsu. Most of the time you will say "jj" because that is what you type.

Common paths:

```bash
# macOS
brew install jj

# Rust/Cargo
cargo install --locked jj-cli
```

Set identity once:

```bash
jj config set --user user.name "Your Name"
jj config set --user user.email "you@example.com"
```

Use the Git backend unless you have a specific reason not to:

```bash
# New Git-backed repo in the current directory
jj git init

# Existing Git repository: add jj metadata beside .git
jj git init --git-repo .

# Clone a Git repo with jj
jj git clone https://github.com/org/repo.git
```

You can mix `git` and `jj` in a colocated repo. `jj` syncs with Git automatically when you run `jj` commands. This is the recommended on-ramp for Git users: keep your existing Git remote, keep GitHub/GitLab, keep your CI, keep other people's workflows, and only swap your local command-line workflow.

A few setup details matter:

- `jj git init` creates a `.jj` directory. If you have a global Git ignore file, it is reasonable to ignore `.jj/` globally so you never accidentally add it in non-colocated experiments.
- `jj` respects `.gitignore`; you do not need a separate ignore format.
- If `jj` warns that name/email are not configured, set them before creating pushable commits. Changing identity later does not automatically rewrite already-created commits unless you ask it to.
- Prefer `jj git init` / `jj git clone` while the native backend is still less common. Use Git-backed repos for interop today.

---

<a id="agent-skill"></a>

## Agent skill for AI coding assistants

If you use AI coding agents, install the included [`idiomatic-jj` skill](skills/idiomatic-jj/SKILL.md). The skill gives agents a compact, operational version of this guide: check `jj status`/`jj log`, treat `@` as the working-copy commit, remember completed work is often `@-`, use `split`/`squash` instead of a Git index, handle bookmarks as publication pointers, inspect revsets before mutating history, and recover with the operation log.

The [README](README.md#install-the-skill-in-the-top-5-coding-harnesses) has install and usage instructions for Pi, Zed, Claude Code, OpenAI Codex CLI, and Cursor. After installing, prompt your agent with something like:

```text
Use the idiomatic-jj skill for this repository. Prefer jj-native commands and explain any Git-to-jj translation.
```

This is especially useful for models that understand Git well but still default to non-idiomatic jj advice such as staging files, assuming an active branch moves with commits, or treating conflicts as a `--continue` state machine.

---

<a id="survival-workflow"></a>

## The 10-minute survival workflow

```bash
# See where you are
jj status          # or: jj st
jj log

# Start work from trunk/main
jj new main        # or: jj new trunk() in repos configured with trunk()

# Edit files. jj automatically snapshots them into @.
$EDITOR file

# Inspect changes
jj diff            # diff for @ against its parent
jj status

# Finish current change and start the next empty one
jj commit -m "feat: useful change"

# You are now on a fresh empty @. The commit you just made is @-.

# Push it for review with an auto-generated bookmark
jj git push --change @-     # short: jj git push -c @-
```

If you prefer named PR branches/bookmarks:

```bash
jj bookmark create my-feature -r @-
jj bookmark track my-feature
jj git push
```

Update from remote:

```bash
jj git fetch
jj rebase -b @ -d main      # rebase current branch/stack onto main, if needed
```

Undo a mistake:

```bash
jj undo                     # undo last jj operation
jj op log                   # inspect operation history
jj op restore <operation>   # restore repo state at an earlier operation
```

What happened in that workflow? `jj new main` created a fresh change on top of `main` and made it the working-copy commit. Your file edits were automatically snapshotted into that commit. `jj commit -m ...` gave that commit a description and then created a new empty working-copy commit on top. Because the finished work is now the parent of the empty working copy, pushing uses `@-`.

This sounds fussy in prose. In practice, it removes the dirty/index/HEAD split from Git. There is no "did I stage that?" state. `jj status` tells you what is in `@`, `jj diff` shows what `@` changes relative to its parent, and `jj log` shows where `@` is in the graph.

---

<a id="command-map"></a>

## Git-to-jj command map

| Goal | Git | jj |
|---|---|---|
| New repo | `git init` | `jj git init` |
| Clone | `git clone URL` | `jj git clone URL` |
| Status | `git status` | `jj status` / `jj st` |
| Log current line | `git log --oneline --graph` | `jj log` or `jj log -r ::@` |
| Log all visible commits | `git log --all --graph` | `jj log -r 'all()'` or `jj log -r ::` |
| Diff working copy | `git diff` | `jj diff` |
| Show commit | `git show REV` | `jj show REV` |
| Stage all + commit | `git add -A && git commit -m msg` | `jj commit -m msg` |
| Amend parent with current work | `git commit --amend -a` | `jj squash` |
| Amend interactively | `git add -p && git commit --amend` | `jj squash -i` |
| Split current work into commits | `git add -p && git commit ...` | `jj split` |
| New work on top of current | `git switch -c tmp` / keep committing | `jj new` |
| Switch to a commit for editing | `git switch` / `git checkout` | `jj edit REV` |
| Create branch | `git branch name REV` | `jj bookmark create name -r REV` |
| Move branch | `git branch -f name REV` | `jj bookmark move name --to REV` |
| Fetch | `git fetch` | `jj git fetch` |
| Pull | `git pull --rebase` | `jj git fetch` then `jj rebase ...` |
| Push branch/bookmark | `git push origin name` | `jj git push --bookmark name` |
| Push current change | N/A | `jj git push -c REV` |
| Rebase stack | `git rebase main` | `jj rebase -b REV -d main` |
| Restore file | `git restore path` | `jj restore path` |
| Delete commit | `git reset --hard` / rebase | `jj abandon REV` |
| Reflog | `git reflog` | `jj op log` |
| Undo last operation | hard in Git | `jj undo` |

Read this table as a translation guide, not a rename list. Some rows are direct translations (`status`, `diff`, `fetch`). The interesting rows translate the model. `jj split` solves the same human problem as `git add -p` after jj has removed the index. `jj bookmark` maps to Git branches at the boundary, while local editing still happens on commits rather than on a checked-out branch.

When in doubt, ask "which commit am I editing?" and "which bookmark do I eventually want to move or push?" Those two questions replace a surprising amount of Git's HEAD/index/branch state machine.

---

<a id="concept-translation"></a>

## Concept translation: Git words vs jj words

| Git concept | jj concept | Practical difference |
|---|---|---|
| `HEAD` | `@` plus parent `@-` | `@` is the working-copy commit, including current file edits. |
| Commit hash | Commit ID | Same object when Git-backed. Changes whenever commit content/metadata changes. |
| No Git equivalent | Change ID | Stable identity for an evolving change across rewrites. |
| Branch | Bookmark | Named pointer. There is no checked-out/current bookmark. |
| Remote-tracking branch | Remote bookmark, e.g. `main@origin` | Similar purpose; can be tracked/untracked by local bookmark. |
| Index/staging area | Real commits + `squash`, `split`, `restore` | You manipulate changes between commits instead of staging paths/hunks. |
| Reflog | Operation log | Repo-wide operation history, used for undo/restore. |
| Merge/rebase/cherry-pick conflicts | First-class conflicts | Conflicts can be committed/rebased/resolved later; no `--continue` flow. |

The table compresses a deeper model change: jj treats the commit graph as the source of truth, with the working copy as one editable commit in that graph. Git's UI exposes several overlapping states: refs, HEAD, index, working tree, rebase state, merge state, reflog state. jj collapses much of that into commits plus operations.

The cost is vocabulary. Saying "branch" when you mean "bookmark" is usually fine at the Git boundary. Locally, it can send you down the wrong path. Saying "HEAD" when you mean `@-` creates the same problem. Learn `@`, `@-`, change ID, bookmark, revset, operation; those are the terms that make jj click.

---

<a id="working-copy"></a>

## The working-copy commit, `@`, `@-`, and change IDs

In jj, your working directory is represented by a commit called **the working-copy commit**, written `@`.

```text
@   current working-copy commit
│
○   parent, written @-
│
○   earlier history
```

Useful references:

```bash
@       # current working-copy commit
@-      # first parent of @; often your last completed change
@--     # grandparent
REV+    # children of REV
REV-    # parents of REV
main    # local bookmark named main
main@origin  # remembered remote bookmark
```

When you edit files, jj snapshots those edits into `@`. The **commit ID** changes, but the **change ID** stays stable. That stable change ID is why jj can transparently evolve commits and keep descendants/bookmarks following rewrites.

`jj commit -m "message"` is roughly:

```bash
jj describe -m "message"
jj new
```

So after committing, your described work is usually `@-`, and `@` is a fresh empty commit. If `jj status` says the working copy is clean after a commit, that does not mean there is no current commit; it means the current working-copy commit is empty relative to its parent.

This is also why jj output often shows both a **change ID** and a **commit ID**. The change ID is optimized for humans working locally: it remains stable as the change evolves. The commit ID is the Git-compatible immutable object ID: it changes whenever the commit's content, parents, author/committer metadata, or description changes. If you amend a Git commit, the hash changes; jj makes that evolution explicit and gives it a stable name.

You can refer to either kind of ID on the command line. In everyday use, prefer the short highlighted prefixes from `jj log`. jj deliberately uses different character sets for change IDs and commit IDs, which reduces ambiguity when you paste short IDs.

---

<a id="local-workflows"></a>

## Everyday local workflows

There are three common local styles. They can be mixed within one repo; they are not global modes. Pick the one that matches the kind of work in front of you. If you are learning, start with the simple workflow, then add squash/split when you feel the missing index muscle memory.

### Workflow A: simple commit-as-you-go

Good for first week usage and small changes. This is closest to normal Git: edit files, look at the diff, commit with a message, repeat. The main difference is that your edits are already in `@`; `jj commit` is more like "name this change and move on" than "package staged content into a new object."

```bash
jj new main
# edit files
jj diff
jj commit -m "feat: add thing"
# edit more files
jj commit -m "test: cover thing"
```

### Workflow B: squash workflow, like a better amend loop

Good when you want a clean commit while experimenting freely. This is the workflow that feels most like using Git's index for a carefully curated commit, except the "staging area" is a real child commit. You keep the meaningful commit as the parent and use the current working-copy commit as scratch space. When a piece of scratch work belongs in the real commit, squash it down.

```bash
jj new main
jj describe -m "feat: add parser"
jj new                    # scratch @ above the described commit

# edit files, run tests, edit more
jj diff

# Move all current scratch changes into the parent commit
jj squash

# Or move selected hunks/files
jj squash -i
jj squash src/parser.rs
```

Mental mapping: current `@` is scratch; parent `@-` is the commit you are polishing.

### Workflow C: edit workflow, hop between commits

Good when working on a stack and touching old commits.

```bash
jj log
jj edit <change-id-or-revset>
# edit files; the selected commit evolves
jj new                    # return to a new child when done
```

Because descendants auto-rebase, editing an older commit in a stack is normal. In Git, editing the third commit in a stack usually means interactive rebase, stop/edit, amend, continue, and hope the rest of the stack reapplies. In jj, you can `jj edit` that commit directly; when it changes, descendants are automatically rewritten on top of the new version.

The tradeoff: while you are editing an old commit, new file edits go into that old commit. Run `jj log`/`jj status` often until this is muscle memory. When finished, move back to a suitable child or create a new one with `jj new`.

### Split messy work

```bash
jj split                  # interactively split @ into two commits
jj split -r REV           # split another commit
```

This replaces much of the `git add -p` / partial commit workflow. You can do messy work first and separate it afterward. The built-in diff editor lets you select files/hunks; external diff tools can be configured later if you dislike the terminal UI.

### Move changes between commits

```bash
jj squash                 # move @ changes into parent
jj squash -i              # pick hunks
jj restore --from A --to B path   # copy file state/change across revisions
```

---

<a id="bookmarks-remotes-prs"></a>

## Bookmarks, branches, remotes, and PRs

A **bookmark** is jj's branch-like named pointer. At the Git boundary, bookmarks map to branches: pushing bookmark `feature` updates branch `feature` on the remote. Locally, bookmarks matter less than Git branches. You can do lots of local work with anonymous commits and create a bookmark later, when you need a stable name or a remote PR branch.

Critical Git-user gotcha: **there is no active bookmark.** If you are "on main" in your head and run `jj commit`, `main` does not automatically move. You must create/move bookmarks explicitly, or push a change with `-c`. This is not an oversight; it is what lets jj be branchless locally and avoid making every line of work depend on a named branch.

Think of bookmarks as publication handles, not as the place where editing happens. You edit commits. Later, you decide which bookmark should point at which commit. That one sentence prevents most Git-user confusion.

```bash
jj bookmark list --all
jj bookmark create feature -r @-
jj bookmark move feature --to @-
jj bookmark delete feature
```

Fetch/push:

```bash
jj git fetch
jj git push --bookmark feature
jj git push -c @-          # create/push a generated bookmark for this change
```

### PR workflow: generated bookmark

Fastest for GitHub/GitLab when you do not care about branch names:

```bash
jj new main
# edit
jj commit -m "refactor: simplify API"
# edit
jj commit -m "feat: add API option"

jj git push -c @-          # push parent of empty working copy
```

### PR workflow: named bookmark

```bash
jj new main
# edit, commit one or more times
jj commit -m "feat: add API option"

jj bookmark create api-option -r @-
jj bookmark track api-option
jj git push
```

### Updating a PR with a new follow-up commit

```bash
jj new api-option
# address review comments
jj commit -m "fix: address review"
jj bookmark move api-option --to @-
jj git push
```

### Updating a PR by rewriting clean commits

```bash
# Start from the commit that needs the fix; trailing - means parent.
jj new api-option-
# edit fix
jj squash                 # fold fix into parent
jj git push --bookmark api-option
```

`jj git push` performs safety checks similar in spirit to `--force-with-lease` when moving existing remote bookmarks. It checks jj's last-seen state of the remote before updating it, so an unexpected remote move should make the push fail until you fetch and reconcile.

For GitHub, a generated bookmark from `jj git push -c REV` is often enough: GitHub sees a branch and can open a PR. If your team cares about branch names, create a named bookmark. If your team uses branch naming conventions (`user/feature`, ticket IDs, etc.), encode those in bookmark names.

Remote bookmark names use `bookmark@remote` syntax, e.g. `main@origin`. To test someone else's branch without tracking it locally, you can often run `jj new their-branch@origin`. To keep following it as a local bookmark, use `jj bookmark track`.

---

<a id="stacked-work"></a>

## Stacked work

jj is excellent at stacked changes because commits are easy to rewrite and descendants move along. A stack is a line, or small graph, of commits where later commits depend on earlier ones. Git can represent this perfectly well. Git's command-line UX makes updating the middle of the stack feel dangerous. jj makes it routine.

The habit to build: keep each commit reviewable. One reason, one message, one coherent diff. If you discover that commit 3 needs a change while you are working on commit 6, edit or create a fix near commit 3, squash it there, and let jj rebase commits 4-6 automatically.

```bash
jj new main
# work on base refactor
jj commit -m "refactor: isolate storage layer"
# work on feature depending on it
jj commit -m "feat: add import pipeline"
# work on tests/docs
jj commit -m "test: cover import pipeline"

jj log
jj git push -c @-          # push top change/stack with generated bookmark
```

To edit an earlier commit:

```bash
jj edit 'description("refactor")'
# make fix
jj new @+                  # or jj edit the stack tip when done
```

Useful stack-ish revsets:

```bash
jj log -r 'main..@'                  # ancestors of @ not in main
jj log -r 'remote_bookmarks()..@'    # local stack not on any remote
jj log -r 'reachable(@, mutable())'  # connected mutable work around @
```

For serious stacked PR management on GitHub, investigate external tools like `jj-stack`/`jj-stacked`, but learn native bookmarks first. Native jj understands the stack; hosting platforms usually understand independent branch heads and PR base branches. Stack helper tools bridge that social/hosting gap, not a core jj limitation.

---

<a id="history-editing"></a>

## History editing without fear

jj treats rewriting as ordinary. These are normal commands, not emergency tools. The working assumption is that local mutable history is yours to improve until you publish it or until it becomes part of an immutable base like `main`, a tag, or a remote bookmark your config treats as immutable.

This is the biggest psychological difference from Git. In Git, rewriting often feels like a dangerous alternate mode because you can strand commits or confuse branch reflogs. In jj, rewriting is an operation recorded in the operation log; descendants and bookmarks generally follow rewrites in the way you meant.

These are normal commands, not emergency tools:

```bash
jj describe -m "better message"   # edit description of @
jj rebase -b REV -d DEST          # move branch/stack rooted at REV onto DEST
jj squash -i                      # move selected changes into parent
jj split                          # split a commit interactively
jj abandon REV                    # delete/abandon a commit
```

If you regret it:

```bash
jj undo
jj redo
jj op log
jj op show <op>
jj op restore <op>
```

Rule of thumb: if you are trying a scary rewrite, run `jj op log --limit 1` first so you can easily see the restore point. After experimenting, `jj op log` will show every operation as a repo-wide snapshot of reference/commit visibility changes. `jj undo` is the convenient one-step version; `jj op restore` is the time machine.

Two caveats: the operation log is local, and it does not replace backups. Also, restoring an old operation changes your repo view; it does not coordinate with teammates or remotes. Treat it as local recovery.

---

<a id="conflicts"></a>

## Conflicts

jj can store conflicted states in commits. That means a rebase/merge can complete even if conflicts exist; you resolve them by editing the conflicted commit, not by running `rebase --continue`. This is strange at first if you are used to Git stopping the whole repository in a temporary rebase/merge state.

In jj, a conflict is data in a commit. The commit may be visible in `jj log`, descendants may already have been rebased on top of it, and you can choose when to resolve it. When you edit the conflicted commit and remove the conflict markers from files, that commit evolves into a resolved version and descendants update accordingly.

Typical flow:

```bash
jj rebase -b my-work -d main
jj status                 # reports conflicts if any
jj log -r 'conflicts()'   # find conflicted commits
jj edit <conflicted-rev>
# fix files
jj status
jj diff
jj new                    # continue with a clean child when done
```

Why this matters:

- You can postpone conflict resolution.
- Descendants can still be rebased.
- Conflict resolutions are part of the commit's diff and can be rebased later.
- There is no special `--continue` state machine to get stuck in.

If tools dislike jj conflict markers, configure Git-style markers:

```bash
jj config set --user ui.conflict-marker-style git
```

---

<a id="revsets"></a>

## Revsets: jj's query language

Most jj commands accept a **revset**, a small expression language for selecting commits. Git has many separate flags for revision selection (`--branches`, `--not`, `A..B`, `--author`, pathspecs, pickaxe, etc.). jj consolidates much of that into one composable language.

You do not need to master revsets on day one. Learn `@`, `@-`, `main..@`, and `remote_bookmarks()..@` first. The payoff comes later, when the same expression style works for `log`, `diff`, `show`, `rebase`, `abandon`, and more.

Core syntax:

```bash
@                  # working-copy commit
@-                 # parent of @
@+                 # children of @
::@                # ancestors of @, including @
main..@            # ancestors of @ that are not ancestors of main
A::B               # descendants of A that are ancestors of B
x | y              # union
x & y              # intersection
x ~ y              # x minus y
```

Useful functions:

```bash
root()                         # root commit
all()                          # all visible commits
mine()                         # authored by current user
bookmarks()                    # local bookmark targets
remote_bookmarks()             # remote bookmark targets
conflicts()                    # commits with conflicts
description(regex:'fix|feat')  # match messages
files('src')                   # commits changing paths under src
latest(mine(), 10)             # newest 10 of your commits
```

Examples:

```bash
jj log -r 'main..@'
jj log -r 'mine() & remote_bookmarks()..'
jj log -r 'description(regex:"TODO|WIP")'
jj diff -r '@-'
jj show 'latest(mine(), 1)'
```

Shell quote revsets containing `|`, `&`, `~`, `*`, or parentheses. If a revset behaves oddly, first suspect your shell ate part of it. Single quotes are usually safest on Unix shells.

Read complex revsets inside-out:

```bash
(mine() & remote_bookmarks()..)
# commits authored by me
# intersected with commits not reachable from any remote bookmark
# => my local unpublished commits

(main..@)::
# commits in my current line after main
# plus their descendants
# => useful for seeing a whole local stack/context
```

---

<a id="config-aliases"></a>

## Suggested config and aliases

Keep this minimal until you understand defaults. jj is very configurable, especially log templates, colors, diff editors, merge tools, revset aliases, and default revsets. Resist the urge to recreate your entire Git alias file immediately; first learn where jj's defaults differ from Git.

```bash
# Optional: turn off pager if you prefer raw terminal output
jj config set --user ui.paginate never

# Common shorthand aliases. Values are TOML arrays.
jj config set --user aliases.l '["log", "-r", "::@"]'
jj config set --user aliases.ll '["log", "-r", "all()"]'
jj config set --user aliases.tug '["bookmark", "move", "main", "--to", "@-"]'
```

Equivalent `jj config edit --user` form, plus a useful Git-style `HEAD` revset alias:

```toml
[aliases]
l = ["log", "-r", "::@"]
ll = ["log", "-r", "all()"]
tug = ["bookmark", "move", "main", "--to", "@-"]

[revset-aliases]
'HEAD' = '@-'
```

Notes:

- `trunk()` is a built-in revset alias intended to resolve the default trunk commit/bookmark. In simple repos, `main` may be enough; in mixed remote setups, `trunk()` can be clearer.
- Do not over-alias commands before learning the native vocabulary. The model matters.
- If you use a graphical merge/diff tool, configure `ui.diff-editor` for `jj split` / `jj squash -i` and `ui.merge-editor` for conflict resolution. The built-in tools are good enough to start, but GUI diff tools can make partial selection less tiring.
- Consider repo-specific config for remote defaults if you use a fork: fetch from `upstream`, push to `origin`. That mirrors common GitHub contribution setups.

---

<a id="faq"></a>

## FAQ / gotchas for Git users

### Is jj replacing Git in my project?

Usually no. In day-to-day use, jj stores commits in a real Git repository. Other collaborators can keep using Git. You can often use Git tooling unchanged.

### Can I mix `git` and `jj` commands?

Yes in colocated Git-backed repos. If you do something with Git, the next jj command imports/syncs Git state. Try not to mix during one delicate rewrite unless you know what state each tool sees.

A pragmatic approach: use jj for local editing/history operations, and fall back to Git for one-off commands you already know (`git grep`, unusual submodule workflows, project-specific release scripts, etc.). Afterward, run `jj status` so jj imports the Git-side changes and shows you the graph it sees.

### Where is the staging area?

There is none. Use:

- `jj split` to split current work into commits.
- `jj squash -i` to move selected hunks into the parent.
- `jj restore` to copy/revert file states between revisions.

### What is the equivalent of `git commit`?

`jj commit -m "msg"` describes the current working-copy commit and creates a new empty working-copy commit on top. The commit you just made is normally `@-`.

If you want to set or edit the message without moving on, use `jj describe`. That distinction is useful during longer work: describe the intent early, keep editing, then `jj new` or `jj commit` only when you want a new change on top.

### Why did my commit hash change after editing the message or files?

Because Git commit IDs hash content and metadata. jj preserves the **change ID** across rewrites so humans and commands can keep referring to the evolving change.

### Why did `main` not move after I committed?

Bookmarks are not active branches. Move it explicitly if needed:

```bash
jj bookmark move main --to @-
```

Most feature work should not move `main`; create/push a feature bookmark or push a change with `jj git push -c @-`.

### Why does `jj git push --all` say nothing changed?

It pushes bookmarks, not every anonymous local commit. Create/move a bookmark or use:

```bash
jj git push -c @-
```

### What is `@-` and why does every push example use it?

After `jj commit`, `@` is a fresh empty working commit. Your completed work is its parent, `@-`. Pushing `@` would push the empty work-in-progress commit; pushing `@-` pushes the completed change.

### Where is `git pull`?

Use two explicit steps:

```bash
jj git fetch
jj rebase -b @ -d main
```

Adjust `-b`/`-d` for your stack and destination. This separation avoids Git's overloaded fetch+merge/rebase behavior.

### Why does `jj log` hide most remote history?

By default it shows local work and useful context, not every remote commit. Use explicit revsets:

```bash
jj log -r 'all()'
jj log -r '::@'
jj log -r 'remote_bookmarks()..@'
```

### How do I checkout a branch?

Usually you create a new working commit on top of a bookmark:

```bash
jj new main
jj new feature
jj new feature@origin
```

If you need to edit an existing commit directly:

```bash
jj edit REV
```

### How do I make a merge commit?

Create a new commit with multiple parents:

```bash
jj new left right
jj describe -m "merge left and right"
```

### How do I abandon/reset bad local work?

```bash
jj abandon REV       # delete a commit/change from visible history
jj restore path      # restore path in @ from parent
jj undo              # undo last operation
```

### Can I lose work?

It is harder than in Git. It is still possible. Local jj operation history is not a remote backup. Pushing Git commits does not push jj's operation log. Use `jj op log`/`jj undo` for local recovery and remotes/backups for disaster recovery.

Also remember that some data never belongs in version control: ignored files, generated artifacts, local databases, secrets, and editor swap files may not be captured by jj. jj's safety net is excellent for tracked repository state, not for every byte under your project directory.

### Why are there empty commits everywhere?

An empty `@` is normal: it is your current place to type future work. Empty commits can also be useful placeholders for planned changes or merge structure. jj will label them as empty so you can tell whether a change actually modifies files.

### What is `jj evolog`?

`jj evolog` shows how a change evolved over time: previous hidden commit IDs for the same change ID, message edits, content edits, and so on. If `jj log` shows the current visible graph, `jj evolog` is the per-change archaeology tool. It is especially helpful when you think, "I had a different version of this change 20 minutes ago."

### How do I see only unpublished work?

Start with:

```bash
jj log -r 'remote_bookmarks()..@'
```

That means: ancestors of `@` that are not reachable from any remote bookmark. Depending on your repo, you may prefer `main..@`, `trunk()..@`, or a custom revset alias.

### What should I do with `.jj/`?

Do not commit it. In colocated repos, `.jj/` is local metadata beside `.git/`. It is not meant to be shared as project content. If you frequently run tools that show untracked dot-directories outside Git ignore handling, add `.jj/` to your global ignore.

### Should my whole team switch?

No need. Individual adoption works well because of Git compatibility. Team adoption mainly requires agreeing on bookmark/PR conventions and teaching the "no current branch" rule.

### When should I still use Git?

Use Git when a tool or workflow has no jj equivalent yet, or when following project-specific Git instructions. In a colocated repo, jj will sync afterward.

---

<a id="further-reading"></a>

## Further reading

Primary sources used:

- [Jujutsu for everyone](https://jj-for-everyone.github.io/) and its [source repo](https://github.com/jj-for-everyone/jj-for-everyone.github.io): beginner tutorial; good structure, written for non-Git users.
- [Steve Klabnik's Jujutsu Tutorial](https://steveklabnik.github.io/jujutsu-tutorial/) and [source repo](https://github.com/steveklabnik/jujutsu-tutorial): narrative tutorial for Git users.
- Official docs: [Tutorial](https://docs.jj-vcs.dev/latest/tutorial/), [Git comparison](https://docs.jj-vcs.dev/latest/git-comparison/), [Git command table](https://docs.jj-vcs.dev/latest/git-command-table/), [Git compatibility](https://docs.jj-vcs.dev/latest/git-compatibility/), [GitHub/GitLab workflow](https://docs.jj-vcs.dev/latest/github/), [Bookmarks](https://docs.jj-vcs.dev/latest/bookmarks/), [Revsets](https://docs.jj-vcs.dev/latest/revsets/), [Conflicts](https://docs.jj-vcs.dev/latest/conflicts/), [FAQ](https://docs.jj-vcs.dev/latest/FAQ/).
- Chris Krycho, [jj init](https://v5.chriskrycho.com/essays/jj-init/): strong explanation of why jj is worth trying from existing Git repos.
- Kristoffer Balintona, [jj workflows and the operation log](https://kristofferbalintona.me/articles/jujutsu-jj-vcs-workflows-and-the-convenience-of-its-operation-log/): practical workflow notes and why operation-log-based recovery changes how you experiment.
