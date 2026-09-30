---
name: nuvio-push-and-commit
description: Commit and push pending changes in the nuvio-specs workspace and any dirty submodules under repos/, with a message written from the actual diff. Use when the user runs /nuvio-push-and-commit or asks to commit and push work in this project.
disable-model-invocation: true
---

Commit and push the pending changes in this workspace, writing a proper commit message from the real diff.

**Input**: optional free text after the command (e.g. a hint for the message or a scope such as `nuviodesktop`). If given, use it to limit which repo or files are committed and to inform the message.

## Repos involved

- The workspace root (`nuvio-specs`, branch `development`): OpenSpec artifacts under `openspec/`, `.claude/`, `CLAUDE.md`, `.gitmodules`.
- Submodules under `repos/` (`NuvioTV`, `NuvioMobile`, `NuvioDesktop`, `plexio`, `YouTubio`). Each is its own git repo with its own remote and branch.

Commit each repo separately, **submodules first**, then the workspace. Run every git command for a submodule from inside that submodule's directory.

## Steps

1. **Survey.** For the workspace root and each directory in `repos/`, run `git status --short` and `git branch --show-current`. Skip repos with nothing to commit. Report what is dirty before touching anything.

2. **Decide what to stage, per repo.**
   - Stage only files that belong to the work being committed. Add paths explicitly; never `git add -A` or `git add .`.
   - Do not stage: `local.properties`, build output, secrets or `.env` files, editor files, or untracked files you did not create or that the user did not mention (for example another change's folder or a repo-level `CLAUDE.md`). List anything skipped so the user can ask for it.
   - A submodule pointer bump (`M repos/<name>` in the workspace) is only staged after that submodule's commit exists, and only when the user's request covers it. Say so when a pointer is left unstaged.
   - `.gitmodules` changes are staged only if the user asked.

3. **Write the message from the diff.** Read `git diff --staged` (and `git log --oneline -5` to match the repo's style). Never commit with a placeholder, a generic message ("update", "changes") or one written before reading the diff.
   - Subject: imperative, at most about 72 characters, describing what changed and why it matters. Follow the repo's existing convention, e.g. Conventional Commits (`feat(streams): ...`, `fix(windows): ...`) in the Nuvio submodules, and plain sentence style in the workspace root.
   - Body (when the change is more than trivial): short paragraphs or bullets on what and why, wrapped near 72 columns.
   - Describe only what is in the staged diff. Mention unverified parts (untested native code, skipped tests) rather than implying they were checked.
   - End the message with the attribution line from the session's system-reminder, if one is present, separated by a blank line.
   - Pass the message with a heredoc so newlines are preserved.

4. **Commit.** Run `git commit` in each repo. If a hook fails, fix the cause and make a new commit; do not use `--no-verify` and do not amend unless the user asks.

5. **Push.** For each repo that received a commit:
   - Confirm the branch with `git branch --show-current`. Do not push from a detached HEAD.
   - Push the current branch to its upstream: `git push`. If there is no upstream, use `git push -u origin <branch>`.
   - Push to `origin` only. Never push to `upstream` remotes, never force-push, and never push to `main`/`master` of a repo unless it is already the current branch and the user asked for that.
   - Push submodules before the workspace so a pointer bump never references an unpushed commit.
   - If a push is rejected (non-fast-forward), stop and report; do not force. Offer `git pull --rebase` and wait for the user.

6. **Report.** For each repo show: branch, commit hash and subject, push result, and anything left uncommitted or unstaged with the reason.

## Guardrails

- Never commit or push without reading the diff and writing a message specific to it.
- Never use `--force`, `--no-verify`, `git reset --hard`, or `git stash` in this flow.
- If nothing is dirty, say so and stop; do not create empty commits.
- If a repo's branch looks wrong for the work (for example, submodule on `Dev` while the workspace is on `development`), continue on the branch that is checked out and mention it in the report.
- This skill is project-local (`.claude/skills/` in nuvio-specs) and is not meant to be copied to user-level skills.
