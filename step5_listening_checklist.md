# Step 5 — Temperature × Alpha Diagnostic + Grid: Listening Evaluation Checklist

**Question:** does `t0.8`'s bleed (Step 2) come from the pron-adapter × low-temp
interaction, or from the base MaqamRock v2 LoRA's own latent tendency,
sharpened by low temperature? If the former, does a lower alpha recover the
pronunciation gain without the bleed?

**Fixed across all 8 tracks:** `semantic_temperature=0.8`, seed `20260924`,
Hijaz + Kurd prompts (same as Step 2 / fine sweep).

**Protocol:** 1-5 holistic score + explicit bleed flag (yes/mild/no) — score
and bleed are separate axes, don't infer one from the other.

**n note:** 1 track per config per maqam (8 tracks total). Enough to read the
alpha=0 fork and the direction of the 0.3→0.4→0.5 ladder, not enough to crown
a best alpha off a single delta — don't over-read one strong or weak track.

**Important — this batch was blinded together in one new key** (not reused
from Step 2), so `c3050_a0.5_t0.8` here is a fresh listen, not a copy of
Step 2's score for that config. Once the tracks + `KEY.json`/`KEYS.txt` land,
fill in the blind filename column from the key, then listen blind and fill in
the rest.

---

## alpha = 0 — no pron adapter, untouched v2 baseline (a0_t0.8)

*The diagnostic track. If this bleeds, the effect is temperature × base-v2
LoRA, independent of the pron adapter — stop here, the alpha ladder below
won't change that conclusion.*

- [ ] **Hijaz** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:
- [ ] **Kurd** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:

## alpha = 0.3 — fallback merge, existing (c3050_a0.3_t0.8)

- [ ] **Hijaz** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:
- [ ] **Kurd** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:

## alpha = 0.4 — new merge, baked this step (c3050_a0.4_t0.8)

- [ ] **Hijaz** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:
- [ ] **Kurd** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:

## alpha = 0.5 — primary merge, existing, re-rendered in this batch (c3050_a0.5_t0.8)

*Same config as Step 2's `t0.8` (Hijaz 4/5 no bleed, Kurd 4/5 mild bleed),
re-rendered here so it sits in the same blind batch as the other three
alpha points instead of being compared across separately-blinded sessions.*

- [ ] **Hijaz** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:
- [ ] **Kurd** — file: `_____________`
  - Score: `___/5`
  - Bleed flag: ☐ yes ☐ mild ☐ no
  - Notes:

---

## Decision rule (apply after all 8 are scored)

- **If `a0_t0.8` bleeds on either maqam** → the effect is temperature ×
  base-v2-LoRA, independent of the pron adapter. Lowering alpha will not
  remove it. **Shelve the alpha-reduction idea**, log it closed with this
  finding. The 0.3/0.4/0.5 tracks are still useful context (they show
  whether alpha modulates severity at all) but don't reopen the idea on
  their strength alone.
- **If `a0_t0.8` is clean on both maqams** → the bleed really is an
  alpha × pron-adapter interaction. Read the ladder **0.3 → 0.4 → 0.5**
  for where bleed clears while score holds **≥3.5/5** on both maqams.
  - If some alpha clears bleed at ≥3.5/5 → real candidate. Log as an open
    thread in `claude_context.md` for a proper n>2 validation — n=1 per
    point is a screening pass, not a validation, same rule as Step 2.
  - If all three (0.3/0.4/0.5) still bleed → no alpha floor in this range
    removes it. Drop the temperature-knob idea.
