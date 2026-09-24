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
pronunciation gain for recitation-y delivery is a failure, not a partial win
— this got tested for real this session (see below) and the failure mode is
worse than "a bit of bleed": at high alpha it's total collapse into Quran
recitation, not a rock song.

**Design:** two LoRAs, merged offline: `W = W_base + 1.0·dW_style +
alpha·dW_pron`. v2 checkpoint itself is never touched. `alpha=0` must
reproduce v2 bit-for-bit — **verified live, PASS** (both hashes matched,
session 11). This invariant is what makes "no regression" true by
construction; trust it.

## Where things stand (as of session 14)

Dataset (6,280 dual-script pairs, 9 reciters, finalized session 7), training
(6,100-step AR-only LoRA, 4 checkpoints: 1525/3050/4575/6100=final), and the
merge tool (`merge_pron_lora.py`, rank-concatenation, alpha baked into a
separate file per value — no runtime slider exists in audio.cpp/ai-toolkit)
are all built, verified, and closed. Not open questions anymore.

**Session 13** generated a blinded 5-config × 4-maqam listening sweep
(configs: `a0`, `c3050_a0.5`, `c3050_a1.0`, `cfinal_a0.5`, `cfinal_a1.0`;
20 mp3s in `PRON_SWEEP_EVAL_INPUT/`, key in `KEY_open_after_listening.txt`).

**Session 14 — user completed the blind listening eval, decoded and
analyzed** (`PRON_SWEEP_EVAL_INPUT/my_evaluation.txt`). Result:

| config | avg score /5 | explicit "Quran bleed" tracks |
|---|---|---|
| a0 (baseline) | 3.0 | 0/4 |
| **c3050_a0.5** | **3.5 (best)** | **0/4** |
| c3050_a1.0 | 0.5 | 4/4 |
| cfinal_a0.5 | 3.125 | 1/4 |
| cfinal_a1.0 | 0.0 | 4/4 |

**Decision: `checkpoint=3050, alpha=0.5` is the winning config.** Beats
baseline on score, zero bleed flags, and fixed a specific baseline error
(بالله) that both the baseline and the final-checkpoint config got wrong.
Treat this as "good enough" per the user's own stated preference — a
narrower confirmatory sweep (e.g. alpha 0.4/0.6 at checkpoint 3050) is
optional polish, not required before shipping.

## Durable lessons (apply to future sessions/projects, not just this one)

- **Alpha near 1.0 is not "more flavor," it's mode collapse.** Every
  alpha=1.0 track (both checkpoints) scored 0 and was independently flagged
  as literal Quran recitation, several abnormally short (12–30s) — the
  model stops singing and just recites. Don't test alpha=1.0 again; the
  useful range is well below it.
- **Checkpoint matters as much as alpha, and now it's ear-confirmed, not
  just loss-inferred.** `loss/ar_kl` (AR drift) rises the *entire* training
  run and never plateaus, while `loss/ar_ce` (pronunciation signal) flattens
  early (~step 1500). Ear results now confirm the practical consequence:
  at matched alpha=0.5, checkpoint 3050 is clean but the final checkpoint
  (6100) leaks a bleed flag. **Prefer an earlier checkpoint over the final
  one** whenever a training run shows this ar_kl-keeps-rising shape.
- **A 0/5-style catch-all score plus free-text notes is a fine substitute
  for a rigid pre-built rubric.** The user skipped the structured
  `PRON_SWEEP_LISTENING_EVAL.md` checklist and just wrote a flat per-track
  eval — it still cleanly separated collapse from ordinary pronunciation
  variance (0 = "this is Quran," not a song). Don't over-engineer the
  eval template next time; a free-text pass + a blind key is enough.
- **Agent prose reports don't reliably survive intact — verify claims
  against actual files/hashes**, not the commit message or chat summary
  describing them. This has bitten the project multiple times before and
  is worth re-checking every time a "done, verified" claim shows up.
- **Delegate token- or compute-heavy work (large data, GPU runs, log
  analysis) to the user's separate coding agent** by writing a complete,
  copy-pasteable, ordered prompt — no placeholders, no back-and-forth. Do
  small/cheap things (a quick calc, reading one file) directly here.
  Writing implementation code directly in this WebUI session should be rare.

## Open thread (not alpha/checkpoint-related)

`في ذمة الله` came out wrong in **every** non-collapsed config, including
the untouched baseline (`a0`). That points to a caption/lyric-text issue in
that specific song, not a pron-LoRA effect — investigate the source lyric
text for that line independently; don't fold it into further alpha tuning.

## Next step

Bake the production merge at `checkpoint=3050, alpha=0.5` and treat the
pilot as successful. Optional, not required: a tight confirmatory sweep
(alpha 0.4/0.6, checkpoint 3050 only) before locking it in. Separately,
look at the `في ذمة الله` caption text.

## Maintaining this file

Update only when a real decision, rejected idea, corrected assumption, or
generalizable lesson lands. Keep it short — this file was cut down hard in
session 14 specifically because it had accumulated too much historical
narrative that's already safe in git history. Resist letting it regrow into
a session-by-session log; a link to the relevant commit/branch is enough
for anything that's just "what happened."
