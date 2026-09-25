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

## In flight: generation-knobs investigation (dispatched session 15)

Every track so far has only ever varied `alpha` and checkpoint step;
`run_one.sh` hardcodes LoRA scale, attention mode, and `cot=off`, and
leaves every yue2 sampling/guidance option at its audio.cpp default.
Dispatched a read-only, no-GPU-execution investigation (same format as the
earlier `docs/investigation.md`) to catalog what else exists — sampling
params (`semantic_temperature/top_p/top_k/repetition_penalty`),
`guidance_scale`, `num_inference_steps`, the untested checkpoint 1525,
whether `merge_pron_lora.py`'s tensor layout supports per-layer/selective
alpha, and whether `semantic_prefix` could anchor a generation's opening
away from recitation-style delivery.

**Target:** `docs/investigation_generation_knobs.md`, branch
`pron-lora-knobs-investigation` (main repo). **Report pending.**

## Next step

1. Bake the production merge now: `c3050_a0.5` primary, `c3050_a0.3`
   fallback. Not blocked by anything below.
2. When the knobs investigation report lands: spot-check its 1-2 strongest
   claims against the actual source/files before trusting them (see lesson
   above), then design — but don't yet run — a small, cheap empirical test
   of the top 2-3 candidate knobs (sampling params need no merge, so this
   is free; checkpoint 1525 needs one ~3.5s merge).
3. Dispatch the Kurd lyric-swap test (4 tracks: Kurd-lyric-on-Hijaz,
   Hijaz-lyric-on-Kurd, at a0 and c3050_a0.5) whenever there's listening
   bandwidth for it — independent of (2).
4. `في ذمة الله` lyric-text check stays a separate, low-priority task.

## Maintaining this file

Update only when a real decision, rejected idea, corrected assumption, or
generalizable lesson lands. Keep it short. Resist letting it regrow into a
session-by-session log; a link to the relevant commit/branch is enough for
anything that's just "what happened."
