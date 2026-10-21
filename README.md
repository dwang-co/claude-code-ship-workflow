# claude-code-ship-workflow

A Claude Code skill (`/ship`) and nine agents that run a software change the way a small engineering team would: scoped, specced, built, reviewed by someone who didn't write it, tested, and handed back as a draft PR.

## Why this exists

When one AI agent plans, writes, and approves its own code, the review is weak. It makes the same assumptions in the review that it made in the build, so it misses the same mistakes.

Engineering teams solve this with roles. The person who specs isn't the person who builds, the reviewer isn't the author, and QA tries to break the change instead of confirming it. This repo gives Claude Code those roles.

`/ship` is the orchestrator. It doesn't write code. It decides which specialists a change needs, gives each one only what it should see, and refuses to move forward on a failed gate.

## How a change moves through it

| Stage | Who | What happens |
|---|---|---|
| 0. Scope | Orchestrator | Turns the request into acceptance criteria: observable statements of done. Too fuzzy to write them? It asks you. |
| 0.5. Spec | `planner` | Reads the codebase and writes an implementation spec, with open questions up top. Skipped for trivial changes. |
| 1. Build | `engineer` | Implements to the criteria, with tests. |
| 2. Review | `code-reviewer` + Codex | Blind review: sees the task, criteria, and diff, never the engineer's reasoning. Runs on a different model, with Codex in parallel when installed. |
| 3. Design | `design-strategist` | Only if UI changed. Checks the design system, brand fit, and the user's path. |
| 4. Narrative | `gtm-strategist` | Only if user-facing. Checks the story, positioning, and whether the target user would understand it. |
| 5. QA | `qa-verifier` | Tries to refute "it works" against each criterion. Every claim needs evidence. |
| 6. Gate | Orchestrator | Typecheck, tests, lint, build. |
| 7. PR | Orchestrator | Opens a draft PR with the criteria, how each was verified, reviewer verdicts, and anything skipped. |

Copy changes add `copywriter` then `editor`. Changes touching AI prompts add `prompt-reviewer`, which runs the real eval harness.

The pipeline scales to the change. A typo doesn't get a five-agent panel. Skipped stages are listed in the PR.

## The rules it follows

- Review runs on a different model from the engineer, and the reviewer never sees the engineer's reasoning.
- Nothing merges without your approval for that specific PR. The pipeline stops at a draft PR, and it never deploys.
- A failed gate gets fixed. A refuted QA verdict is never reworded into a pass.
- Each run saves its artifacts to `.pipeline/<date>-<task>/`, so you can audit it or resume after a crash.
- The strongest model goes to the planner and reviewers, where mistakes are costly and hard to see. Cheaper models handle work that gets checked downstream.

## What's here

```
skills/ship/SKILL.md                  the orchestrator: stages, gates, hard rules
skills/ship/references/escalated-hunt.md   what to do when review keeps finding new problems
agents/                               planner, engineer, code-reviewer, qa-verifier,
                                      design-strategist, gtm-strategist,
                                      copywriter, editor, prompt-reviewer
```

## Install

Copy the skill and agents into your Claude Code config:

```bash
git clone https://github.com/dwang-co/claude-code-ship-workflow.git
cp -r claude-code-ship-workflow/skills/ship ~/.claude/skills/
cp claude-code-ship-workflow/agents/*.md ~/.claude/agents/
```

Then run `/ship` with a task, such as `/ship add rate limiting to the signup endpoint`.

To use one project only, copy into that project's `.claude/` folder instead. A project's own agents take priority.

Some stages lean on tools this repo doesn't include. All are used only when you have them:

- The [Codex CLI](https://github.com/openai/codex) for the second code review.
- The Claude in Chrome extension, which `design-strategist` uses to look at the real UI.
- Your own `/design-solution` skill for new UI direction, and `/scope` to decide whether something should be built at all.
- A `/dual-review` skill (Claude plus Codex, blind to each other) that `/ship` calls on risky specs.

## Make it yours

The agents read your project's `CLAUDE.md`, glossary, and ADRs for its rules, so they check your conventions instead of generic ones. Edit the agent files to match how your team works.
