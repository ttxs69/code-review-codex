# Code Review Plugin (Codex)

Maximum-effort, recall-optimized code review for git diffs and pull requests. Runs 10 independent finder angles in parallel, single-vote verifies each candidate, and closes gaps with a final sweep — capped at 15 ranked findings.

This is a Codex port of [`pi-code-review-max`](https://github.com/ttxs69/pi-code-review-max) (which itself adapts the algorithm from [`code-review-max`](https://github.com/ttxs69/code-review-max)). The pipeline is unchanged; only the packaging, the agent-dispatch idiom (`spawn_agent` vs the Agent tool), the instruction filename (`AGENTS.md` vs `CLAUDE.md`), and the comment-footer text differ. The skill also falls back to PR review when a PR number or URL is passed.

## Overview

The skill runs a four-phase review:

1. **Gather the diff** — `git diff` for working tree / branch ranges, `gh pr diff` for pull requests.
2. **Find candidates** — 10 angles (5 correctness + 3 cleanup + 1 altitude + 1 conventions) run as parallel subagents, each surfacing up to 8 findings.
3. **Verify** — one subagent per remaining candidate returns CONFIRMED / PLAUSIBLE / REFUTED; recall mode keeps anything non-REFUTED.
4. **Sweep** — one fresh subagent looks for defects the first pass missed.

Output is capped at 15 ranked findings. For PR targets, the report is posted as a `gh pr comment`; for branch ranges, working trees, or file paths, findings are returned inline as a JSON array.

## Layout

Codex requires the marketplace manifest at `.agents/plugins/marketplace.json` (not at the repo root). Plugin paths in that manifest resolve relative to the marketplace root — i.e. the repo root — so plugins live at `plugins/<name>/`.

```
code-review-codex/
├── .agents/plugins/marketplace.json            # Codex marketplace manifest (required location)
├── plugins/
│   └── code-review/                            # Matches the name in marketplace.json
│       ├── .codex-plugin/plugin.json           # Codex plugin manifest
│       └── skills/
│           └── code-review/
│               ├── SKILL.md                    # Skill entrypoint (frontmatter + instructions)
│               └── agents/openai.yaml          # UI metadata for chips / lists
├── LICENSE                                     # Apache 2.0 (carried over from the source plugin)
└── README.md
```

## The `/code-review` skill

Performs maximum-effort, recall-optimized code review on a diff (working tree, branch range, pull request, or file path).

**What it does:**

1. Gathers the diff (`gh pr diff` for PRs, `git diff` for branch ranges / working trees, optionally narrowed to a path).
2. Spawns 10 parallel finder subagents — each up to 8 candidates:
   - **Correctness angles**
     - **A**: Line-by-line diff scan (input/state/timing/platform footguns)
     - **B**: Removed-behavior auditor (deleted guards, dropped error paths, narrowed validation)
     - **C**: Cross-file tracer (callers/callees, precondition/return-shape changes)
     - **D**: Language-pitfall specialist (falsy-zero, mutable defaults, nil-map, SQL injection, DST, etc.)
     - **E**: Wrapper/proxy correctness (delegate routing, missing forwards)
   - **Cleanup angles**
     - **Reuse**: existing helper the diff reimplements
     - **Simplification**: redundant state, copy-paste, dead code, deep nesting
     - **Efficiency**: wasted work, sequential when parallel is fine, captured-scope leaks
   - **Altitude angle**: is the fix at the right depth, or a bandaid over shared infra?
   - **Conventions angle**: AGENTS.md compliance (user-level `~/.codex/AGENTS.md`, repo-root, plus ancestor dirs of changed files).
3. Spawns one verifier subagent per deduped candidate — returns CONFIRMED / PLAUSIBLE / REFUTED. Recall mode keeps anything not REFUTED.
4. Spawns one gap-sweep subagent — up to 8 additional defects not already on the list.
5. Caps the final report at 15 findings ranked most-severe first. For PRs, posts the report as a `gh pr comment`. For non-PR targets, returns the findings as a JSON array inline.

**False positives filtered before posting a PR comment** (still surfaced in JSON output):

- Linter / typechecker / compiler issues (CI will catch them)
- Pre-existing issues not introduced by the PR
- Pedantic nitpicks and general quality issues (unless explicitly required by a cited AGENTS.md rule)
- Real issues on lines the PR did not modify (unless the PR re-exposed them via a touched function)

## Installation

### Option A — From this marketplace (recommended)

This repo is itself a Codex marketplace. Add it, then install the plugin — no manual JSON editing required:

```bash
codex plugin marketplace add ttxs69/code-review-codex
codex plugin add code-review@code-review-codex
```

The `marketplace add` step accepts any of:
- GitHub shorthand: `ttxs69/code-review-codex`
- HTTPS URL: `https://github.com/ttxs69/code-review-codex`
- SSH URL: `git@github.com:ttxs69/code-review-codex.git`

Updates: run `codex plugin marketplace upgrade code-review-codex` to refresh the snapshot, then re-run `codex plugin add code-review@code-review-codex` to pick up the new version.

### Option B — Drop into personal skills directory

Copy `plugins/code-review/skills/code-review/` into `~/.codex/skills/code-review/` (i.e. `$CODEX_HOME/skills/code-review/`). The skill is then invocable directly as `/code-review` (no plugin namespace), bypassing the marketplace entirely.

## Invocation

Once installed as a plugin, Codex auto-namespaces plugin skills as `plugin-name:skill-name`, so the slash command is:

```
/code-review:code-review
```

(Use `/code-review` instead if you went with Option B above.)

Codex may also auto-invoke the skill when it judges it relevant, because `disable-model-invocation` is `false`.

## Requirements

- Git repository with GitHub integration
- GitHub CLI (`gh`) installed and authenticated (`gh auth login`)
- `AGENTS.md` files (optional but recommended for guideline checking)

## Differences from the upstream

| Aspect | `code-review-max` (upstream) | This Codex port |
|---|---|---|
| Manifest | n/a (skill-only) | `.codex-plugin/plugin.json` + `.agents/plugins/marketplace.json` |
| Skill unit | `skills/code-review-max/SKILL.md` | `plugins/code-review/skills/code-review/SKILL.md` (renamed to preserve the existing `/code-review` install path) |
| Required frontmatter | `name`, `description` | `name`, `description`, `allowed-tools` |
| Slash command | `/code-review-max` | `/code-review:code-review` (plugin-namespaced) or `/code-review` (personal skills dir) |
| Instruction file | `CLAUDE.md` | `AGENTS.md` (incl. user-level `~/.codex/AGENTS.md`) |
| Agent dispatch idiom | "Use the Agent tool" | "Spawn a subagent (`agent_type: default`, `reasoning_effort: low|medium`)" |
| Comment footer | n/a (returns JSON inline) | When target is a PR: posts via `gh pr comment` with "Generated with Codex CLI" footer; otherwise returns JSON inline |
| Output cap | ≤15 findings | ≤15 findings |

The 10-angle pipeline, 3-state verification, gap sweep, and JSON finding shape are unchanged.

## Configuration

### Raising the output cap

Default is 15 findings. Edit `plugins/code-review/skills/code-review/SKILL.md` and change the `15` in Phase 4 to your preferred cap. Note that surfacing more findings increases the false-positive rate.

### Customizing review focus

Edit `plugins/code-review/skills/code-review/SKILL.md` to add or modify finder angles (e.g. security, accessibility, performance). Each angle should fit the 10-angle structure: independent from the others, single-vote verifiable, up to 8 candidates.

## License

Apache License 2.0 — see [LICENSE](LICENSE). Review algorithm adapted from [`pi-code-review-max`](https://github.com/ttxs69/pi-code-review-max) (Apache-2.0).
