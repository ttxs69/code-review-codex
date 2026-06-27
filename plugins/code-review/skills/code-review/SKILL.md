---
name: code-review
description: Maximum-effort code review optimized for recall. Reviews a git diff (working tree, branch range, pull request, or file path) using 10 independent finder angles, single-vote verification, and a final gap sweep, returning at most 15 ranked findings. Use when you want the most thorough possible review of pending changes and catching real bugs matters more than avoiding false positives.
allowed-tools: Bash(gh search:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git blame:*), Bash(find:*), Bash(cat:*), Read
disable-model-invocation: false
---

`max effort → 5+5 angles × 8 candidates → 1-vote verify → sweep → ≤15 findings`

You are reviewing for **recall** at maximum effort: catch every real bug. At
this level, catching real bugs matters more than avoiding false positives — a
missed bug ships. Err on the side of surfacing.

## Phase 0 — Gather the diff

Determine the review target from the user's argument or context:

- **Pull request** (most common): a PR number, URL, or branch name → use
  `gh pr view <PR>` to identify the repo and the head/base SHAs, then run
  `gh pr diff <PR>` to fetch the unified diff. Use `gh` for everything PR-
  related; do not web-fetch.
- **Branch range or working tree**: run `git diff @{upstream}...HEAD`. If
  there is no upstream, fall back to `git diff main...HEAD` (or
  `git diff HEAD~1`). If the working tree has uncommitted changes, also run
  `git diff HEAD` and include them in scope — the review often runs before
  the commit.
- **File path argument**: scope the diff to that path
  (`git diff @{upstream}...HEAD -- <path>`), but still read enclosing
  functions for hunks you surface.

Treat the resulting diff as the review scope. Bugs in unchanged lines of a
touched function are in scope — the PR re-exposes or fails to fix them.

## Phase 1 — Find candidates (5 correctness + 3 cleanup + 1 altitude + 1 conventions, up to 8 each)

Run **10 independent finder angles** by spawning subagents via `spawn_agent`.
Use `agent_type: "default"` for each. Set `reasoning_effort: "medium"` for
correctness angles A–E and `reasoning_effort: "low"` for cleanup/altitude/
conventions. Spawn them **in parallel in a single response** so they run
concurrently. Pass each subagent the diff plus enough enclosing context to
work (file paths, surrounding functions when hunks need them). Each returns
**up to 8 candidate findings** as
`{file, line, summary, failure_scenario}`. Do NOT let one angle's conclusions
suppress another's — if two angles flag the same line for different reasons,
record both.

### Angle A — line-by-line diff scan

Read every hunk in the diff, line by line. Then Read the enclosing function
for each hunk — bugs in unchanged lines of a touched function are in scope
(the PR re-exposes or fails to fix them). For every line ask: what input,
state, timing, or platform makes this line wrong? Look for inverted/wrong
conditions, off-by-one, null/undefined deref, missing `await`, falsy-zero
checks, wrong-variable copy-paste, error swallowed in catch, unescaped regex
metachars.

### Angle B — removed-behavior auditor

For every line the diff DELETES or replaces, name the invariant or behavior
it enforced, then search the new code for where that invariant is
re-established. If you can't find it, that's a candidate: a removed guard,
a dropped error path, a narrowed validation, a deleted test that was
covering a real case.

### Angle C — cross-file tracer

For each function the diff changes, find its callers (Grep for the symbol)
and check whether the change breaks any call site: a new precondition, a
changed return shape, a new exception, a timing/ordering dependency. Also
check callees: does a parallel change in the same PR make a call unsafe?

### Angle D — language-pitfall specialist

Scan for the classic pitfalls of the diff's language/framework — for example:
JS falsy-zero, `==` coercion, closure-captured loop var; Python mutable
default args, late-binding closures; Go nil-map write, range-var capture;
SQL injection; timezone/DST drift; float equality. Flag any instance the
diff introduces.

### Angle E — wrapper/proxy correctness

When the PR adds or modifies a type that wraps another (cache, proxy,
decorator, adapter): check that every method routes to the wrapped instance
and not back through a registry/session/global — e.g. a caching provider
holding a `delegate` field that resolves IDs via `session.get(...)` instead
of `delegate.get(...)` will re-enter the cache or recurse. Also check that
the wrapper forwards all the methods the callers actually use.

### Reuse

Flag new code that re-implements something the codebase already has — Grep
shared/utility modules and files adjacent to the change, and name the
existing helper to call instead.

### Simplification

Flag unnecessary complexity the diff adds: redundant or derivable state,
copy-paste with slight variation, deep nesting, dead code left behind. Name
the simpler form that does the same job.

### Efficiency

Flag wasted work the diff introduces: redundant computation or repeated I/O,
independent operations run sequentially, blocking work added to startup or
hot paths. Also flag long-lived objects built from closures or captured
environments — they keep the entire enclosing scope alive for the object's
lifetime (a memory leak when that scope holds large values); prefer a
class/struct that copies only the fields it needs. Name the cheaper
alternative.

### Altitude

Check that each change is implemented at the right depth, not as a fragile
bandaid. Special cases layered on shared infrastructure are a sign the fix
isn't deep enough — prefer generalizing the underlying mechanism over adding
special cases.

### Conventions (AGENTS.md)

Find the AGENTS.md files that govern the changed code: the user-level
`~/.codex/AGENTS.md` (if present), the repo-root AGENTS.md, plus any
AGENTS.md in a directory that is an ancestor of a changed file (a
directory's AGENTS.md only applies to files at or below it). Read each one
that exists, then check the diff for clear violations of the rules they
state.

Only flag a violation when you can quote the exact rule and the exact line
that breaks it — no style preferences, no vague "spirit of the doc"
inferences. In the finding, name the AGENTS.md path and quote the rule so
the report can cite it. If no AGENTS.md applies, return nothing for this
angle.

Cleanup, altitude, and conventions candidates use the same
`file`/`line`/`summary` shape; in `failure_scenario`, state the concrete
cost (what is duplicated, wasted, harder to maintain, or which AGENTS.md
rule is broken) instead of a crash. Correctness bugs always outrank
cleanup, altitude, and conventions findings when the output cap forces a
cut.

## Phase 2 — Verify (1-vote, 3-state)

Dedup candidates that point at the same line/mechanism, keeping the one with
the most concrete failure scenario. For each remaining candidate, spawn
**one verifier** subagent (`agent_type: "default"`,
`reasoning_effort: "medium"`): give it the diff, the relevant file(s), and
the candidate, and have it return exactly one of:

- **CONFIRMED** — can name the inputs/state that trigger it and the wrong
  output or crash. Quote the line.
- **PLAUSIBLE** — mechanism is real, trigger is uncertain (timing, env,
  config). State what would confirm it.
- **REFUTED** — factually wrong (code doesn't say that) or guarded
  elsewhere. Quote the line that proves it.

Spawn verifiers **in parallel** in a single response. Keep candidates where
the vote is CONFIRMED or PLAUSIBLE.

This is recall mode — a single non-REFUTED vote carries the finding. Do NOT
drop on uncertainty.

## Phase 3 — Sweep for gaps

Spawn **one more finder** subagent (`agent_type: "default"`,
`reasoning_effort: "medium"`) as a fresh reviewer who has the verified list.
Re-read the diff and enclosing functions looking ONLY for defects not
already listed. Do not re-derive or re-confirm anything already there — the
job is gaps. Focus on what the first pass tends to miss: moved/extracted
code that dropped a guard or anchor; second-tier footguns (dataclass default
evaluated once, `hash()` non-determinism, lock-scope shrink, predicate
methods with side effects); setup/teardown asymmetry in tests; config
defaults flipped.

Surface **up to 8 additional candidates**, each naming a defect not already
on the list. If nothing new, return an empty sweep — do not pad.

## Phase 4 — Compose the report

Take the surviving findings (CONFIRMED + PLAUSIBLE from Phase 2, plus any
non-duplicate additions from Phase 3) and rank them most-severe first. Cap
the final list at **15** findings. If more than 15 survive, keep the 15
most severe. If nothing survives verification, the report is empty.

Each finding carries: `file`, `line`, `summary`, `failure_scenario`.

### If the target was a pull request

1. Compose a GitHub-flavored Markdown comment. For each finding include a
   link of the form
   `https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>`
   where the full SHA is the head commit of the PR (resolve via
   `git rev-parse HEAD` inside the repo, or `gh pr view <PR> --json headRefOid`).
   Provide at least one line of context before and after — center the link
   on the line you cite. Repo name must match the repo under review.
2. Filter out linter/typechecker/compiler issues (formatting, missing
   imports, type errors) — those will be caught by CI.
3. Filter out pre-existing issues not introduced by the PR.
4. Filter out pedantic nitpicks and general quality issues unless explicitly
   required by an AGENTS.md cited in the finding.
5. Filter out real issues on lines the PR did not modify, unless the PR
   touched a surrounding function and re-exposed them (then cite the
   surrounding function's lines, not the unchanged line).
6. Post the comment with
   `gh pr comment <PR> --body-file -` (write the body to a temp file and
   pass it via `--body-file`). Use this exact format:

   ```markdown
   ### Code review

   Found <N> issues:

   1. <brief description of bug>

   <link with full sha and L<start>-L<end>>

   2. ...

   🤖 Generated with [Codex CLI](https://github.com/openai/codex)

   <sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>
   ```

   Or, if no findings survive:

   ```markdown
   ### Code review

   No issues found. Checked for bugs and AGENTS.md compliance.

   🤖 Generated with [Codex CLI](https://github.com/openai/codex)
   ```

### Otherwise (branch range, working tree, or file path)

Return the findings as a JSON array directly in the response:

```json
[
  {
    "file": "path/to/file.ext",
    "line": 123,
    "summary": "one-sentence statement of the bug",
    "failure_scenario": "concrete inputs/state → wrong output/crash"
  }
]
```

Ranked most-severe first. `[]` if nothing survives.

## Notes

- Do NOT run builds, typecheckers, linters, or test suites. These run
  separately in CI and are not in scope.
- Use `gh` for all GitHub interaction (PR fetch, comment post); do not
  web-fetch.
- A missed bug ships. When in doubt between surfacing and dropping, surface.
- Make a todo list at the start to track the four phases.
