# Request templates

Render one of these per reviewer and pass it in the subagent prompt, together with
the contents of `reviewer.md` (or `security-reviewer.md` for a security pass).

## Standard request

```
## Code Review Request
### Mode
{one-line description: "PR owner/repo#n", "Reviewing changes between `main` and `feature`",
"Reviewing uncommitted changes (staged + unstaged)", "Reviewing commit `<sha>`", ...}

### Changed Files ({N} files, +{added}/-{removed} lines)
| File | +/- | Type |
| --- | --- | --- |
{for each file} {path} | +{a}/-{r} | {ext}

### Excluded Files ({N})
- `{path}` (+{a}/-{r}) — {reason}

### Assigned Files
{the subset of files this reviewer owns}

### Distribution
You are one of {k} reviewers running in parallel. Focus only on the assigned files
and the diff below.

### Reviewer Instructions
{omit this section when dispatching the dedicated `reviewer` subagent, whose system
prompt already carries the criteria; otherwise insert the contents of
references/reviewer.md}

### Diff
<diff>
{raw diff, or per-file previews when the full diff is too large}
</diff>

### Additional Instructions
{the user's extra words, if any}
```

When the full diff is too large, replace the Diff section with per-file previews of
the first `max(5, floor(100 / files))` lines and say the full diff is too large.
For a PR, add: use the provided diff and `gh pr diff <n> --repo <owner/repo>`;
never use local `git diff` or `git show` for PR content.

## Custom request

Use when the user supplied only instructions and there is no diff to attach.

```
## Code Review Request
Mode: custom instructions.

### Custom Instructions
{instructions}
```

Then dispatch a single reviewer with the instructions and no diff, letting it read
what the instructions ask for.

## Headless request

Use when there is no interactive UI and the user asked for a review of recent
changes with no other target.

```
## Code Review Request
Mode: headless review request.
Distribution: exactly one reviewer for recent code changes.

### Focus
{focus, if any}
```

## Grouping

Assign files by locality, not round-robin: same directory or module together,
related functionality together, tests with the implementation files they cover.
Each reviewer gets a contiguous, coherent slice. Never hand a reviewer a file it
cannot read the consumers of.
