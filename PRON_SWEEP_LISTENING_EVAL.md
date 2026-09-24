# Pron-LoRA Alpha Sweep — Listening Evaluation Checklist

**Purpose:** go through all 20 tracks systematically instead of vibes-listening, and
end up able to say which config (pron checkpoint × alpha) wins **without knowing
which is which until you're done.**

**Your working hypothesis going in (write it down so you can check it against
what you actually hear):** the pron adapter should act like a **spice, not a
main ingredient** — a pinch, not a cup. If 0.5 already sounds like it's carrying
recitation flavor, that's a real finding, not a failed run — it tells us the
useful range is probably **below** what this sweep tested (0.5 / 1.0), and a
follow-up sweep at 0.1–0.2 is the next move, not "turn it back down and hope."

**Rule while listening: don't peek at `KEY_open_after_listening.txt` or
`KEY.json` until every track in a maqam is scored.** Score blind, reveal after.

---

## How to score each track

For every track, you're judging two *separate* things — don't let one bleed into
the other in your head:

1. **Pronunciation clarity** — did ح, خ, ع, ض sound crisp and distinct, or did
   they drift (ح→ه, ع→أ, ض losing its emphatic quality)? This is the thing the
   adapter is *supposed* to improve.
2. **Style/cadence integrity** — does it still sound like MaqamRock, or does it
   sound like it's slipping toward recitation (elongated vowels, tajweed-ish
   pacing, wrapping up early, breathy pauses)? This is the thing the adapter is
   *not allowed* to touch. If this score is bad, a high pronunciation score
   doesn't save the config — the whole design exists to keep this side clean.

Score both **1–5**:

- **Pronunciation clarity:** 1 = muddy/wrong letters throughout · 3 = mixed,
  some clear some drifted · 5 = consistently crisp ح/خ/ع/ض
- **Style integrity (no bleed):** 1 = clearly sounds like recitation bled in ·
  3 = a hint of it, not sure if I'm imagining it · 5 = indistinguishable from
  normal MaqamRock delivery

A track only "wins" if it's good on **both** axes. A 5/1 (perfect letters,
heavy bleed) is not a good result for this project.

---

## Maqam: Hijaz

Tracks (blinded labels — fill in as you listen, don't reorder):

### Track A
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments that stood out (good or bad, with rough timestamp): 
  - 
- Bleed moments (stretched vowel, recitation-y pause, wraps up early), with timestamp:
  - 
- Anything else notable:

### Track B
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments that stood out:
  - 
- Bleed moments:
  - 
- Anything else notable:

### Track C
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments that stood out:
  - 
- Bleed moments:
  - 
- Anything else notable:

### Track D
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments that stood out:
  - 
- Bleed moments:
  - 
- Anything else notable:

### Track E
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments that stood out:
  - 
- Bleed moments:
  - 
- Anything else notable:

**Before revealing the key — rank A-E, best to worst, on each axis:**
- Pronunciation clarity ranking: ___ > ___ > ___ > ___ > ___
- Style integrity ranking: ___ > ___ > ___ > ___ > ___
- My overall pick for Hijaz (before reveal): ___

---

## Maqam: Kurd

### Track A
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track B
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track C
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track D
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track E
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

**Before revealing the key — rank A-E, best to worst, on each axis:**
- Pronunciation clarity ranking: ___ > ___ > ___ > ___ > ___
- Style integrity ranking: ___ > ___ > ___ > ___ > ___
- My overall pick for Kurd (before reveal): ___

---

## Maqam: Nahawand

### Track A
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track B
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track C
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track D
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track E
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

**Before revealing the key — rank A-E, best to worst, on each axis:**
- Pronunciation clarity ranking: ___ > ___ > ___ > ___ > ___
- Style integrity ranking: ___ > ___ > ___ > ___ > ___
- My overall pick for Nahawand (before reveal): ___

---

## Maqam: Ajam

### Track A
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track B
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track C
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track D
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

### Track E
- Pronunciation clarity (1-5): ___
- Style integrity / no bleed (1-5): ___
- ح/خ/ع/ض moments:
  - 
- Bleed moments:
  - 
- Notes:

**Before revealing the key — rank A-E, best to worst, on each axis:**
- Pronunciation clarity ranking: ___ > ___ > ___ > ___ > ___
- Style integrity ranking: ___ > ___ > ___ > ___ > ___
- My overall pick for Ajam (before reveal): ___

---

## Reveal + roll-up (fill in after all 4 maqams are scored blind)

Open `KEY_open_after_listening.txt` / `KEY.json` now, and map each label back to
its config. Then transfer your scores into this table (5 configs × 4 maqams,
both axes):

| config | Hijaz clarity | Hijaz bleed | Kurd clarity | Kurd bleed | Nahawand clarity | Nahawand bleed | Ajam clarity | Ajam bleed | avg clarity | avg bleed |
|---|---|---|---|---|---|---|---|---|---|---|
| a0 (baseline) | | | | | | | | | | |
| c3050_a0.5 | | | | | | | | | | |
| c3050_a1.0 | | | | | | | | | | |
| cfinal_a0.5 | | | | | | | | | | |
| cfinal_a1.0 | | | | | | | | | | |

**Questions to answer once the table's filled in:**

1. **Does a0 (the v2 baseline) already score reasonably on clarity in your ear,
   or is the "occasionally wrong pronunciation" problem clearly audible there?**
   This is your reference point — everything else should be judged as a delta
   from this, not in isolation.
2. **Checkpoint 3050 vs final, holding alpha fixed** — session 12's val-loss
   numbers suggested 3050 might have most of the pronunciation gain with less
   drift than final. Does that hold up by ear? (Compare `c3050_a0.5` vs
   `cfinal_a0.5`, and separately `c3050_a1.0` vs `cfinal_a1.0`.)
3. **Alpha 0.5 vs 1.0, holding checkpoint fixed** — is the clarity gain from
   1.0 actually worth the bleed cost, or does 0.5 get most of the benefit for
   less style risk? (Compare `c3050_a0.5` vs `c3050_a1.0`, and
   `cfinal_a0.5` vs `cfinal_a1.0`.)
4. **Is there a maqam where bleed shows up much more than the others?** If one
   maqam is consistently the worst on style integrity across all four non-baseline
   configs, that's a maqam-specific interaction worth flagging, not just a config
   ranking.
5. **The "spice, not ingredient" call:** does *any* config here land in a zone
   you're happy with, or does even the lowest tested alpha (0.5) already feel
   like too much? If it's the latter, that's the case for a follow-up sweep at
   0.1–0.2 rather than picking a "least bad" option from this batch.

**My overall conclusion / next step:**

