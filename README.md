# pr-review

A skill for [OpenCode](https://opencode.ai) that reviews a code change the way a careful human would. It reads the diff, drops the noise, fans out read-only reviewer subagents scaled to the size of the change, then turns their findings into one verdict.

It exists because a single agent reviewing a 2,000 line diff does two things badly. It skims, and it anchors on the first problem it finds. Several reviewers working in parallel on separate files do neither.

## What it does

- Takes a GitHub PR, a base branch comparison, a commit, or uncommitted changes.
- Drops lock files, generated code, snapshots, build output, binaries and images, and reports what it dropped.
- Scales the number of reviewers from 1 to 16 by lines and files changed, grouped by locality.
- Dispatches every reviewer in one turn so they run in parallel.
- Consolidates findings by root cause, priority and confidence.
- Returns a summary, an impact line, findings with `path:line`, a security line, and a `correct` or `incorrect` verdict.

The reviewers do the reading. The agent owns the diff, the grouping, the dispatch and the write up.

## Install

Copy the skill into your skills directory.

```
git clone https://github.com/safety-adam/opencode-skill-pr-review.git
mkdir -p ~/.config/opencode/skills
cp -r opencode-skill-pr-review ~/.config/opencode/skills/pr-review
```

Claude Code reads `~/.claude/skills/` instead. Either way the folder name must match the `name` in the frontmatter, so `pr-review`.

Then invoke it.

```
/pr-review 1234
/pr-review https://github.com/owner/repo/pull/1234
```

## Requirements

- [OpenCode](https://opencode.ai) or Claude Code with skills enabled.
- The [`gh`](https://cli.github.com) CLI, authenticated, for GitHub PRs.
- The bundled `reviewer` and `security-reviewer` subagents are optional. The skill falls back to a generic agent with the same instructions passed in the prompt.

The two subagents are read only by design. They carry edit, shell and subagent permissions denied, and they get the diff in the prompt rather than fetching it themselves. That keeps a reviewer from wandering, and it means a review can never change the code it is reviewing.

## How the fan out scales

| Lines changed | Files | Reviewers |
| --- | --- | --- |
| under 100 | or 2 or fewer files | 1 |
| under 500 | | min(2, files) |
| under 2000 | | min(4, ceil(files / 3)) |
| under 5000 | | min(8, ceil(files / 2)) |
| 5000 or more | | min(16, files) |

Above 50,000 lines or 20 files the skill stops passing the whole diff and gives each reviewer a preview per assigned file.

## Trust only this review

Existing approvals, green CI and other bot reviews are context, not evidence. A change with ten approvals gets the same scrutiny as one with none. This is the part that matters most in practice, because review pressure is usually social rather than technical.

## Files

- `SKILL.md` is the procedure.
- `references/reviewer.md` is the standard reviewer brief.
- `references/security-reviewer.md` is the security pass, which traces attacker controlled input to a dangerous sink.
- `references/request-templates.md` holds the request templates.

## Pairs with

[opencode-skill-pr-watch](https://github.com/safety-adam/opencode-skill-pr-watch) takes a PR from reviewed to merged and verified.

## Licence

MIT.
