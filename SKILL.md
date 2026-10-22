---
name: pr-review
description: >
  Review a code change and return a structured, evidence-backed, prioritised
  verdict. Use when the user asks to review a PR, review changes, review a diff,
  review a commit or branch, or pastes a GitHub PR URL and asks for a verdict.
  Covers GitHub PRs, base-branch comparisons, uncommitted changes and single
  commits. Fans out across read-only reviewer subagents scaled to the size of the
  diff, filters noise files, and reports findings by priority with an overall
  correctness verdict. For landing a PR after review, use pr-watch.
---

# PR Review

Review a change and return a tight, prioritised, evidence-backed verdict. The flow:
gather the diff, drop noise files, scale a fan-out
of read-only reviewers to the size of the change, then consolidate their findings
into one verdict. OpenCode ships its own lightweight `/review` command; this skill
is the deeper, multi-reviewer path. Invoke it as `/pr-review`.

The reviewers do the reading. You own the diff, the grouping, the dispatch and the
final write-up. Never review the diff yourself in place of the reviewers.

## 1. Pick the target

Work out what to review before anything else.

- **GitHub PR URL or `owner/repo#number`** (also accept bare `/pr-review 1234` in a
  repo that has a PR for the current branch). Strip `/changes`, `/files`, `/commits`.
- Otherwise ask which mode, with the `question` tool if you have it:
  1. Review against a base branch (PR style)
  2. Review uncommitted changes (staged + unstaged)
  3. Review a specific commit
  4. Custom review instructions
- If the session is non-interactive and no target was given, default to
  uncommitted changes.
- Extra words in the request are review instructions, not part of the target. Keep
  them and pass them through as additional instructions.

## 2. Gather the diff

Run independent commands in parallel. Use the repository the change belongs to,
usually an existing worktree under `~/repos/`. Do not switch branches or stash in a
worktree another agent is using.

- **GitHub PR**
  - `gh pr view <n> --repo <owner/repo> --json title,body,author,additions,deletions,changedFiles,baseRefName,url`
  - `gh pr diff <n> --repo <owner/repo>`
  - `gh pr checks <n> --repo <owner/repo>`
  - `gh pr view <n> --repo <owner/repo> --json reviews,reviewDecision`
  - `gh api repos/<owner/repo>/pulls/<n>/comments` for inline threads.
  - Existing reviews, `reviewDecision` and CI checks are context only. They are not
    evidence and must not influence the verdict. See "Trust only this review".
- **Base branch (PR style)**
  - `git branch -a` for the branch list, then `git diff <base>...HEAD`.
  - Compare against `origin/<base>` where it exists.
- **Uncommitted**
  - `git status --short`, `git diff`, `git diff --cached`. Concatenate staged and
    unstaged. In a `jj` repo use `jj --ignore-working-copy diff --git`.
- **Commit**
  - `git log --oneline -20` to choose, then `git show <sha>` (use `--format=` for
    diff only).
- **Custom**
  - Take the diff from uncommitted changes; if there is none, pass the instructions
    alone to a single reviewer with no diff.

If the diff is empty, say so and stop. A diff that is entirely filtered files is
also empty for review purposes.

## 3. Build the file table

Split the diff on `^diff --git` into per-file blocks. For each block count added
(`+`, not `+++`) and removed (`-`, not `---`) lines and keep the raw block.

Drop these, and report what you dropped with the reason:

- lock files (`*.lock`, `*-lock.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `Gemfile.lock`, `poetry.lock`, `composer.lock`, `flake.lock`)
- minified (`*.min.js`, `*.min.css`), generated (`*.generated.*`), snapshots (`*.snap`), source maps (`*.map`)
- build output (`dist/`, `build/`, `out/`), vendor (`node_modules/`, `vendor/`)
- images (`.png .jpg .jpeg .gif .ico .webp .avif`), fonts (`.woff .woff2 .ttf .eot .otf`), binaries (`.pdf .zip .tar .gz .rar .7z`)

## 4. Scale the fan-out

Let `lines` be total added plus removed, and `files` the number of reviewable files.

| lines | files | reviewers |
| --- | --- | --- |
| < 100 | or ≤ 2 files | 1 |
| < 500 | | min(2, files) |
| < 2000 | | min(4, ceil(files / 3)) |
| < 5000 | | min(8, ceil(files / 2)) |
| ≥ 5000 | | min(16, files) |

Group files by locality: same directory or module together, related functionality
together, tests with the implementation files they cover.

**Large diff fallback.** If the diff exceeds 50,000 lines or 20 files, do not pass
the whole thing. Give each reviewer a preview of the first
`max(5, floor(100 / files))` lines per assigned file and tell it the full diff is
too large. Use the non-preview form otherwise.

## 5. Dispatch the reviewers

Render one request per reviewer from `references/request-templates.md` (standard,
custom or headless). Each request carries the assigned files, the diff, and any
additional instructions.

Then spawn one subagent per request in a single turn so they run in parallel.

- Use the `reviewer` subagent by default. It already carries the review criteria as
  its system prompt, so pass only the assignment. Do not paste
  `references/reviewer.md` on top of it.
- If `reviewer` is unavailable, use the `explore` agent and include the contents of
  `references/reviewer.md` in the prompt. As a last resort use `general` and state
  plainly that the task is read-only.
- The dedicated reviewer has shell denied, so the diff must be in the request.
  Never ask a reviewer to fetch the diff itself. If you use `explore` or `general`,
  still put the diff in the prompt.
- For PRs, tell the reviewer to use the provided diff, not local `git show`.
- If the user asked for a security pass, or the change touches auth, input
  handling, secrets, network or permissions, also dispatch the `security-reviewer`
  subagent over the same scope, or `explore` with
  `references/security-reviewer.md`.

Reviewers return findings and a verdict. Do not let one reviewer review files it
was not assigned.

## 6. Consolidate

Collect every finding, drop duplicates of the same root cause, and order by
priority then confidence. Keep file and line references. If two reviewers disagree
on correctness, read the contested code yourself before deciding.

## Verify, don't trust

The diff is not the only evidence. Independently confirm any factual claim the
change relies on: config keys exist and are consumed, defaults are what is claimed,
an API behaves as stated. For dependencies or forks, check the deployed version or
commit, not upstream `main`. Prefer reading a clone under `~/repos/`, otherwise
`gh api .../contents/<path>` or `gh api search/code`. Surface open reviewer
questions rather than restating a concern someone already raised.

## Trust only this review

Existing approvals, `reviewDecision`, green CI, bot reviews (CodeRabbit, Copilot,
and other automated reviewers) and prior human reviews are context, not evidence. Never treat
them as reassurance. Never cite them as a reason to approve, never soften a finding
because others already approved, and never lower the bar to match a PR that is
already approved. Review a PR with ten approvals exactly as one with none. Do not
repeat or defer to another reviewer's conclusion. This skill's own reviewers are
the only source for the verdict, and the verdict stands on its own regardless of
what anyone else concluded.

## Output

Dense, no filler, no restating the diff line by line.

### Summary
1–2 sentences: what the change does and why.

### Impact
What changes at runtime if this merges: blast radius, reversibility, the realistic
failure mode, who or what is affected. The "so what", not a re-description.

### Findings
Only issues that met the criteria. For each: priority, a one-line title, the
`path:line` reference, the trigger and impact, and confidence. Group by priority,
P0 first. Say "no blocking findings" when that is the honest answer.

### Security
One line, including "no security impact" when true.

### Verdict
`correct` or `incorrect`, with a confidence and a one-line reason, then either
**Approve** or **Add comments** with the numbered list, each marked blocking or
non-blocking. A clean change gets a one-line approve. Base the verdict only on this
skill's reviewers. Do not mention existing approvals, `reviewDecision` or CI status
as support for the verdict, and do not let them change it.

## Safety and honesty

- Read-only throughout. Never edit files, build, push or resolve threads here.
- Report what you could not verify instead of implying you checked it.
- Reference code as clickable `path:line` links.

## Reference files

- `references/reviewer.md` — the standard reviewer instructions handed to each subagent.
- `references/security-reviewer.md` — the security pass.
- `references/request-templates.md` — the request templates to render.
