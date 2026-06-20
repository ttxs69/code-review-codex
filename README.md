# Code Review Plugin (Codex)

Automated code review for pull requests using multiple specialized subagents with confidence-based scoring to filter false positives.

This is a port of Anthropic's [`code-review`](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review) Claude Code plugin to the [OpenAI Codex CLI](https://github.com/openai/codex) plugin/skill format. The logic is unchanged; only the packaging, the agent-dispatch idiom, the instruction filename, and the comment-footer text differ (Codex reads `AGENTS.md` rather than `CLAUDE.md`).

## Overview

The plugin automates pull request review by spawning multiple subagents in parallel to independently audit changes from different perspectives. It uses confidence scoring to filter out false positives, ensuring only high-quality, actionable feedback is posted.

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

Performs automated code review on a pull request using multiple specialized subagents.

**What it does:**

1. Checks if review is needed (skips closed, draft, trivial, or already-reviewed PRs)
2. Gathers relevant `AGENTS.md` guideline files from the repository
3. Summarizes the pull request changes
4. Spawns 5 parallel subagents to independently review:
   - **#1**: Audits changes for `AGENTS.md` compliance
   - **#2**: Shallow scan for obvious bugs in the changes themselves
   - **#3**: Reads git blame/history for context-based issues
   - **#4**: Reads previous PRs that touched these files, applies prior comments
   - **#5**: Reads code comments in modified files, checks compliance with them
5. Scores each issue 0-100 for confidence level
6. Filters out issues below the 75 confidence threshold
7. Re-checks PR eligibility before posting
8. Posts a review comment with high-confidence issues only

**Confidence scoring:**

- **0**: Not confident, false positive
- **25**: Somewhat confident, might be real
- **50**: Moderately confident, real but minor
- **75**: Highly confident, real and important
- **100**: Absolutely certain, definitely real

**False positives filtered:**

- Pre-existing issues not introduced in PR
- Code that looks like a bug but isn't
- Pedantic nitpicks
- Issues linters/typecheckers will catch
- General quality issues (unless in `AGENTS.md`)
- Issues with lint-ignore comments

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

## Differences from the Claude Code original

| Aspect | Claude Code original | This Codex port |
|---|---|---|
| Manifest | `.claude-plugin/plugin.json` | `.codex-plugin/plugin.json` (Codex also reads the `.claude-plugin/` path, but we use the native one) |
| Command unit | `commands/code-review.md` | `plugins/code-review/skills/code-review/SKILL.md` |
| Required frontmatter | `description`, `allowed-tools` | `name`, `description`, `allowed-tools` |
| Slash command | `/code-review` | `/code-review:code-review` (plugin-namespaced) or `/code-review` (personal skills dir) |
| Instruction file | `CLAUDE.md` | `AGENTS.md` |
| Agent dispatch idiom | "Use a Haiku / Sonnet agent" | "Spawn a subagent (low / medium reasoning effort)" |
| Comment footer | "Generated with Claude Code" | "Generated with Codex CLI" |

The review pipeline, confidence rubric, false-positive heuristics, and link format are all unchanged.

## Configuration

### Adjusting the confidence threshold

Default is 75. Edit `plugins/code-review/skills/code-review/SKILL.md`:

```markdown
Filter out any issues with a score less than 75.
```

Change `75` to your preferred threshold (0-100).

### Customizing review focus

Edit `plugins/code-review/skills/code-review/SKILL.md` to add or modify subagent tasks (e.g. security, performance, accessibility).

## License

Apache License 2.0 — see [LICENSE](LICENSE). Carried over from the source plugin at `anthropics/claude-plugins-official`.
