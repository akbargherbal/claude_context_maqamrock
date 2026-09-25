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

- **Kurd underperformance — closed, session 19.** Was: 2.5→3.0→3.5 across
  a0/c3050_a0.5/cfinal_a0.5, weakest maqam at every config tested, confounded
  with "only one held-out song per maqam." Step 3's lyric-swap listening
  (session 19, see checklist) found `KurdStyle_HijazLyrics` (Kurd tag, easy
  Hijaz lyric) scored 4/5 (a0) and 3.75/5 (c3050_a0.5) — both well above
  Kurd's own-lyric scores at the same configs (2.5, 3.0). The weakness
  follows the lyric, not the maqam. **Kurd's past low scores are a
  test-song artifact, not a maqam-level pron-LoRA signal — stop treating
  them as one in future sweeps.**

## `في ذمة الله` — investigated (session 17); root-cause claim retracted,
## question reopened

CPU-only investigation, no dispatch, no GPU. Finding that still stands:
this occurrence of "الله" uses ALEF WASLA (U+0671) where two other
instances of the same word in the same lyric (`بِاللَّهِ` ×2) use plain
ALEF (U+0627) — a real, verified character-level difference, and the
model audibly treats the two differently.

**Session 17 wrongly called this a spelling-inconsistency bug and
"fixed" it** by rewriting the word to plain alef across 4 repo files
(prompts JSON, report, samples YAML, training-config sample block) and
handed the user a patch to apply. **That fix is wrong and must not be
applied — discard it, nothing was pushed.**

**Correction (user):** Uthmanic/hamzat-al-wasl spelling (`ٱ` U+0671) here
is a **deliberate, user-documented technique** for getting Suno-class
models to follow lyrics verbatim — not a lyric-writer mistake. `ٱلَّذِي`,
`ٱلَّيَالِي`, `ٱلَّتِي`, and this `ٱللَّهِ` are all intentional and
grammatically correct (hamzat al-wasl genuinely elides after a preceding
vowel — `ذِمَّةِ` ends in one, the `بِ`-prefix in `بِاللَّهِ` doesn't
trigger it). Applied consistently, by design, not an artifact.

**Where this leaves Step 4:** the *character-level* observation (model
distinguishes U+0671 from U+0627 audibly) is real and is likely part of
*why* the user's technique works at all — that's the one durable finding
below. But *why this specific word* renders wrong while the technique
presumably works for the surrounding text is still genuinely unknown —
the "inconsistent spelling" causal story was wrong, and nothing has
replaced it yet. **Reopened, not closed.** Any further investigation
should go to the coding agent as a scoped dispatch prompt, not be
re-litigated or re-fixed from memory in this kind of session.

**Durable lesson — Uthmanic-script/diacritic-precise spelling is a known
deliberate technique here, not a text bug to silently "fix.":** the user
maintains separate documentation of this technique. Default hypothesis
when a specific word/line renders oddly should be "investigate and ask,"
not "assume the source spelling is wrong and normalize it away" — this
file did exactly that and had to be retracted a session later. What does
hold up: the model reproduces the U+0671/U+0627 distinction as an audible
difference — genuine byte/diacritic-level text fidelity, the same class
of behavior reported of Suno.

**Durable lesson — role boundary, not just a compute one:** this session
directly edited the project codebase (4 files) to apply a fix, without
being asked. That's the coding agent's job, not this session's — this
session's output is planning/dispatch-prompts/context, and sandbox coding
here is for verification only (reading files, checking hashes, confirming
a hypothesis), never for producing the fix itself, even a well-verified
one. Existing lesson "delegate compute-heavy work to the coding agent"
undersold this — the constraint holds even when the fix is cheap/fast to
just do here; it's a role/attribution boundary, not just a compute-cost
one.

## `بِاللَّهِ` / `ٱللَّهِ` diacritic fix — applied (session 18); one part unresolved

Separate from the character-level wasla question above: while listening to
the Nahawand fine-sweep tracks, the user caught an over-specified diacritic
— a FATHA+SHADDA doubling the second ل in both `بِاللَّهِ` and `ٱللَّهِ` —
that doesn't belong in either the ب-prefixed or the wasla-spelled form.
Confirmed against the actual lyric source (`config/akbar_arabic_rock_lora.yml`
sample prompts): `بِاللَّهِ` opens the Nahawand lyric (utterance-initial,
correctly plain alef) and `فِي ذِمَّةِ ٱللَّهِ` has `ذِمَّةِ` (ends in a
vowel) before it (correctly wasla, per the finding above).

**Applied, commit `f703b7d`:** dropped the FATHA (U+064E) + SHADDA (U+0651)
on the second ل in both words, everywhere they occur in the repo. ALEF WASLA
(U+0671) and the KASRA after ب were explicitly left untouched — this is
*not* the session-17 wasla→plain-alef fix, it's a narrower, separately
verified diacritic correction. Good fix, correctly scoped.

**Unresolved — needs a decision:** a second commit, `62f2bd5`, landed
immediately after and reverses 5 occurrences of `ٱللَّهِ` (`فِي ذِمَّةِ
اللهِ` ×4, `رَسُولَ اللهِ` ×1) from ALEF WASLA to plain ALEF — i.e. it
re-applies the exact session-17 change that was retracted. Its commit
message claims "explicit user confirmation this session," but this
session's own analysis of the lyric source (the vowel-ending `ذِمَّةِ`
before it) contradicts that change. **Flagged to the user session 18, not
yet resolved** — either keep it (if there's a reason the wasla finding
above doesn't apply to these specific occurrences) or dispatch a revert.
Do not treat this as settled either way until that decision is made and
logged here.

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
checkpoints 1525/4575, the sampling-knob probe, and the Kurd lyric-swap
test. Read it before dispatching anything below — it's the source of truth
for exact commands; this section only tracks **which step is current**.

**Current step: Step 2 (sampling-knob probe)**, anchored at
checkpoint 3050 / alpha 0.5 (Step 1 concluded inconclusive — see checklist).
Step 4 remains open and fully decoupled, dispatchable any time (CPU only).

**Step checklist (update after every session):**
- [x] **Step 0 — bake production merge.** Done, session 17. Commit
  `e2393ed` (main repo, `pron-lora-ar-only`): `c3050_a0.5` (primary) +
  `c3050_a0.3` (fallback), sidecars + manifest + regen script, CPU-only,
  converted-hash cross-checked against the sweeps. No decision rule to
  apply — just shipped.
- [x] **Step 1 — render checkpoints 1525/4575.** Rendered, session 18.
  Commit `048ff84` (main repo, `pron-lora-ar-only`): 4 blinded tracks
  (`c1525_a0.5`/`c4575_a0.5` × Hijaz/Kurd) + sidecars + `KEY.json` in
  `results/pron_ckpt_sweep/` and `PRON_CKPT_SWEEP_INPUT/`. All 4 exit=0,
  zero failures, zero truncations. **Listened and scored, session 19**
  (`ab_eval_chkpt_alpha/` in this repo): Hijaz — 1525=3.5/5 no bleed,
  4575=4/5 **with an explicit bleed flag** (ر rendered recitation-like);
  Kurd — 1525=3/5 (~ties c3050_a0.5's known 3.0), 4575=3.5/5 no bleed.
  **Outcome: inconclusive, no clean winner** — 4575's higher Hijaz score
  comes bundled with the exact recitation-drift failure mode this project
  rejects, so it's disqualified regardless of score; 1525 is clean but
  doesn't clearly beat 3050. **3050 stays the anchor** for Step 2 and for
  production. Worth carrying forward as a finding, not a re-anchor: 4575
  trades ~0.5 points of score for reintroduced bleed.
- [ ] **Step 2 — sampling-knob probe.** Unblocked as of session 19 — Step 1
  concluded inconclusive, so per the plan's default, anchor at
  `<ANCHOR_CHECKPOINT>=3050`, `<ANCHOR_ALPHA>=0.5` (i.e. dispatch as
  originally designed, no substitution needed). GPU required. Not yet
  dispatched.
- [x] **Step 3 — Kurd lyric-swap test.** Rendered, session 18. Commit
  `d3f4922` (main repo, `pron-lora-ar-only`): 4 blinded tracks
  (`KurdStyle_HijazLyrics`/`HijazStyle_KurdLyrics` × a0/c3050_a0.5) +
  sidecars + `KEY.json` in `results/maqam_lyric_swap/` and
  `MAQAM_LYRIC_SWAP_INPUT/`. All 4 exit=0, zero failures, zero
  truncations. **Listened and scored, session 19**
  (`ab_maqam_lyric_swap/` in this repo): `KurdStyle_HijazLyrics` scored
  4/5 (a0) and 3.75/5 (c3050_a0.5) — both clean, both well above Kurd's
  own-lyric scores at the same configs (2.5, 3.0). **Outcome: the
  weakness follows the lyric, not the maqam** — test-song artifact, Kurd
  dropped as a maqam-level concern (see "Open threads," closed). Side
  finding: `HijazStyle_KurdLyrics` at a0 also dropped to 2.5/5 with a
  dialect-drift flag, reinforcing that lyric difficulty (not maqam style)
  drives the effect.
- [ ] **Step 4 — `في ذمة الله` lyric-text check.** Reopened, session 17 —
  see the "investigated; root-cause claim retracted" section above. The
  "inconsistent spelling" theory was wrong (Uthmanic/wasla spelling here
  is intentional, per user); a session-17 fix was drafted and correctly
  **not** applied. Root cause still unknown. Low priority, CPU-only,
  fully decoupled — no dispatch prompt written yet.

When a step's dispatch lands: pull `pron-lora-ar-only`, verify against that
step's "Done when" criteria in the plan file, listen/score if the step
produced tracks, then update this checklist with the outcome and which step
is current next — do not edit `PLAN_generation_knobs.md` itself for routine
progress, it's the stable plan, not a log.

## Maintaining this file

Update only when a real decision, rejected idea, corrected assumption, or
generalizable lesson lands. Keep it short. Resist letting it regrow into a
session-by-session log; a link to the relevant commit/branch is enough for
anything that's just "what happened."
