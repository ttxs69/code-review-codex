---
name: code-review
description: Code review a pull request
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git log:*), Bash(git show:*), Bash(git blame:*), Bash(find:*), Read
disable-model-invocation: false
---

Provide a code review for the given pull request.

To do this, follow these steps precisely:

1. Eligibility check. Call `spawn_agent` with `agent_type: "explorer"`, `reasoning_effort: "low"`, `fork_context: false`, and a message asking it to run `gh pr view <PR>` and report whether the pull request (a) is closed, (b) is a draft, (c) does not need a code review (eg. because it is an automated pull request, or is very simple and obviously ok), or (d) already has a code review from you from earlier. Then `wait_agent` on its id and `close_agent` it. If not eligible, do not proceed.
2. Gather AGENTS.md. Call `spawn_agent` (`agent_type: "explorer"`, `reasoning_effort: "low"`, `fork_context: false`) with a message asking it to use `find` to return, for the codebase, the file paths (not contents) of the root AGENTS.md file (if one exists), plus any AGENTS.md files in the directories whose files the pull request modified. `wait_agent`, then `close_agent`, and keep the returned path list for later.
3. Summarize the change. Call `spawn_agent` (`agent_type: "explorer"`, `reasoning_effort: "low"`, `fork_context: false`) with a message asking it to view the pull request (PR description, diff, changed file list) via `gh pr view` / `gh pr diff` and return a 3-5 sentence summary of the change. `wait_agent`, then `close_agent`.
4. Then, call `spawn_agent` FIVE TIMES IN A SINGLE RESPONSE (so they run in parallel), each with `agent_type: "default"`, `reasoning_effort: "medium"`, `fork_context: false`, and a focused message for exactly ONE of the subagent tasks below. Then `wait_agent` on all five ids (pass them together so you block until all finish) and `close_agent` each one. The subagents should do the following, then return a list of issues and the reason each issue was flagged (eg. AGENTS.md adherence, bug, historical git context, etc.):
   a. Subagent #1: Audit the changes to make sure they comply with the AGENTS.md. Note that AGENTS.md is guidance for Codex as it writes code, so not all instructions will be applicable during code review.
   b. Subagent #2: Read the file changes in the pull request, then do a shallow scan for obvious bugs. Avoid reading extra context beyond the changes, focusing just on the changes themselves. Focus on large bugs, and avoid small issues and nitpicks. Ignore likely false positives.
   c. Subagent #3: Read the git blame and history of the code modified, to identify any bugs in light of that historical context.
   d. Subagent #4: Read previous pull requests that touched these files, and check for any comments on those pull requests that may also apply to the current pull request.
   e. Subagent #5: Read code comments in the modified files, and make sure the changes in the pull request comply with any guidance in the comments.
5. Confidence scoring. For each issue found in #4, call `spawn_agent` IN PARALLEL with `agent_type: "default"`, `reasoning_effort: "low"`, `fork_context: false`, and a message giving it the PR, the issue description, and the list of AGENTS.md files (from step 2). Ask it to return a score from 0-100 for its confidence that the issue is real vs a false positive. For issues flagged due to AGENTS.md instructions, it must double check that the AGENTS.md actually calls out that issue specifically. Give this rubric to the subagent VERBATIM:
   a. 0: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
   b. 25: Somewhat confident. This might be a real issue, but may also be a false positive. The subagent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant AGENTS.md.
   c. 50: Moderately confident. The subagent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
   d. 75: Highly confident. The subagent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach in the PR is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant AGENTS.md.
   e. 100: Absolutely certain. The subagent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.
   `wait_agent` on all scoring ids together, then `close_agent` each one.
6. Filter out any issues with a score less than 80. If there are no issues that meet this criteria, do not proceed.
7. Re-check eligibility by repeating the check from #1 (spawn_agent + wait_agent + close_agent) to make sure that the pull request is still eligible for code review.
8. Finally, use the gh bash command to comment back on the pull request with the result. When writing your comment, keep in mind to:
   a. Keep your output brief
   b. Avoid emojis
   c. Link and cite relevant code, files, and URLs

Examples of false positives, for steps 4 and 5:

- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (eg. missing or incorrect imports, type errors, broken tests, formatting issues, pedantic style issues like newlines). No need to run these build steps yourself -- it is safe to assume that they will be run separately as part of CI.
- General code quality issues (eg. lack of test coverage, general security issues, poor documentation), unless explicitly required in AGENTS.md
- Issues that are called out in AGENTS.md, but explicitly silenced in the code (eg. due to a lint ignore comment)
- Changes in functionality that are likely intentional or are directly related to the broader change
- Real issues, but on lines that the user did not modify in their pull request

Notes:

- Do not check build signal or attempt to build or typecheck the app. These will run separately, and are not relevant to your code review.
- Use `gh` to interact with Github (eg. to fetch a pull request, or to create inline comments), rather than web fetch
- Make a todo list first
- You must cite and link each bug (eg. if referring to an AGENTS.md, you must link it)
- For your final comment, follow the following format precisely (assuming for this example that you found 3 issues):

---

### Code review

Found 3 issues:

1. <brief description of bug> (AGENTS.md says "<...>")

<link to file and line with full sha1 + line range for context, note that you MUST provide the full sha and not use bash here, eg. https://github.com/owner/repo/blob/1d54823877c4de72b2316a64032a54afc404e619/README.md#L13-L17>

2. <brief description of bug> (some/other/AGENTS.md says "<...>")

<link to file and line with full sha + line range for context>

3. <brief description of bug> (bug due to <file and code snippet>)

<link to file and line with full sha + line range for context>

🤖 Generated with [Codex CLI](https://github.com/openai/codex)

<sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>

---

- Or, if you found no issues:

---

### Code review

No issues found. Checked for bugs and AGENTS.md compliance.

🤖 Generated with [Codex CLI](https://github.com/openai/codex)

---

- When linking to code, follow this format precisely, otherwise the Markdown preview won't render correctly: https://github.com/owner/repo/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
  - Requires full git sha
  - You must provide the full sha. Commands like `https://github.com/owner/repo/blob/$(git rev-parse HEAD)/foo/bar` will not work, since your comment will be directly rendered in Markdown.
  - Repo name must match the repo you're code reviewing
  - # sign after the file name
  - Line range format is L[start]-L[end]
  - Provide at least 1 line of context before and after, centered on the line you are commenting about (eg. if you are commenting about lines 5-6, you should link to `L4-7`)
