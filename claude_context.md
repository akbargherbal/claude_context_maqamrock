# claude_context.md — Pronunciation-LoRA Project Memory

**Purpose:** written by Claude, for Claude, to restore context in a future
*stateless* session. Full history lives in git commits on both repos —
don't re-narrate it here. This file holds only: where things stand, the one
scope invariant that must never drift, durable lessons that generalize
beyond a single session, and the concrete next step. If it's greppable from
the repos, it doesn't belong here.

**Repos** (re-clone each session — container state doesn't persist):
- Main: `https://github.com/akbargherbal/maqamrock-yue2-lora-finetuning.git`
  — branch `pron-lora-ar-only` (never commit pron work to `main`)
- Secondary: `https://github.com/akbargherbal/arabic-phoneme-difficulty-quran.git`
  — branch `pron-lora-prep`

---

## The goal — do not let this drift

MaqamRock's shipped LoRA (v2) is mature/stable (8.5/10). This side-project's
*only* target is crisper ح/خ/ع/ض articulation inside the existing MaqamRock
style. It is explicitly **not** about giving MaqamRock any recitation
flavor (melisma, tajweed cadence, elongated endings). Any result that trades
pronunciation gain for recitation-y delivery is a failure, not a partial win.

**Design:** two LoRAs, merged offline: `W = W_base + 1.0·dW_style +
alpha·dW_pron`. v2 checkpoint itself is never touched. `alpha=0` reproduces
v2 bit-for-bit — **verified live, PASS** (session 11). Trust this invariant.

## Where things stand (as of session 15) — decision made, ready to ship

Dataset, training (6,100-step AR-only LoRA, checkpoints 1525/3050/4575/6100),
and the merge tool are all built, verified, and closed — not open questions.

**Session 13** ran a blinded 5-config × 4-maqam sweep. **Session 14** decoded
it: `checkpoint=3050, alpha=0.5` beat the untouched baseline (3.5 vs 3.0
avg) with 0/4 bleed flags; alpha=1.0 collapsed at both checkpoints.

**Session 15** ran a fine sweep around that result — alpha ∈
{0.2, 0.3, 0.55, 0.65} at checkpoint 3050, Hijaz + Kurd only — to check for
a hidden peak above 0.5. **Finding: no peak.** Scores don't move
monotonically with alpha; the 0.65 tracks scored highest on a holistic
1-5 scale, but the listener's own free-text notes flagged one of them as
audibly Quranic-recitation delivery. Bleed flags first appear at 0.55.

**Decision: ship `c3050_a0.5` as the production merge; bake `c3050_a0.3` as
a cheap fallback** (similar score, 0 bleed, more margin below the 0.55
bleed-onset point). **Do not test alpha ≥ 0.55 again** — it's past the
useful range, same conclusion as the earlier alpha=1.0 collapse finding.
This is independent of the open knobs investigation below and can proceed
now.

## Correction to a prior entry

Session 14 recorded "every alpha=1.0 track scored 0." Checked against the
actual eval this session: 7/8 did; `Kurd_D` (`c3050_a1.0`) scored 2/5 with
an explicit bleed flag, not 0. The collapse conclusion and the ~0 average
both still hold — only the "every track" wording was wrong. Lesson below.

## Durable lessons (apply to future sessions/projects, not just this one)

- **A holistic 1-5 "rate it as a song" score does not penalize recitation
  bleed.** The single highest-scoring fine-sweep track had a listener note
  describing it as clearly Quranic in delivery. Score and bleed are two
  different axes — track bleed as an explicit yes/mild/no flag per track,
  don't infer it from the number.
- **At n=1 track per alpha step (0.05–0.1 spacing), score deltas of 1-2
  points are noise, not a peak.** Don't chase a local max in a fine sweep
  unless it replicates (repeat seed) or the design expected noise.
- **Re-verify agent/session summary claims against the actual data before
  repeating them** — even a self-authored one-line summary ("every track
  scored 0") can drift from what the eval file actually says. Caught once
  already (session 14→15); treat any absolute claim ("every", "always") in
  this file as worth a quick recheck before leaning on it.
- Prior lessons still hold: prefer an earlier checkpoint over the final one
  when `ar_kl` never plateaued in training; delegate compute/token-heavy
  work to the user's coding agent with a complete, scoped, copy-pasteable
  prompt; verify "done" claims against files/hashes, not prose.

## Open threads

- **Kurd underperformance** (2.5→3.0→3.5 across a0/c3050_a0.5/cfinal_a0.5,
  weakest maqam at every config tested). Caption density is ruled out.
  Confounded with "only one held-out song per maqam" — never separated
  maqam-specific weakness from this-particular-song weakness. **Planned
  test (not yet dispatched):** swap lyrics — generate Kurd's caption+maqam
  with the Hijaz held-out lyric, and Hijaz's caption+maqam with the Kurd
  lyric, at a0 and c3050_a0.5. If Kurd stays weak with an easy lyric, it's
  maqam-specific; if the weakness follows the lyric, it's a test-song
  artifact.
- **`في ذمة الله`** came out wrong in every non-collapsed config *including
  the untouched baseline* — a caption/lyric-text issue unrelated to the
  pron-LoRA. Investigate the source lyric text for that line independently;
  don't fold it into alpha/checkpoint tuning.

## Generation-knobs investigation — done, report frozen (session 16)

The read-only investigation dispatched session 15 landed on
`docs/investigation_generation_knobs.md`, branch
`pron-lora-knobs-investigation` (main repo), commit `6478a43`. **Session 16
verified it**: spot-checked its most load-bearing claims (the `guidance_scale`
1.01-effective-default, the CFG negative-prefix construction) directly
against a fresh clone of `0xShug0/audio.cpp` at the pinned commit
(`ac16661d…`) — both confirmed exactly as described, source-level, not just
against the doc's own citations. Cross-checked repo-internal claims
(`ar_ce`/`ar_kl` checkpoint table, `merge_pron_lora.py`'s rank-concat logic,
the "generic CLI flags are no-ops" claim) against `DECISIONS.md`/
`PROGRESS.md`/source — no contradictions found.

A subsequent housekeeping/docs-reconciler pass (still session 16) froze the
report (added to `DEFAULT_EXCLUDES`, matching `docs/investigation.md`'s
treatment) because its citations resolve to the external `audio.cpp` repo,
which the reconciler can't check against this one. **Re-verified after that
pass: zero-line diff on the report itself; none of the files it cites content
from were touched.** The report stands as accurate. It will not be edited
again — new findings from the follow-through work go into this file or a new
doc on `pron-lora-ar-only`, not back into the frozen file.

## Next step — see `PLAN_generation_knobs.md` (this repo)

That file has the full modular breakdown (dispatch prompt + done-when +
decision rule per step) for: baking the production merge, rendering
checkpoints 1525/4575, the sampling-knob probe, the Kurd lyric-swap test, and
the `في ذمة الله` lyric-text check. Read it before dispatching anything below
— it's the source of truth for exact commands; this section only tracks
**which step is current**.

**Status as of session 16: no step yet dispatched.** Start with Step 0 (bake
— unblocked) and Step 1 (checkpoint 1525/4575 — higher-value than the
sampling probe since it tests the repo's own named drift risk, and its
outcome decides what checkpoint Step 2 anchors to). Steps 3 and 4 are fully
decoupled and can slot in anytime.

When a step's dispatch lands: pull `pron-lora-ar-only`, verify against that
step's "Done when" criteria in the plan file, listen/score if the step
produced tracks, then update this section with the outcome and which step
is current next — do not edit `PLAN_generation_knobs.md` itself for routine
progress, it's the stable plan, not a log.

## Maintaining this file

Update only when a real decision, rejected idea, corrected assumption, or
generalizable lesson lands. Keep it short. Resist letting it regrow into a
session-by-session log; a link to the relevant commit/branch is enough for
anything that's just "what happened."
