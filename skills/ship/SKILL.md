---
name: ship
description: Orchestrate an agent-managed development task end to end — scope it into acceptance criteria, spec non-trivial work with the planner agent (with a dual-model adversarial review when the spec carries real judgment), delegate implementation to the engineer agent, run adversarial code review on a different model (plus Codex, when available), pull in design-strategist and gtm-strategist when the change is user-facing, gate through qa-verifier and the project's Definition of Done checks, and open a draft PR with verification evidence. Use whenever the user hands off a feature, bug fix, or change to build and ship — "build X", "implement Y", "fix Z", "ship this" — or invokes /ship explicitly. Not for pure questions, analysis, or planning-only requests. For whether something should be built at all, see /scope first.
---

# /ship — agent-managed delivery protocol

You are the orchestrator. You coordinate, gate, and report — the specialist agents (in this project's `.claude/agents/` if it defines its own, otherwise `~/.claude/agents/`) do the work. The protocol exists to produce *verified* work with decorrelated review, at a cost proportional to the change. Follow the stages in order; skip only where the stage-selection rules below say a stage doesn't apply, and disclose every skip in the final report.

Before starting, read the project's CLAUDE.md and any glossary/ADR/invariant conventions it defines — every agent prompt you write should carry this project's own invariants, not generic assumptions or another project's rules.

## Run folder — durable stage artifacts

Every run gets a folder: `.pipeline/<YYYY-MM-DD>-<task-slug>/` (add `.pipeline/` to the project's `.gitignore` on first use — it's a local paper trail, not PR content). As each stage completes, save its output there before moving on: `spec.md` (planner), `dual-model-review-round1.md`/`round2.md`/etc. (each Stage 0.5 adversarial spec-review pass, when it runs), `acs.md`, `design-spec.md` (`/design-solution`, when invoked), `copy-draft.md` (`copywriter`) and `copy-final.md` (`editor`, when either run), `engineer-handoff.md`, `review-verdict.md`, `design-review.md`, `gtm-review.md`, `qa-evidence-round1.md`/`round2.md`/etc. (each `qa-verifier` pass, when it runs —
round-numbered so a REFUTED round isn't silently overwritten by a later VERIFIED one; see Stage 5).
Conversation context dies with the session; these files are what makes a run auditable, resumable after a crash, and — critically — what `/retro` reads as evidence. A stage that ran but left no artifact is invisible to the learning loop.

**Treat this as a blocking checkpoint, not a reminder to keep in mind.** Before making the next `Agent`/`SendMessage` call anywhere in the pipeline, confirm the round that just completed already has its output captured in the run folder — write it first if not. This explicitly includes iterative rounds coordinated by resuming an agent via `SendMessage` rather than a fresh `Agent` dispatch (e.g. a Stage 0.5 spec-revision round, or a Stage 2 fix-and-re-review loop) — a resumed agent's conversation thread is exactly as invisible to `/retro` and to a fresh `qa-verifier` dispatch as a completed one whose summary never left the transcript. A passive version of this reminder (below, in Provenance) failed to stick across two full runs the very next day — restating it more emphatically didn't fix it; treating "write the artifact" as a precondition for the next action does.

**This checkpoint's own trigger doesn't fire between every stage — Stage 6 (DoD gate) and Stage 7 (git/PR operations) are Bash calls, not `Agent`/`SendMessage` calls**, so the generic reminder above has nothing to hang on between QA finishing and the PR body needing `qa-evidence-round1.md`. Each stage that produces an artifact writes it the moment that stage's own work completes — don't defer any artifact write to "whenever the next dispatch happens to be." Before Stage 7 opens the PR, mechanically verify every artifact implied by Stage 0's stage-selection actually exists and is non-empty (including at least one `qa-evidence-round*.md`) — check against what Stage 0 actually selected, not one unconditional list, since a correctly-skipped stage has no artifact and shouldn't fail this check.

## Stage 0 — Scope and acceptance criteria

Before any delegation:

1. Check the task's language against the project's own glossary doc, if one exists; challenge vague or conflicting terms before proceeding. If scope is genuinely ambiguous, ask the user — one good question now beats a wrong build.
2. Write **acceptance criteria**: concrete, observable statements of done ("an unauthenticated request to X returns 401"). These are the contract for the whole pipeline — the engineer builds to them and the qa-verifier refutes against them. A task too fuzzy to produce ACs is not ready to ship; go back to the user. **Write them to `acs.md` in the run folder now, as this step completes** — every downstream stage that needs the ACs reads or gets injected this file directly; nothing downstream re-derives or manually restates the list from memory, which is what caused a real false out-of-scope finding once (see Stage 2 below). **If a later fix round (Stage 2) intentionally changes behavior an existing AC explicitly locks down** — as a disclosed, reviewed tradeoff, not a regression — **update `acs.md` in that same fix round to state the exception.** Don't leave the AC's original wording to fail QA and get patched reactively afterward. **When an AC's pass/fail depends on live AI/model output rather than a pure code guarantee, write the deterministic guarantee and the AI-dependent success rate as two separate claims, not one blended absolute statement.** "The join always happens" is checkable for a code-level normalization step; it is not a fair claim about a model call, however explicit the prompt instruction is. State what the code guarantees unconditionally (e.g. "no embedded newline or cross-script space ever reaches the export"), and separately state the model's observed reliability on the actual behavior it controls (e.g. "the model joins successfully in N/M live-sampled runs on case X") — verified via live samples, not asserted.
3. List **OPEN QUESTIONS**: every ambiguity or judgment call the task leaves unresolved, surfaced explicitly — never silently guessed. Resolve them with the user (or state the default you're taking and why) before delegation.
4. Decide which stages apply (see stage selection below) and say so up front.
5. If continuing work on an existing branch rather than starting fresh, check how far it's diverged from `main` (`git log main..HEAD --oneline | wc -l`, and whether files this task touches have changed independently on `main` since the branch point). A long-lived branch can be superseded by unrelated work that already landed — running the full review pipeline against it wastes every downstream stage on work that needs reconciliation, not review. Surface a real divergence to the user before delegating, the same way Stage 5's scope-creep check does mid-run. **Always `git fetch origin main` first and diff/compare against `origin/main`, never the local cached `main` ref, before drawing any divergence or contamination conclusion.** A worktree's local `main` ref can be stale relative to the remote with no warning — diffing against it can make already-merged, unrelated work look like it's part of this branch.
6. Check whether another session might be concurrently active in this exact working directory (other `claude`/`codex` processes touching this repo, or just ask) before deciding to skip worktree isolation. A shared, un-isolated checkout means another session's branch switches, uncommitted edits, or background reviews can silently mix into this run — a branch getting swapped out from under a task, or a code-review tool picking up someone else's uncommitted file, are both real, observed failure modes, not hypothetical. When in doubt, isolate.
7. **Intake: record whether this task's input is a raw ask or a `/scope` goal prompt.** If it's a
   `/scope` goal prompt: link or paste the actual verdict (BUILD / BUILD SMALLER, and any stated
   cut-line) into `acs.md`'s context so it's traceable, not just pasted prose with no record of
   where it came from. Treat a BUILD SMALLER cut-line as a hard constraint on this task's scope
   unless something has genuinely changed since `/scope` ran. **Collision check, regardless of
   whether `/scope` already ran**: does this task conflict with, duplicate, or partially reopen an
   existing decision (an ADR, a roadmap/backlog entry, prior committed direction — whatever this
   project uses for this)? Surface a real collision to the user before delegating rather than
   letting planner discover it mid-spec. **When the task plans to change a shared type or field,
   grep the WHOLE repo for its usages before scoping — not just the directory the feature
   conceptually belongs to.** Most `/ship` invocations skip `/scope` by
   design — trivial
   fixes, or work already scoped with a clear decision on record — so this can't live only inside
   `/scope`'s own collision check; Stage 0 has to own a baseline version of it for every task,
   scoped or not. **If the task did come from a `/scope` goal prompt, don't treat its collision
   check as settled just because it already ran** — see Stage 0.5 below for why. **If this
   collision check surfaces genuine ambiguity that isn't cleanly resolved, that itself qualifies as
   a trigger for Stage 0.5's dual-review pass** (see Stage 0.5's trigger list) — rather than adding
   a separate Stage 0 review gate, since Stage 0 produces a problem statement and ACs, not a
   solution, and there's nothing yet for a solution-level adversarial pass to review.

   **Automatic premise screen for unscoped, non-trivial, undecided tasks.** If this task is
   non-trivial, did not come from `/scope`, and isn't already backed by a recorded decision, run a
   single-pass check automatically — no approval needed to run it. Check the project's roadmap/
   backlog, existing ADRs, its glossary doc, and memory for prior evidence this problem is real,
   then apply `/scope`'s own pressure-test lens (outcome vs. output, the four risks, riskiest
   assumption, painkiller vs. vitamin) to the task as stated. Every verdict must cite the specific
   source checked and what was or wasn't found there — never a bare label. Land on exactly one of:
   evidence found — looks validated; evidence found — looks like an unvalidated/output-driven
   pattern; or no evidence either way, stating explicitly whether that absence is expected
   (internal/technical work) or itself the concern (an unvalidated user-facing bet). If the result
   is "unvalidated pattern," confidence is low, or evidence conflicts, escalate to Stage 0.5's
   dual-review trigger (see the added condition there) rather than resolving it single-handed. Log
   the trigger decision itself every run — "ran, because X" or "skipped, because Y" — the same
   disclosure standard this skill already applies to skipped stages. A genuine "unvalidated
   pattern" finding gets a durable roadmap/backlog entry, not just a mention in the run report, and
   is stated plainly in Stage 0's output — never as a blocking question, just as a fact the user can
   act on or wave off. This check is additive to any other process gates the user's own standing
   instructions require (a kickoff interview, an architecture review, pre-code approval) — it
   doesn't replace them. This is a flagged experiment, not incident-evidenced yet; revisit once it's
   had a chance to run.
8. **If the project has an architecture-review gate** (an `/arch-review`-style skill, or an equivalent standing rule requiring review before non-trivial changes) **confirm it ran for this task before Stage 1.** This is a prose confirmation question, not a file check, if that gate's own output isn't persisted anywhere to check against — ask directly and record the answer rather than searching for an artifact that may not exist. If the task touches a category that gate treats as protected/high-risk, confirm that was explicitly acknowledged, not silently assumed. Apply this to every non-trivial task the gate's own rules say it covers, not just ones that happen to touch an obviously protected file.

## Stage 0.5 — Implementation spec (planner) — *if non-trivial*

For any task that is more than a copy tweak, one-line fix, or trivially-scoped change: delegate to the `planner` agent with the task, the ACs, and the run-folder path for `spec.md`. It reads the codebase and produces a concrete implementation spec — files and paths, signatures, edge cases, patterns to copy — with OPEN QUESTIONS at the top.

**If this task's input is a `/scope` goal prompt, planner re-verifies its premises — it does not inherit them.** `/scope` runs as a single inline pass in the orchestrator's own context, with no independent review by default (its own Verdict step only triggers `/dual-review` for a narrow, conditional case — see `/scope`'s own instructions). Planner's deep codebase read is the first point those claims meet fresh evidence: before writing the implementation spec, confirm `/scope`'s collision check still holds (nothing relevant has changed since it ran) and its stated riskiest assumption still looks right. Note any drift explicitly in the spec's own OPEN QUESTIONS rather than silently building on a possibly-stale premise. This applies even when `/scope` ran only minutes before Stage 0.5 starts — elapsed conversation turns and time are enough for codebase state to move, the same reason this project already re-checks git-state drift at multiple points (Stage 0 item 5 above, Stage 2's pre-review check, Stage 7's pre-push reconcile) rather than trusting one upstream look.

**If this run is isolated in a worktree, give every dispatched subagent the FULL ABSOLUTE path to the worktree for any run-folder file it reads or writes — never a bare relative path like `.pipeline/<slug>/spec.md`.** A subagent's tool calls do not inherit the orchestrator's worktree-modified working directory; telling it "you're in a worktree" in prose is not enough. Confirmed directly: a planner dispatch given only a relative run-folder path wrote `spec.md` to the main checkout instead of the isolated worktree, invisible until the orchestrator searched the whole repo for it — and a later engineer dispatch on the same run independently developed a habit of copying every handoff to both locations each round, which is the symptom of this exact gap never having been fixed at the root. After any subagent dispatch that was told to write a run-folder artifact, verify it landed at the expected absolute path rather than assuming it did.

**Worth considering, not a required gate: if the spec's core deliverable is a new, unproven AI/prompt-based classifier or checker, and the project already has real labeled examples lying around (documented findings, eval fixtures with known-good/known-bad cases), a cheap standalone probe — run the drafted prompt directly against that existing ground truth, no pipeline integration needed — can validate or kill the core hypothesis before committing to the full build.** One instance of this caught a real precision problem for a fraction of the cost of building the full pipeline integration first and finding out via live eval afterward. Evidenced once so far — treat as a judgment call to reach for when it obviously fits, not a mandatory step for every spec.

If the task is new UI-facing surface (not just a fix to something existing) and needs a real design direction, not just an implementation of one already given: invoke the `/design-solution` skill (if the project has one) alongside planner, feeding it the task and any reference material. If that surface also needs new or substantial user-facing copy (new sections, new states, onboarding, marketing) and a `copywriter` agent is defined: delegate to it with `/design-solution`'s output (or the planner spec, if `/design-solution` didn't run) so copy is written against the actual structure it fills rather than freehanded later by whoever implements it.

**If an `editor` agent is defined, copywriter's draft always goes to it before it counts as done** — the same decorrelation principle as Stage 2, applied to prose instead of code: hand `editor` the draft, the spec it was written against, and nothing else copywriter said about its own choices, so editor forms an independent view. `editor` returns an edited, improved version plus a verdict (APPROVED or NEEDS REWRITE). NEEDS REWRITE loops back to `copywriter` with the named structural gap — don't send prose back and forth polishing wording when the actual problem is the approach. `editor`'s version, not copywriter's draft, is what goes forward.

**If the project has a mechanical AI-writing-tells pass (a `/humanizer`-style skill), run it on the approved copy before it goes to `engineer`.** `editor` judges voice, clarity, and structural correctness; a narrower mechanical pass (em dash overuse, promotional language, rule of three, and similar tells) isn't `editor`'s job to catch by itself. This kind of check is typically a skill, not an agent — subagents can't invoke skills, so run it yourself, in the orchestrator's own context, directly on `editor`'s approved text. Any rewrite it makes is mechanical cleanup, not a new round of judgment calls — it doesn't loop back through `editor` unless it changes meaning, which it shouldn't.

Save every artifact that ran (`design-spec.md`, `copy-draft.md`, `copy-final.md`, `copy-humanized.md`) to the run folder alongside `spec.md`.

**Dual-model adversarial spec review — proportional to the spec's actual risk, not every spec.** Run this whenever Stage 0.5's spec meets at least one of:
- It's a deep root-cause investigation (diagnosing an instability or a recurring defect, not implementing a feature) that will spend real API budget on a data capture.
- Planner's own OPEN QUESTIONS name a genuine judgment call — more than one viable technical approach, an ambiguous root cause, a real design tradeoff — not just a clarifying detail a quick answer resolves.
- The spec's conclusions could gate a change to the project's highest-blast-radius surface — whatever files or logic a pre-launch or production gate treats as load-bearing (an AI-prompt file backed by a live eval harness, an auth/payments path, a schema migration — whatever this project itself flags as such).
- Stage 0's collision check (item 7) surfaced genuine ambiguity that wasn't cleanly resolved.
- The spec touches a bright-line risk category regardless of file path: an irreversible schema or
  data migration, an authorization or tenant-isolation design, a production runbook a human will
  execute against live systems, an expensive external integration, or an AC that depends on a
  disputed measurement or proxy.
- Stage 0's automatic premise screen (item 7) landed on "unvalidated pattern," returned low
  confidence, or found conflicting evidence.

Skip it for a spec that names one clear approach with no open questions and no high-blast-radius surface — present that straight to the user as today. The point of this stage is cost proportional to risk, not a second gate on every spec regardless of stakes; a spec with a genuinely unambiguous approach doesn't need two extra model calls to confirm it.

When triggered, before presenting the spec to the user, invoke the `/dual-review` skill on it (Claude + Codex, parallel, blind, synthesized) rather than hand-rolling a bespoke Codex+Opus dispatch here — `/dual-review` is the reusable mechanism for this now (see its own skill file for the brief-writing/dispatch/synthesize mechanics); this paragraph used to specify the dispatch directly, before `/dual-review` existed as a proper skill. Each model type has different blind spots; single-vendor convergence is not real convergence for this shape of task — a spec that reads as fully resolved after two same-vendor rounds can still be hiding a defect a different model family would have caught immediately (see this file's Provenance for the case that established this).

Iterate fixes with whichever model found the issue. **Convergence requires both model families to review the same post-fix revision and each return no new substantive defect** — not "a fresh round from either model comes back clean," which only proves one model is satisfied, not that both actually looked at what changed. A model that already approved an earlier revision hasn't necessarily re-examined what changed since; treating its silence as re-confirmation is how a real defect slips through unreviewed by anyone on the round that actually matters. Do this proactively, before the user has to ask for the second model type.

**This is artifact re-verification, not cross-model debate — the two are different, and only the
first is safe to iterate freely.** If the two reviewers instead reach a genuine, material
disagreement about the *approach itself* (not a defect in a revision), don't have them read and
respond to each other's stated position — that trades away the independence the whole mechanism
depends on. Follow `/dual-review`'s own disagreement-resolution protocol (its Step 5) rather than
re-deriving one here. A genuine, evidence-checked disagreement that survives it is a valid terminal
outcome to hand to the user — not a failure state, and not something to force into false consensus.

**If a spec's dual-review rounds keep surfacing *new* substantive problems round after round** — the
same serial pattern that triggers Stage 2's escalated hunt — stop iterating locally and take the
problem back to the user or back to Stage 0's framing, rather than spending more rounds on a spec that may
be resolving the wrong problem. **Write the stop condition into that round's synthesis file BEFORE
dispatching the next round** (e.g. "if round N+1 finds a new *silent* miss, stop and take it to the user —
no round N+2 patch"), so the decision isn't relitigated once a result is in hand. Classify each remaining finding as *loud* (fails visibly — a wrong name, a false alarm) or
*silent* (a real object could go unchecked) when the design is fail-closed — only silent findings
justify another round.

**Present the spec (and design-spec/copy-final, when they exist) to the user for approval before Stage 1.** This is the user's cheapest checkpoint: a plain-language statement of what's about to be built, gated before any code exists. Surface the planner's open questions as questions, not footnotes — and explicitly attribute each one to its source ("planner flagged this as ambiguous while writing the spec" vs. "this is the orchestrator's own judgment call") rather than presenting a bare question.

Two rules keep the spec from becoming an echo chamber:
- **The code-reviewer never sees the spec.** It reviews from task + ACs + diff only and re-derives its own view — a reviewer verifying "does the diff match the spec" would inherit the spec's errors instead of catching them. QA likewise verifies the ACs (outcomes), not spec conformance (means).
- **Deviations are disclosed, not hidden.** If the engineer departs from the spec mid-build (the specced approach doesn't fit, an API doesn't exist), the handoff and PR body say what changed and why. A spec the code quietly contradicts is worse than no spec.

## Stage 1 — Implement (engineer)

Delegate to the `engineer` agent with: the task, the ACs, the spec (when Stage 0.5 ran), and relevant context (files, ADRs, glossary terms). Independent slices can go to parallel engineer agents; keep each slice's ACs separate.

**For any dispatch expected to run long (extensive live verification, many tool calls), instruct the agent to write its handoff/progress incrementally to its run-folder artifact as it goes, not only at the very end.** A long dispatch can be interrupted by something with nothing to do with its own correctness — the local machine sleeping, a stream stall — and when that happens, an agent whose transcript can't be recovered has to be redispatched from zero, losing everything already done. Don't wait to add this ad hoc after the first loss.

## Stage 2 — Adversarial code review (code-reviewer + Codex, in parallel)

**Before generating the diff for review, re-check whether `main` has moved since the branch's base commit** — this matters most after any Stage 1 dispatch that ran long. A long-running engineer dispatch is exactly the window where unrelated work can land on `main` (yours or someone else's), and a branch/worktree created before that work landed will show it as phantom "removed" content once diffed against the now-advanced `main` — wasting a review pass on confusion rather than the actual change, and risking a reviewer misreading a phantom hunk as a real regression. **Run `git fetch origin main` before this check, not just `git log`/`git diff` against the local ref** — the local ref doesn't update itself and a stale one produces exactly the same phantom-diff confusion this check exists to prevent (see Stage 0 item 5's identical note). If `main` has advanced: rebase onto it, confirm it applies cleanly, re-run the test suite, then generate the diff. This is a second checkpoint, not a duplicate of Stage 0's divergence check — Stage 0 catches a branch that was already old when the run started; this one catches drift that happens *during* the run.

Every implementation gets reviewed by the `code-reviewer` agent, and the review must be **blind**: hand the reviewer only the task, the acceptance criteria, and the diff — never the engineer's handoff, rationale, or conversation, and never the Stage 0.5 spec. Sub-agents start with clean context, so this isolation holds as long as you don't paste the engineer's reasoning into the prompt. An anchored reviewer inherits the engineer's framing and rubber-stamps it; an unanchored one has to form its own model of what the code should do, which is where disagreement — the value of this stage — comes from.

**Give the blind reviewer `acs.md`'s actual contents — the absolute path to it, or its verbatim text injected into the dispatch — never a manually re-typed summary.** A shortened restatement that omits legitimately-approved scope produces a false "out of scope" finding the reviewer has no way to know is wrong, since it can't see the actual approved spec. This is exactly the mistake a hand-restated summary caused before Stage 0 required writing `acs.md` to disk as its own step — don't recreate the bug by typing the list again here instead of pointing at the file.

**The reviewer must also run on a different model than the engineer** — decorrelated blind spots. `code-reviewer`'s agent definition typically pins a top-tier model; `engineer` often has no override and inherits the session model. When the session is already on that top tier, both land on the same model even though the pin looks like it guarantees otherwise. Check the session's actual model before every code-reviewer call, not from memory, and **record what you found and what you did about it**:
- Record the engineer's effective model and the reviewer's requested model (an explicit `model` override on the Agent-tool dispatch takes precedence over the agent definition's own frontmatter pin).
- If they match, dispatch the reviewer with an explicit override to a genuinely different available tier.
- If no different tier is meaningfully available, don't claim decorrelation you don't have — note in `review-verdict.md` that same-model Claude-side decorrelation was unavailable this round, and rely on the parallel Codex review as the real cross-check.
- Only Claude models can power agents in this harness regardless of tier chosen, which is a narrower kind of decorrelation than it looks like — same-vendor models share training lineage, RLHF philosophy, and the categories of thing their training makes salient, regardless of size/tier. This is exactly why the parallel Codex review below is load-bearing, not optional redundancy.

**If Codex CLI is available, run `codex review` in parallel with `code-reviewer`, starting from the first round, not after Claude converges.** Sequencing it last (run Claude to convergence, then Codex once at the end) sounds like it saves the scarcer resource, but it doesn't — it just delays finding out you needed it, and late detection is the one thing SDLC cost curves punish hardest. Worse, a finding either reviewer surfaces can silently drop if it isn't explicitly tracked to resolution once the *other* reviewer's next round moves on to fresher issues — track every finding from both to an explicit outcome (fixed, or consciously deferred with a stated reason), never let it lapse just because attention shifted.

```
codex review -c 'model_reasoning_effort="high"' --base main
```
(or `--commit <sha>` / `--uncommitted`, matching whatever `code-reviewer` is reviewing). This runs directly via Bash, in your own context — Codex isn't a Claude agent and can't be dispatched through the Agent tool, and it doesn't need to be; it's a CLI call you make and read the output of, the same shape as the DoD gate.

**Two flag traps, both already hit more than once — check both before running this command, not after it errors:**
1. `codex review` has **no `-o`/`--output-last-message` flag** — that's `codex exec`-only. Redirect stdout to a file instead (`codex review ... > transcript.txt 2>&1`) and read the tail for the verdict.
2. The optional `[PROMPT]` argument (custom review instructions) **cannot be combined** with `--uncommitted`/`--base`/`--commit` — the CLI errors immediately if both are passed. Pick one: a diff-selection flag alone (the default review is usually informative enough on its own), or commit the change first and use `--commit <sha>` with no custom prompt if extra framing is genuinely needed. If the project provides a wrapper that enforces these flag shapes (e.g. a `scripts/codex-review.sh`), use it instead of typing the raw command. A mechanical guard beats prose.

**Record the resolved Codex model in `review-verdict.md` alongside the Claude-side model check above** — read the `model` line from `~/.codex/config.toml` (absent = "CLI default"). Same decorrelation-recording discipline applied to the other reviewer: a config drift on the Codex side (a different machine, an edited `config.toml`, a CLI version bump) is otherwise invisible until a review looks off after the fact, and an escalated hunt can burn several rounds against a wrong model before anyone would think to check.

At `model_reasoning_effort="high"`, this consistently exceeds a 120-second foreground timeout — dispatch it with `run_in_background: true` from the start rather than waiting for the automatic background fallback to kick in.

**Subprocess/agent completion notifications are not always reliable in either direction — verify against ground truth, not the notification's framing.** A "completed" task-notification for a backgrounded `codex exec`/`codex review` call fired 3 separate times while the process was still genuinely running (confirmed via `ps aux` and continued output-file growth) — before treating that result as final, check the process is actually gone (`ps aux | grep 'codex exec\|codex review'`) or confirm the output file has stopped growing across two checks a few seconds apart. The same caution runs the other way for `Agent`-tool dispatches: a "failed"/session-limit notification does not necessarily mean no work was produced — a planner agent's revision spec had already been fully written to disk before a session-limit error cut off its own final summary message, discovered only by reading the file directly instead of assuming the failure meant no output. Acting on either a premature "completed" or an overstated "failed" risks either reading a partial transcript as the final verdict, or throwing away real, finished work and re-dispatching from scratch.

Run it whenever `code-reviewer` runs, with one exception: skip it for pure mechanical changes (dependency bumps, one-line config edits) where there's structurally no logic for a second lens to have an opinion on. For everything else, don't gate it behind a severity judgment call ("is this scary enough to warrant it") — that heuristic misses real cases; a change that doesn't look high-stakes by category can still be exactly where an independent lens catches something the primary reviewer missed. Consolidate both reviewers' findings before the first fix round — one combined list to `engineer`, not two sequential rounds — then re-run both again only if either found something real.

NEEDS CHANGES verdicts (from either reviewer) loop back to the engineer with the findings. Re-review after fixes. Do not carry unresolved blocking findings forward to QA — that's paying for verification of code already known to be wrong.

This applies to every fix dispatch, including inside an escalated hunt: hand the engineer the finding — the defect and its evidence — not a prescribed fix mechanism. The engineer designs the fix, keeping fix-design and review decorrelated the same way engineer and code-reviewer already are; an orchestrator-prescribed mechanism that turns out wrong ships unchecked, since nothing downstream is set up to question the orchestrator's own design.

**When the orchestrator does specify a fix direction for a CTA-emphasis, visual-state, responsive-breakpoint change, or a shared/parent layout primitive other unrelated children depend on** (not the usual case per the paragraph above, but happens when the finding itself names a direction): trace one hop of downstream consequence before finalizing it — what does this value look like in the adjacent state (the other auth/subscription state), the adjacent breakpoint (one step wider/narrower), or — for a shared primitive — every other child that primitive already has?

**If an agent stalls or fails twice in a row on the same step — whether via a resumed thread or two fresh dispatches of the same task — stop retrying that approach.** For a resumed thread: dispatch fresh with tighter, more prescriptive instructions addressing what's actually blocking it (unchanged from the original rule — observed: the same engineer thread stalled twice in a row attempting the same empirical verification, a fresh dispatch with revised instructions completed the task in one pass). For two fresh dispatches that both stalled: try one lighter-weight variant first — explicit absolute file paths for every read, no reliance on Bash/shell tool access — before falling back to the orchestrator's own direct verification.

**If the project defines a `prompt-reviewer` agent (or equivalent) for AI-prompt/eval-gated surfaces, any diff touching that surface also gets it, in parallel with `code-reviewer` and `codex review`, not instead of either.** Prompt behavior is only observable by running the model, not by reading the diff — `code-reviewer` and `codex review` both read code and can approve a change that looks correct and silently regresses live behavior. A prompt-reviewer-equivalent agent runs the project's real eval harness and gates on its accepted baselines. Its NEEDS CHANGES verdicts loop back to the engineer the same as the other two; consolidate all reviewers' findings before the first fix round.

**Every reviewer's actual findings — code-reviewer, codex review, and especially prompt-reviewer's live eval numbers — must be written to `review-verdict.md` in the run folder as each review completes, not summarized from memory later.** A reviewer's real analysis living only in conversation history is invisible to `qa-verifier` (a fresh dispatch with no conversational access) and to `/retro`. Do this even when a review round is later superseded by a fix — the record of what was found and how it was resolved is exactly what makes the next stage's re-review efficient instead of a repeated derivation.

**A finding against a file the current diff doesn't touch is correctly out of scope for THIS fix,
but not for the backlog.** When a reviewer fully diagnoses a real, verified defect outside
the current diff's scope, log it to the project's durable backlog (this project's tracker) the same pass it's
found, not just in the run folder — the diagnostic cost is already paid; don't make a future
reviewer re-pay it.

**Escalate to a parallel-adversarial hunt once this branch shows the *serial* pattern** — 2+ real findings trickling in one per round across 3+ rounds (fix → re-review → one more thing…), not two reviewers converging on real findings in the same first round and both confirming the fix on re-review, which is the process working. Run the off-ramp check below FIRST. **When the trigger fires, read `references/escalated-hunt.md` (this skill's directory) in full before dispatching anything** — it holds the hunt protocol: 3-4 decorrelated lenses on the full cumulative diff, two-bucket convergence (release-affecting vs advisory-precision), the structural-fix question, the pre-redispatch fix check with `advisor()`, the small-function carve-out, failed-`Workflow` handling, and the cross-vendor rule. It stays an orchestration change only — no human checkpoint, no reduction in what gets reviewed; a genuine architecture/scope question surfaced mid-hunt still goes to the user.

**Off-ramp: distinguish "bug magnet" from "wrong tool for the job" before escalating.** Before dispatching an
escalated hunt, ask: do the findings so far share one root cause traceable to the chosen
approach/tool itself, such that no plausible amount of additional hardening converges? If yes, call
`advisor()` (or take it to the user) for a replace-or-cut decision instead of escalating — a
parallel hunt against an unsound approach mainly finds more instances of the same defect, at
multiplied cost. If findings look like genuinely distinct bugs on a real surface, escalate as
normal. This is distinct from the small-function carve-out below (which substitutes independent
verification for the hunt on a narrow, well-understood surface) — this one substitutes a
scope/approach decision for continued review entirely. If that structural fix itself keeps accumulating real findings across rounds, pre-commit a stop condition before dispatching the next round (see Stage 0.5 and `references/escalated-hunt.md`).

**Flagged experiment, lighter and earlier-firing than the escalation trigger above: when 2 consecutive rounds find a gap in the same underlying detection/classification function — not just anywhere in the diff, the same function — the next fix must generalize that function's check rather than patch one more specific input.** This is a fix-design principle, not a workflow escalation — it doesn't trigger the parallel hunt or add reviewers, it just changes what "the fix" for round N+1 is asked to be. When generalizing means replacing the mechanism, take the off-ramp. Extended once; revisit at the next retro that touches this file: promote off "experiment" if it fires with a generalizable remedy, narrow or drop it if it keeps colliding with the off-ramp or fires on unrelated gaps.

**Pattern-propagation check.** When a round's fix establishes a reusable invariant or fix shape — not just a one-off patch — the same round's dispatch must also check every sibling call site or path where the same invariant should apply, and record each as fixed, unaffected, or consciously deferred. A sibling rediscovered several rounds later is a wasted round, not a new finding. This includes sibling *dimensions* of the same resource, not just sibling call sites — a check that bounds one axis (byte count) can leave a correlated axis (entry count) on the exact same resource completely open, and the reviewer who finds the second axis often frames it as a brand-new finding when it's really the same gap recurring. Two independent instances in one review arc: a decompression guard was fixed for total bytes, then found unbounded on entry count, then found unbounded on *raw* pre-dedup entry count — three rounds for one resource; a database-id-based rate-limit key was normalized for case, then found still bypassable via a second spelling variant — the same equivalence gap on a second axis. Before calling a bounding fix complete, ask explicitly: what's the other way this same input could vary that this check doesn't yet cover?

## Stage 3 — Design review (design-strategist) — *if UI touched and the project defines this agent*

Any change to components, pages, styles, or user-visible layout goes to `design-strategist`. Advisory findings and future-work recommendations go into the PR body, clearly separated so they read as notes, not defects.

`design-strategist` drives a real browser (`claude-in-chrome`, if available) for the visual/journey checks — it isn't a code-only reviewer. If it reports the tooling itself as unavailable, that's a Blocking finding by design; don't proceed past it.

Blocking findings split into two kinds, and they route differently:
- **Mechanical** (wrong token, contrast failure, spacing off-rhythm, a value that should reference an existing variable) — loop straight back to the engineer, same as any other blocking finding.
- **Directional** (the design itself doesn't solve the problem — unclear hierarchy, a UX gap, "this needs a different approach") — invoke the `/design-solution` skill (if the project has one) with the stated problem before looping back to the engineer, instead of the orchestrator improvising a fix direction inline. `/design-solution` reads the project's actual design system and hands back an implementation-ready spec; give that to the engineer along with the original finding. If the fix touches or introduces user-facing copy, delegate `copywriter` from that same spec, then `editor` on copywriter's draft (same as Stage 0.5), before handing off to the engineer.

The same applies when the user describes a UX problem directly, without prescribing the fix (e.g. "the before/after isn't clear, propose a solution") — invoke `/design-solution` (and `copywriter` → `editor`, if new copy is needed) before Stage 1 rather than after, so the engineer builds from a concrete direction instead of guessing.

## Stage 4 — Narrative review (gtm-strategist) — *if user-facing and the project defines this agent*

Any change with user-visible copy, features, onboarding, or messaging surface goes to `gtm-strategist`. Only brand-promise violations (per the project's own stated positioning) block; everything else lands in the PR body as advisory.

## Stage 5 — QA verification (qa-verifier)

Hand `qa-verifier` the diff and the ACs from Stage 0. It must return per-AC evidence and an overall VERIFIED/REFUTED verdict. REFUTED loops back to the engineer with the reproduction steps, then re-verify. Never soften a REFUTED into "mostly done" — the verdict is binary by design.

**Write each pass to its own round-numbered file — `qa-evidence-round1.md`, `qa-evidence-round2.md`,
etc. — never a single `qa-evidence.md` a later pass overwrites.** A REFUTED verdict on round 1
followed by VERIFIED on round 2 is a real signal (something was wrong, then fixed) — Stage 8's
retro-trigger check depends on being able to see that a REFUTED verdict happened at all, and a
single overwritten file destroys that evidence the moment the re-run succeeds.

**Instruct `qa-verifier` to append to that round's file after each major AC or group of related ACs — not just once at the end.** A dispatch running many real operations (purchases, refunds, webhook calls) is exactly the kind of long-running task an external interruption (machine sleep, stream stall) can cut off mid-way; without incremental writes, a crash loses every AC already verified and forces a full redispatch from scratch, repeating operations that already succeeded once.

`qa-verifier` drives real interactions via `claude-in-chrome` (if available) — a DOM attribute (`href`, `onClick`) is never sufficient evidence on its own for an interaction-based AC.

## Stage 6 — Definition of Done gate

**If the project defines a DoD-gate skill that chains more than the bare CI commands** (testing review, a security spot-check, design QA — e.g. a `feature-done`-style skill), **invoke that skill rather than just running the bare commands yourself.** A DoD skill like this typically catches things the bare commands can't — a database-schema consistency check that only fails at runtime, a security spot-check, a full design-QA pass for UI changes — and the project's own top-level instructions (CLAUDE.md or equivalent) usually already require it before every commit. `code-reviewer`/`codex review` verify the diff against the spec, `qa-verifier` verifies the diff against the ACs — neither of those substitutes for a dedicated DoD skill's checklist, and `/ship` owns the commit, so this can't be treated as an already-covered upstream concern. Persist its full sign-off to the run folder; a Blocker/Critical finding from it blocks Stage 7 the same as a failed build does below.

If the project has no such skill, run its full DoD/CI check sequence yourself (not delegated, so the outputs land in your context) — typecheck, tests, lint, build, or whatever this project's own equivalent commands are.

All checks green or the task is not done. A failed gate is never skipped, worked around, or deferred to the PR — fix, then re-run.

## Stage 7 — Draft PR

Follow these in order — do not skip straight to "commit, push, open PR" once Stages 2-6 are green, even though that's the tempting shortcut once everything else has passed.

1. **Reconcile with current main before touching git at all, especially on a long-running branch.** Check `git log HEAD..origin/main --oneline`: if main has advanced since Stage 2's last drift check, rebase onto the current main, fix any resulting incompatibility (a shared type or fixture changing shape out from under an unrelated branch is a real, observed failure mode, not hypothetical), and re-run the full DoD gate before proceeding. A review loop that runs many rounds (fix ↔ re-review, or an escalated hunt) can span hours — long enough for unrelated work to land on main in the meantime, including a directly conflicting change to the exact file this branch touches. Skipping this because Stage 2 already checked once, or because this reads as background rationale rather than a literal action to take, is exactly the gap that let a merge-blocking conflict surface only after the PR was believed ready — this has recurred across multiple sessions this rule exists to prevent. **Also verify no local uncommitted drift has accumulated across the review loop itself** — a multi-round fix↔re-review cycle edits the working tree every round, but nothing in Stages 1-6 requires committing after each one; if only an early checkpoint commit exists, run `git status`/`git diff HEAD --stat` before opening the PR and commit anything still outstanding, rather than trusting one early "wip" commit from before the review loop began.
2. Branch (`feat/...` or `fix/...` per repo convention), commit, push with retry-on-network-failure, and open a **draft** PR.
3. **Immediately after opening the PR — confirm it's actually mergeable and that CI actually triggered, don't assume a successful push means either.** Run `gh pr view --json mergeable,mergeStateStatus` and `gh run list --branch <branch>`. A conflicting merge state (a concurrent merge to main landing on the same file mid-review, even after step 1's check passed earlier in the run) can silently prevent the required `pull_request`-triggered CI workflow from ever running at all — the platform can't compute the merge ref the workflow needs to check out, so no run ever queues, which looks identical to "CI just hasn't gotten to it yet" unless explicitly checked. If conflicting: resolve it the same way step 1 would have (merge/rebase main in — often a small, mechanical conflict even when both branches touch the same function, since unrelated hunks auto-merge; check before assuming a hard reconciliation is needed), re-run the full DoD gate, re-push, and re-confirm before reporting the PR as ready. "Draft PR opened" and "this PR can actually be merged" are different claims — don't report the first as if it implies the second.
4. If this repo has a separate live-model CI gate (e.g. a prompt/eval workflow, distinct from the deterministic CI check) and it's red: don't treat it identically to a deterministic check failure. Investigate root cause first — check the project's own documented model-noise history and re-run the affected cases in isolation (the same noise-discipline a prompt-reviewer-equivalent agent already applies) before deciding whether it reflects a real regression or accepted, pre-existing variance. Surface the finding and your reasoning to the user explicitly before merging past a red check either way — this is not a decision to make silently.
5. **If this PR adds or modifies a database migration on a project where merging auto-deploys (Vercel-on-merge, or equivalent): explicitly verify the migration is already applied to production before reporting the PR as ready to merge — do not rely on remembering this as a standing rule.** A written version of exactly this rule failing to prevent an incident (added to project memory one morning, violated by a merge that broke production hours later, the same day) is why this is a numbered Stage 7 step and not project-memory prose: a rule a human has to recall under time pressure is not a gate. State explicitly in the PR body whether the migration is (a) already verified applied to the live database, (b) needs to be applied before this PR merges, or (c) is intentionally deferred with a stated reason — never leave it unstated. If the project has (or is building) an automated CI/scheduled check for this, this step is a secondary human confirmation, not a replacement for it — treat the automated gate as the real backstop once it exists.

The PR body must contain:

- What changed and why (outcome first)
- The acceptance criteria and, for each, how it was verified (the qa-verifier evidence)
- Reviewer verdicts (code review, design, GTM where run) and any advisory findings
- Any deviations from the approved spec, and why
- What was consciously skipped or deferred, and why

## Stage 8 — Report back

**Retro-worthiness check, before the final message.** Every run gets one free, mechanical line in
the report: how many dual-review rounds Stage 0.5 ran (0 if it didn't trigger), whether Stage 2's
escalated hunt fired and for how many rounds, and how many deviations from the approved spec the
engineer disclosed. This costs nothing — it's read straight from artifacts already in the run
folder, not a new model call.

If this run crossed a threshold — 2+ dual-review rounds, 2+ disclosed spec deviations, Stage 2's
escalated-hunt trigger fired at all (regardless of how many hunt rounds it took to converge — easy
to miss since a clean Stage 0.5 and a clean first-pass QA give no other signal that anything heavy
happened), or **any** REFUTED `qa-verifier` verdict along the way (check every
`qa-evidence-round*.md` file in the run folder, not just the latest — a later VERIFIED round
doesn't erase an earlier REFUTED one) — and the project has a `/retro`-style skill, run it
scoped to this run's own run folder before the final message, rather than leaving it as something
the user has to remember to invoke later.

**When running several `/ship` tasks back-to-back in one continuous session (a queue of
pre-launch blockers, a batch of tickets), this checkpoint fires per task — immediately after
Stage 7 opens that task's PR, before starting the next queued task — not deferred to a single
end-of-session summary.**

Present whatever it proposes through its normal
one-at-a-time approval gate — this stays exactly as human-gated as that skill already is; only
*when* it runs changes, not whether the user approves each change. A clean run below the threshold
skips this. Approved changes land in the relevant file the same as any other output from that
process — that's what actually prevents a mistake recurring, not a separate lessons log; a rejected
proposal is recorded in that file's Provenance the same as always, so it isn't re-proposed next
time either.

Before the final message, check for other open PRs more than a few days old (`gh pr list --state open --json number,title,createdAt`, if using GitHub) and note any in the report — not the one just opened/updated this run. A correct fix sitting unmerged is indistinguishable from no fix at all. Surfacing it costs one command; the user decides whether to merge, close, or ignore.

Final message to the user: outcome-first summary, PR link, anything that needs their judgment. If the project's own ADR bar was met by a decision made during the work, write the ADR before ending the turn.

## Stage selection

Scale the pipeline to the change — a five-agent panel on a typo fix is bloat, and bloat is a bug in this system:

| Change type | planner | /design-solution | copywriter → editor | engineer | code-reviewer | codex review | design-strategist | gtm-strategist | qa-verifier | DoD |
|---|---|---|---|---|---|---|---|---|---|---|
| Backend / lib logic | ✓ | — | — | ✓ | ✓ | ✓ | — | — | ✓ | ✓ |
| UI feature / change | ✓ | if new direction needed | if new/substantial copy | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Copy-only change | — | — | if more than a tweak | ✓ | — | — | ✓ | ✓ | light | ✓ |
| Config / tooling / docs | — | — | — | ✓ | ✓ | — | — | — | light | ✓ |

"Light" QA = ACs verified without the full e2e battery. When in doubt, run the stage. Planner applies to the top rows only when the change is genuinely non-trivial — a one-line backend fix doesn't need a spec; a new endpoint does. `/design-solution` and `copywriter` are solutioning steps, not review steps — they're conditional on whether a direction/copy needs to be *produced*, not just implemented from something already clear; a UI change that already has an obvious fix or fully-specified copy skips both and goes straight to engineer. Whenever `copywriter` runs, `editor` runs — there's no copy-only-change row where copywriter fires without editor right after it, same as engineer never ships without code-reviewer. A humanizer-equivalent skill isn't a separate column: it runs automatically after `editor` approves, inside the same "copywriter → editor" block, whenever the project has one and that block runs at all. `codex review` runs wherever `code-reviewer` runs except "Config / tooling / docs" — mechanical changes are the one case with structurally nothing for a second lens to find. A prompt-reviewer-equivalent agent isn't a table column either — it's a diff-path trigger, not a change-type trigger: it runs whenever the diff touches the project's AI-prompt/eval-gated surface, regardless of which row the change otherwise falls under.

## Model tiering (80/20)

Spend the expensive model where an error is invisible until later or multiplies downstream; use the efficient tier where the thinking has already been written down and deterministic gates backstop the output:

- **Top tier (inherit session model, or the strongest available)**: planner and code-reviewer — plan errors compound through every stage, and adversarial bug-hunting is the hardest reasoning in the pipeline. The engineer also stays top-tier by default: it still carries design judgment whenever Stage 0.5 was skipped, and a fast-moving or unfamiliar framework version raises the floor. Revisit the engineer downgrade via `/retro` once spec-driven runs accumulate — if specced tasks show no rework loops, trial a cheaper tier there. `editor` is also pinned top tier — the same adversarial-judgment logic as code-reviewer, and pinning it opposite copywriter's cheaper tier guarantees the two are never the same model regardless of what the session itself is running, so the decorrelation is structural rather than incidental. A prompt-reviewer-equivalent agent is pinned top tier for the same reason — interpreting live eval output for a real-vs-noise regression call is adversarial statistical judgment, not disciplined execution against a checklist.
- **Efficient tier (a cheaper model, pinned in their definitions)**: qa-verifier, design-strategist, gtm-strategist, copywriter — disciplined execution against explicit criteria, backstopped by review immediately downstream (copywriter by `editor`, then `gtm-strategist` further downstream; design-strategist and qa-verifier by the DoD gate and the user's merge review). A solutioning step paired with a review step catches what the solutioning step misses, so the solutioning step doesn't need to be top-tier alone — the review step carries that weight instead.

`/design-solution` and `/scope` are skills, not agents — they run inline in the orchestrator's own context (whatever model is powering the current session), the same way most skills do. They don't get a separate model assignment, but both should operate under an explicit principal-level framing in their own instructions — the rigor comes from how they're asked to work, not from a model override the orchestrator can't apply to itself.

## Hard rules

- **Never merge or deploy without the user's explicit, in-the-moment permission for that specific PR.** Default behavior is unchanged: the pipeline ends at a draft PR, the user reviews and merges. If the user explicitly authorizes merging a named PR in the current conversation ("merge #32", "merge this"), the orchestrator may run the merge itself (`gh pr merge <n>`, matching whatever strategy the user specifies or the repo default). That authorization is scoped to the PR and moment it's given — it is never a standing grant, and a future run defaults back to stopping at a draft PR unless the user grants permission again. **Deploy is not covered by this exception and stays absolute** — a merge and a deploy are different blast radii (a PR is reversible pre-merge, a bad prod deploy is live and user-facing), so relaxing one does not relax the other.
- **After any merge — whether the user merges it or the orchestrator merges with granted permission — sync local state automatically, with no separate approval needed:** switch to `main`, pull, and delete the merged branch both locally and on the remote. This is pure bookkeeping on a branch that's already merged and already safe to discard, not a new authorization-requiring action, so it chains directly off the merge itself. Skip only if the branch is still needed for follow-up work in the same session. Before pulling, run `git status` in the main checkout — do not assume a clean tree. A repo with concurrent sessions can have unrelated, pre-existing uncommitted work sitting in the main checkout at any time, unconnected to the just-merged branch (observed: pulling after a merge hit a conflict against several files' worth of unrelated in-progress edits that predated the entire session). **If `git status` shows anything, stop and surface it to the user rather than auto-stashing and popping around it.** Uncommitted work sitting in a *shared* main checkout most likely belongs to a different concurrent session — silently manipulating another session's in-progress, unrelated edits, even with a plausible-sounding heuristic (favor the just-merged content, preserve the rest), is exactly the kind of cross-session collision worktree isolation exists to prevent elsewhere in this same protocol. Report what's there and let the user decide whether to stash it, check with its owner, or wait. **If checking out `main` fails specifically because it's already checked out in another worktree in this repo** (`fatal: 'main' is already checked out at '<path>'`) — a structural git-worktree lock, distinct from the uncommitted-work case above — don't force it. Still delete the merged branch on the remote (safe, independent of the local checkout); leave the local branch/checkout as-is and note in the report that local main-sync is deferred until that other worktree is free.
- Never skip a failed gate or launder a REFUTED verdict.
- Disclose every skipped stage and every consciously accepted risk in the PR body and the report. The system stays trustworthy only while its reports are.
