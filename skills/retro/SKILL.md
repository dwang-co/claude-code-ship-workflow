---
name: retro
description: Run a retrospective on a /ship session, PR review feedback, or a stretch of agent-assisted work, and turn the lessons into proposed improvements to the agent definitions (`.claude/agents/` or `~/.claude/agents/`), skills (`.claude/skills/` or `~/.claude/skills/`), and CLAUDE.md. Proposals only - every change is a diff presented for the user's approval, one at a time, and this skill applies the edit itself only after that approval; nothing is ever applied without it. Includes a mandatory consolidation pass so the context stays lean. Use when the user says "retro", "what did we learn", "improve the agents/workflow", or after a PR lands with substantial human corrections.
---

# /retro: the learning loop

An agent system gets smarter by compounding better context, because the model itself doesn't change. That only works if the context stays good. A retro that only adds rules ends up with a bloated, contradictory instruction set that makes every later session worse. Your output is proposals, the user decides what stays, and pruning is as much your job as capturing.

Keep one canonical copy of this skill (global, or in one project), not several. Copies drift, and a drifted copy quietly applies stale rules. A project's own local facts belong in that project's `CLAUDE.md`, referencing this skill by name.

## Step 1: Gather evidence

Review whichever of these exist for the period under retro:

- **Run folders** (`.pipeline/<date>-<slug>/`, written by `/ship`): the primary durable record. Read each run's spec, handoffs, verdicts, and QA evidence. Compare what the planner specced against what shipped (deviations signal spec quality), what the reviewer caught against what the engineer handed off (foreseeable-defect signal), and whether specced runs looped less than unspecced ones (the data that decides whether an expensive model can be downgraded).
- The session itself, if still available: where did an agent loop back? What did the code reviewer or QA verifier catch that the engineer should have foreseen? What did a stage miss that a later stage (or the user) caught?
- PR review feedback: what did the user correct at review? Human corrections are the highest-signal input here. Each one is a defect the whole agent pipeline missed.
- Rework and friction: stages that ran but added nothing, questions the orchestrator had to ask that a better prompt would have pre-answered, wrong assumptions traced to a missing or misleading instruction.
- If the session chained a scoping step (such as `/scope`) directly into `/ship`: check whether its output needed clarification or rework once `/ship`'s intake picked it up, such as ambiguity it should have resolved, missing context that had to be re-derived, or a shape that didn't map onto acceptance criteria. If that is a recurring gap rather than a one-off, propose a fix to the scoping skill's output format and/or `/ship`'s intake, and say so explicitly if the fix touches a global skill, since it then affects every project.

A lesson must trace to a *specific observed failure or friction*, not a hypothetical. "This might help someday" is how bloat starts.

**Counterpart check, before anything else.** Any skill or agent this retro's evidence touches may have another maintained copy: project-local vs. global, a worktree's own checkout, or a published copy. Diff every copy that should exist, now, unconditionally, not only when this pass happens to produce a local edit. A sync rule that only fires at the end of a retro cannot catch drift introduced by a session that never touched the other location. Report the diff, not just that you looked: line count alone doesn't prove staleness (legitimate genericization isn't drift), so compare the `## Provenance` sections specifically. An entry present in one copy and absent from the other is real, unreconciled drift, and reconciling it is part of *this* pass. A worktree's checkout can also silently lag its source branch, so name that as a possible staleness source when running in one.

If evidence for the period is thin or absent (a torn-down worktree whose `.pipeline/` folder wasn't kept, no PR feedback to review), say so explicitly and scope the retro to what's available. A stated "insufficient evidence this round" is a valid, expected output. Don't silently proceed on weak evidence, and don't silently produce a thin or empty retro without naming the gap.

## Step 2: Draft proposals

Before drafting a proposal against a file, read its `## Provenance` section for prior rejections of the same or a similar rule (Step 5). Don't re-propose one unless new evidence has emerged since the rejection.

**Also check the lesson against the file's existing live content, not just its Provenance history.** Does an existing rule in the section this touches already cover it and just need tightening? Does the new rule make an existing one redundant, partially redundant, or contradictory? Do this at the moment each proposal is drafted, not as a separate later pass; a consolidation check deferred to "later" is the one that reliably never happens. Land on one of the following, each with a cited audit trail (name the specific sections actually read, since a bare "checked, clean" is unverifiable):

- **Pure addition**: state which sections were checked and why nothing there covers this.
- **Replaces existing rule X**: the diff deletes or rewrites the old text in place rather than stacking beside it; name X.
- **Tightens existing rule X**: merge into one rule rather than two related-but-separate ones; name X.

This determination travels into Step 4 as part of the proposal.

**Sibling check.** Before finalizing which files a proposal touches, ask whether the rule has siblings: other files implementing the same methodology for a different target. If the same rule is stated in several files, a fix applied to only one leaves the others contradicting it. Fast test: *could you delete this file's own name or subject from the lesson's wording and have it still make sense?* If yes, it's a candidate to generalize, not a file-local fix. When the test is positive, grep the adjacent rules and skills directories (user-level and project-level) for the same rule text, and extend the diff to every sibling it genuinely applies to. State the verdict explicitly, even when negative: "checked siblings, instance-specific, no propagation" is a valid, recorded outcome.

For each lesson, draft a concrete diff to the file that owns the behavior: an agent definition, a skill, or CLAUDE.md. Prefer editing an existing rule over adding a new one. Write rules as *why + what*, not bare imperatives, because agents follow rules better when the reason travels with them.

Guard against overfitting: one incident is an anecdote, not a rule. If the lesson generalizes ("QA needs acceptance criteria before it can verify anything"), propose it. If it's specific to one odd task, propose nothing, or at most a note in the PR. A single incident may still be proposed as a **flagged experiment** (Step 4) if the underlying failure class plausibly recurs elsewhere; propose nothing only when the incident is genuinely idiosyncratic.

Proposals aren't limited to edits. When the evidence shows a *recurring workflow with no owner* (the same multi-step sequence reinvented across sessions, or a capability gap no existing agent covers), propose a **new skill** (`.claude/skills/<name>/SKILL.md`: precise triggering description, lean body, the why explained) or a **new agent definition**. New capabilities go through the same one-at-a-time approval gate as edits, and the bar is higher: a new skill is a permanent context cost on every future session, so it must be justified by observed repetition, not anticipated usefulness.

## Step 3: The consolidation pass (mandatory)

Every retro also audits the existing context for bloat, scoped to the files this retro's evidence actually touched or referenced, not a sweep of the whole corpus, which one retro's evidence can't support. Every token in these files is spent on every future session, so each one must make the system better, not worse. Look for:

- **Stale rules**: instructions about code, decisions, or phases that no longer exist.
- **Redundancy**: the same rule stated in two files, or a rule that merely restates what CLAUDE.md already says. One owner per rule; others reference it.
- **Disclosure level**: keep behavior required on every invocation in the skill body; move long provenance narratives, worked examples, and scenario-specific procedures into a clearly named `references/<topic>.md` file that the skill says when to read. Moving text out of sight with no clear retrieval trigger is not consolidation, it's hiding.
- **Dead weight**: rules added by past retros with no evidence they ever changed an outcome. "Earned its keep" is checkable: a Provenance entry (Step 5) can point to a specific later session where the rule visibly fired, by catching something, changing a decision, or preventing a repeat. If no such session is on record, say so ("no evidence either way this pass") rather than treating silence as proof of uselessness or value. Propose deletion only once that absence has persisted across multiple retros with real opportunity for the rule to fire.
- **Contradictions**: rules that pull in different directions. Surface the conflict for the user to resolve rather than letting agents silently pick one.

**Mechanical growth check, on every file this pass touches.** "Propose deletions and merges with the same seriousness as additions" is a mandatory instruction with no forcing function, and instructions like that don't reliably get followed: a file can pass through many retros, with the consolidation step nominally running each time, and still grow every time. So check the file's current **instruction-line count** (everything above its `## Provenance` section) against the count **at the last commit where content was actually deleted, merged, or relocated**. Count instruction lines rather than raw lines, so the mandatory Provenance bookkeeping isn't charged as bloat and relocating narrative out of the always-loaded body gets credit. An edit that only adds does not reset the baseline, or an unrelated one-line touch could defeat the check. Find the baseline via `git log` / `git show <prior-commit>:<path>`, or use the file's Provenance dates as a rough proxy if history is unclear. Report the delta.

This check catches unbounded growth only. It does **not** catch semantic redundancy at flat line count (a rewrite that swaps one bloated section for an equally bloated one). Treat it as one input to the audit above, not a substitute.

If growth since that baseline exceeds roughly 30% or 100 lines, whichever is smaller (a provisional threshold, to be loosened if it fires on additions later judged necessary and tightened if real bloat slips under it), **consolidate in the current pass, before accepting any new additions.** One carve-out: a genuinely high-severity corrective rule (a real, currently live risk) may still go in immediately, but the baseline stays where it was.

Propose deletions, merges, and relocations with the same seriousness as additions. Relocation is usually the right move for narrative (case history, prior-incident detail, "why we decided this") once a file's Provenance section passes roughly 20% of it. Narrative is overview-shaped and doesn't need to load every session, even though it must stay findable. A retro that proposes only additions should be treated as incomplete: say explicitly what you looked at and found clean. If the evidence is too thin to support a real consolidation pass, say that instead of skipping the step silently.

## Step 4: Present for approval

Present each proposal **one at a time** via AskUserQuestion. The user skims, so lead with exactly four lines, each one or two plain sentences, no jargon, using the real incident as the example:

- **What went wrong:** the observed failure or consolidation finding (Step 3).
- **Recommended fix:** what changes, in plain words.
- **Confidence it works:** High / Medium / Low plus a one-line reason. Rank proposals by this, separating a "confident core" (mechanical, bright-line, likely to be followed) from "flagged experiments" (cheap, plausible, unproven).
- **Impact:** what gets better, what it costs (extra context, risk), and what breaks if it's wrong.

Below a divider labeled "reference only," put the diff, the Step 2 determination (pure addition / replaces / tightens), and the Step 2 sibling-check verdict. Keep the end-of-retro summary to the same four headings. The user approves, edits, or rejects. **Never apply a change that wasn't approved**: an unsupervised self-editing loop is exactly the failure mode this design rejects. Rejected proposals are recorded (Step 5) so future retros don't re-propose them.

## Step 5: Provenance

Each touched file keeps a short `## Provenance` footer: one line per retro-originated rule, with the date added and a one-phrase trigger. On later retros, update the line if the rule earned its keep (per Step 3's evidence bar) or propose deletion if it never did. This tally is the data behind consolidation and promotion decisions. When Step 2's sibling check found or ruled out siblings, say so in the line too (a short clause: "also applied to X, Y" or "checked siblings, none applied"), which makes a *skipped* sibling check visible on a later audit.

**Flagged experiments get an actual checkpoint, not just a note.** A blanket "track whether every rule fired" field tends to produce no signal: every entry sits at a permanent "active" with nothing prompting a revisit. Scope it to the smaller set that needs it. When a proposal is approved as a flagged experiment, its Provenance entry also states what would confirm or kill it and a review point, normally "the next retro that touches this file, after N genuinely eligible sessions," not a calendar date. Then give it a real trigger: **Step 3's consolidation pass must walk every flagged-experiment entry in a file it touches** and record one of: cite a session where it fired and promote it off experiment status, extend it once for too few eligible opportunities, or propose removal. A flagged experiment with no stated review point is itself a Step 3 finding.

Rejected proposals are logged too, but logging a rejection is not a silent exception to "nothing is applied without approval." Either (a) fold the rejection-log line into the same batch of diffs presented in Step 4, so the user approves it with everything else, or (b) when the proposal concerned a brand-new file with no `## Provenance` footer to write into, record the rejection in the retro's own run record instead of touching a file the user never approved touching.

## Step 6: Promotion path (project to global)

Two speeds, depending on whether the file already has a global counterpart:

- **A change to a file that already has a global counterpart** (any skill or agent living in both a project's `.claude/` and `~/.claude/`), **and the change is not specific to this project's product or domain** (a process fix, a new decorrelation rule, a corrected tool reference, a sharpened trigger condition): propose the matching edit to the global copy in the *same* retro pass the local edit is approved. A global copy that only gets updated "once it's proven out" drifts stale the moment it stops being actively maintained.
- **A genuinely new rule, skill, or agent with no existing global counterpart**: hold to a higher bar. Let it prove out across multiple sessions before proposing promotion to `~/.claude`. A single-incident lesson promoted too early pollutes every future project's context with something that hasn't earned cross-project trust.

Either way, note in the project-local copy that a rule was promoted or synced, so the two don't drift silently, and flag in the proposal whether the global edit is a *sync* (mirroring an already-approved local change, lower risk) or a *promotion* (a local rule graduating to global for the first time, higher bar).

Before ending the retro, re-run Step 1's counterpart check on every file this pass touched, not just the ones with an approved local edit. That catches drift the pass itself might have just introduced, or drift that was present going in but easy to miss once attention moved to drafting proposals.
