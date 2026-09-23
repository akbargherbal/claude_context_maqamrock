# claude_context.md — Pronunciation-LoRA Project Memory

**Purpose:** This file is written by Claude, for Claude, to restore context in a
future *stateless* session. It is not project documentation — the two repos
already document themselves. This file holds the things a repo *can't* easily
tell a fresh session: decisions made, ideas rejected and why, open questions
left hanging, and the scope guardrails that keep the project from drifting.
Paste this at the start of the next session and treat it as "what I already
know," not "background reading."

**Repos** (re-clone each session — container state doesn't persist):
- Main: `https://github.com/akbargherbal/maqamrock-yue2-lora-finetuning.git`
- Secondary: `https://github.com/akbargherbal/arabic-phoneme-difficulty-quran.git`

---

## The goal, stated precisely (do not let this drift)

MaqamRock's current LoRA (`akbar_arabic_rock_lora`, v2) is **8.5/10 and
considered mature/stable**. The user is *not in a hurry* (expects 3-4 sessions)
and explicitly does not want more effort than necessary once "good enough" is
reached.

The target is narrow: **crisp consonant articulation** — ح sounds like ح not
ه, ع doesn't drift to أ, ض stays distinct — inside the *existing* arabmaqamrock
musical style. This is explicitly **not** about making MaqamRock adopt Quranic
recitation as a style/performance. Any design choice that risks importing
recitation "flavor" (melisma, tajweed cadence, elongated phrase-endings) is a
defect to guard against, not a byproduct to shrug at. This scope guardrail
came up repeatedly and is the thing most likely to get accidentally relaxed
under momentum — watch for it.

## The plan we landed on

Train a **separate, small, AR-only-scoped LoRA** on Quran verse audio + text,
targeting only the problem letters, then **merge offline**:
`W = W_base + 1.0·dW_style + alpha·dW_pron`. Never retrain or touch the
current v2 checkpoint. Alpha is a fully controllable dial, including 0.

**Non-negotiable invariant:** the merge tool must be verified so `alpha = 0`
reproduces the current v2 weights bit-for-bit before it's trusted for
anything. This is what makes "no regression" true by construction rather than
by care.

We chose this over "just train maqamrock further" deliberately:
`loss/ar_kl` rose **unbroken** through all 3000 steps in both v1 and v2 (never
plateaued) — an unvalidated lever, not a safe dial to lean on. "Train more"
also has no stopping rule (4000? 5000? why?). The isolated pilot gives a clean
answer either way (works / doesn't / mixed) and *then* tells you whether
further maqamrock training would even be about pronunciation, or just style.

## Rejected/superseded ideas (don't re-propose these)

- **A binary "representational ceiling" pre-check** (base model, no LoRA — can
  it *ever* produce a correct ح?) as a gate before running anything. Rejected:
  doesn't match reality. The user's pronunciation is "occasionally wrong," not
  "never right," and was already observed improving with training steps. Also
  now moot — see next point.
- Independent evidence the ceiling isn't the risk: the user noticed the
  *existing* MaqamRock LoRA is drifting toward pronouncing `عمرو`'s silent
  waw-al-fariqa as a spoken letter — literal, grapheme-level sensitivity. That
  means fine orthographic distinctions are clearly representable already.
  Good news for the whole plan, but also means the model has **no built-in
  judgment about what wasn't meant to be heard** — it will just as faithfully
  sing a stray waqf mark or copy-paste artifact. This raises the importance of
  clean input text, doesn't lower it.
- **A quantified substitution-rate scorecard as a gate before deciding to run
  the pilot at all** — downgraded. Still worth building, but as the pilot's
  *grading method* (before/after, a few alpha values), not a prerequisite to
  starting.

## Data/methodology decisions

- **Reciter selection:** avoid Mujawwad (melismatic/ornamented) recitations as
  donor audio — highest risk of "flavor bleed." Prefer Murattal or Muallim
  (measured, teaching-style) recitations. `Husary_Muallim` flagged as a good
  candidate. Use multiple reciters, not one, so the adapter generalizes the
  phoneme contrast rather than memorizing one voice.
  Bucket also holds `warsh/` — EXCLUDE: different riwaya from Tanzil's Hafs
  text (audio/text mismatch; verse numbering may differ too). Exclude both
  Mujawwad dirs. Bucket has `_catalog.json`, `_metadata/`, `_registry/`: read
  them before classifying reciters by folder name alone.
- Available donor data: GCS bucket
  `gs://sheikh-fitzgerald-backup/ARABIC_DATA/QURAN_VERSE_BY_VERSE_RECITATIONS_DATASETS/`,
  per-verse files named `SSSAAA.mp3` (surah+ayah), many reciters, both
  Mujawwad and Murattal/Muallim present.
- **Clip length:** verses (≤~12 words, often <1 min) are *not* a drawback.
  Confirmed from the main repo's own config: `train_window_frames: 0` already
  means whole-clip, variable-length training — verses are a shorter instance
  of the existing regime, not a new one. Only watch-item: uniformly short
  clips could in principle teach a "wrap up early" habit if merged at high
  alpha — not a blocker, just something the eval listening step should check.
- **Difficulty scorer:** default `letter_weights` in
  `aya_scoring/weights.py` spread weight evenly across all guttural/emphatic
  letters. For this project, override to specifically upweight ح خ ع ض (the
  user's actual named problem letters), and sort candidates by
  `density_score` (score/word_count), not raw score, to surface short,
  phonetically-dense ayat.
- **LoRA capacity:** keep rank low on the pronunciation adapter deliberately —
  a second, independent defense (alongside AR-only scoping) against the
  adapter learning prosody/cadence instead of just phoneme selection.

## Caption/text format decisions

- Real v2 training caption format (confirmed from user's own
  `notebook.md` sample, since `prepare_yue2_dataset_v2.py` — the actual
  build script used — is **not committed** to the repo, only v1's
  style-only version is): one `.txt` per audio = style/production paragraph,
  then a literal `[Lyrics]` header line, then a section-tagged lyric block
  (`[Intro]`, `[Verse 1]`, `[Chorus]`, etc.), diacritics preserved, **lyric lines come in shatr pairs (= one bayt); `...` sits only on the 2nd line of each bayt**
  (verified on real v2 files, session 2: 9 of 18 lines, strictly alternating) —
  i.e. it marks the bayt-final / qafiya sustain. Songs = ~9 bayts; Intro = first
  bayt, Outro = last bayt, both repeated from the body; sections hold 2–4 bayts.
- For the pronunciation dataset, deviate deliberately:
  - No `arabmaqamrock` trigger word (meaningless without the style LoRA
    loaded during this training; adds noise).
  - No trailing `...` (would work against "no flavor bleed").
  - One bare `[Verse]` tag only (reuse the existing whitelist token, don't
    invent a new one) — no Intro/Chorus/Outro arc, since a single ayah isn't
    a song.
  - Style/vocals language should be plain and measured ("clear precise
    Arabic diction, deliberate measured pace"; no melisma/vibrato words at all — see below), matching what's
    actually on a Murattal/Muallim tape — caption and audio should reinforce,
    not fight.
- **Real v2 samples received (session 2)**: format confirmed — style paragraph,
  then `[Lyrics]` on the very next line, sections separated by blank lines.
  Line length 3–7 words (mostly 5–6) = one shatr. Draft pron-caption template:
  plain, positive-only paragraph (solo male voice, unaccompanied, clear precise
  Arabic diction, measured pace; deliberately NO mention of melisma/vibrato, either
  way — negations prime the concept; real guard = donor audio + low rank) → `[Lyrics]` → `[Verse]` → one ayah per line
  (or 2 consecutive short ayat = one bayt). No `...` even now that we know it is
  bayt-level: ayat aren't metrical bayts and the sustain cue is the flavor risk.
  User's own lyrics use rare Uthmani patches (U+0671 alef-wasla, U+06E1 sukun in
  one word) — mostly standard orthography, supporting dual-script pairing.

## The big text-orthography idea (potentially the main value driver)

Tanzil's **Uthmani script** already encodes, Quran-wide and for free, the
same conventions the user currently hand-patches word-by-word for MaqamRock
(documented in the user's own "Uthmani orthography fixes" notes: alef-wasla +
shadda for sun-letter ال+ل words like الذي/الليل; hidden-madd spelling for
هذا/ذلك/أولئك/لكن-type words). This reframes the project from "get clean donor
audio" to "get access to a text convention that already systematically solves
a class of pronunciation bugs the user is currently fixing by hand."

**Critical follow-on decision:** pair the *same audio* with **both**
orthographies — Uthmani script and Tanzil's parallel "simple" (imla'i) edition,
both fully diacritized — as two separate training examples per selected ayah.
Reasoning: given the model is demonstrably literal/grapheme-sensitive (see
the `عمرو` observation above), training only on Uthmani spelling risks binding
the correction to "this exact glyph sequence" rather than "this sound" — and
the user's actual singing lyrics are mostly *standard* orthography. Dual-script
pairing forces the generalization that's actually needed, and doubles the
effective dataset size for free. Tanzil publishes both editions ready-made, so
this is a join, not new transcription work.

**Waqf/pause marks — the narrow-strip range (corrected in session 2):** only
`0x06D6–0x06DC` are pause signs (ج، صلى، قلى، م…). The wider `0x06D6–0x06ED`
range in `MARK_CODEPOINTS` is NOT all pause marks — it also holds Uthmani marks
that carry pronunciation (`06E1` Uthmani sukun, `06E2` iqlab meem, `06E5/06E6`
small waw/yeh = hidden madd as in هُۥ, `06E8`, `06ED`, `06DF/06E0` silent-letter
zeros). Dagger alef `0670` (هَٰذَا، لَٰكِن) is likewise protected. Stripping the
whole range would destroy the orthography that is this project's main value.
`0x06DD/06DE/06E9` (ayah-end, hizb, sajdah) are pure markers, opt-in strip.
`06EA–06EC` resolved: each occurs exactly once in the whole Quran (11:41, 12:11,
41:44) and encodes special readings (imala/ishmam) — keep marks, and exclude
those 3 ayat from any donor pool (non-standard pronunciation could bleed).
Helper built: `strip_pause_marks()` in the secondary repo. Also: `06DF` marks
silent letters (≈4000x, Uthmani only) — the exact class of the `عمرو` bug.
Pause signs appear in only 1/60 shortlisted ayat (short ayat rarely carry
them), so waqf handling is nearly moot for the pilot. Late-session reframe worth remembering: these
marks might not be pure noise to discard — their positions may also carry
breath/phrasing information the model could learn from. Undecided, don't
over-index on it, but don't reflexively strip-and-forget either without
considering it.

## Shortlist skew (found session 2) — live watch-item

Density-sorting inherently favors tiny ayat: median 4 words, 47/60 are ≤5 words
(max 11), from 33 surahs. Uniformly short clips is exactly the "wrap-up-early"
risk flagged earlier, now concrete rather than hypothetical. Letter coverage in
the 60: ع 47 ayat, ح 24, ض 21, خ 17 — ع dominates by frequency. Undecided:
mix in longer ayat / consecutive-ayah clips, and whether to rebalance letters.
Real target unit (user, session 2; confirmed by real v2 files): songs are
Mu'allaqat in classical meters (Tawil/Kamil/Basit/Wafir); one lyric line = one
shatr ≈ 4–6 words, bayt = 2 lines ≈ 12 words. So: single ayah
of 4–6 words ≈ one shatr; ≤~12 words or two joined consecutive short ayat ≈ one
bayt. The old 3–12 filter was arbitrary but its upper bound happens to match a
bayt; min should rise to 4. Joining ayat = multi-line `[Verse]`; risks: seams
between per-verse mp3s, and extra recitation cadence (flavor-bleed guardrail).

## Evaluation plan

- Hard-letter substitution tally (ع/أ, ح/خ/ه, ض/ظ) as the pilot's grading
  method: same held-out lyrics, base+style vs. base+style+pron-adapter at a
  few alpha values, same seed.
- The existing 52-sample v2 archive (4 held-out prompts × steps 0-3000,
  in GCS) is available if it becomes useful to check whether pronunciation
  was still improving or already plateauing across steps — not done, not
  currently blocking anything.

## Explicitly open / not yet decided

- **Dataset scale** — user deferred this ("want to think it through first").
  My proposal (small pilot, ~10-20 ayat × 2-3 reciters, scale only if
  non-regressive improvement shows) is a suggestion, not yet agreed.
- **Session 2 result:** next-steps 1–3 done by the agent on secondary-repo branch
  `pron-lora-prep` (commit `aebaf26`, unmerged, 71 tests pass): weighted config,
  60-ayah shortlist, `strip_pause_marks()`, dual-script JSONL (60/60 matched,
  no word-count mismatches). Still nothing built for audio/dataset/training.
- **Session 2, second half:** resolved shortlist-length skew using real meter
  info from the user (Mu'allaqat: shatr ≈ 4-6 words, bayt ≈ 12 words) and real
  v2 caption samples the user provided. Locked the pron-caption template as
  positive-only (no melisma/vibrato mention at all, even negated — user's call,
  confirmed better than my own draft: negation primes the concept in
  gen-music models). Task 2 prompt was fully drafted and agreed but not sent
  by session end.
- **Session 3 result:** Task 2 sent and completed by the agent, same branch
  (commit `e779626`, 112 tests pass, up from 71). Verified directly (agent's
  written report didn't come through with the push, same as session 2):
  `pron_shortlist_v2.csv` — 40 ayat, 25 type A (4-6w) / 15 type B (7-12w)
  exactly as speced, all four letters ≥16 ayat (no quota rebuild needed), the
  3 excluded special-reading ayat absent. Dual-script JSONL: 40/40 matched, no
  count mismatches. Caption `.txt` pairs (80 files): exact layout verified
  by `tests/test_captions.py`, which also asserts no protected codepoint
  (session-2-corrected set, `aya_scoring/arabic.py`) was lost vs raw text —
  good, this is a real regression check, not just a smoke test.
  `reciter_audit.json`: `warsh/` and both Mujawwad dirs excluded, al-Ajamy
  duplicate correctly flagged not double-counted, 40-key coverage computed
  per reciter.
  **Gap found, not yet resolved:** the agent's own script docstring says
  `_catalog.json`/`_metadata`/`_registry` were checked and "carry riwayah,
  verse counts and provenance but no performance-style field" — so
  classification fell back to directory-name matching for all 32 dirs, and
  27 of them ended up `unknown` (only the ones with Muallim/Murattal/Mujawwad
  literally in the name got classified). This may be a correct finding
  (the catalog genuinely may lack a style field) or the agent may not have
  looked hard enough — I can't verify bucket contents from this container
  (no GCS network access here). Worth a direct question to the agent before
  picking reciters from the `unknown` bucket.
  **Also not yet confirmed:** Part C step 8 (download 6 shortlisted mp3s
  across 2 full-coverage reciters, report duration/sample-rate/channels) —
  no output for this was committed (audio isn't meant to be committed, so
  that's expected) and no report text came through either. Need to ask the
  user/agent for those 6 numbers directly.

## Working-mode note: delegate token-heavy work to the user's AI agent

This is a WebUI session — context window is a real bottleneck, and the user
has a separate AI agent available (with GPU/compute access, real-time
logging, and a coding environment) that they will relay messages to/from.

**Default to writing a prompt/spec for that agent instead of doing the work
here myself** whenever a task is token-heavy or compute-heavy: large CSVs or
text corpora, log analysis, anything needing a GPU, real-time monitoring, or
actual training/inference runs. **Coding, specifically: write the spec, let
the agent write the code, then verify its output — don't jump into writing
implementation code directly in this session.**

Use judgment, don't over-apply this: small, token-light things (a short bash
listing, reading one config file, a quick calculation) are fine to just do
here directly — routing those to the agent would be needless overhead for
both me and the user. The line is roughly: would this burn a meaningful chunk
of this session's context for something that doesn't need *this* session's
reasoning to produce? If yes, write the agent a prompt instead.

## Natural next steps (whenever resumed)

1. Resolve the two session-3 gaps above: (a) ask the agent to confirm/dig
   deeper on whether the catalog really has no performance-style field before
   trusting the `unknown` classifications, (b) get the 6 duration/sample-rate
   numbers from Part C step 8.
2. Pick final reciters (2-3, not 1, per the earlier generalization decision)
   from the full-coverage list and download the 40-ayah audio sets.
3. Build the merge tool with the alpha=0 bit-for-bit invariant test.
4. Dataset scale is still undecided (user deferred it, session 1) — revisit
   before assembling the actual training set.
5. Only then: dataset assembly, small low-rank AR-only LoRA run, alpha sweep,
   letter-substitution scorecard.

## Maintaining this file

Update it proactively (no need to be asked) when a real *insight* lands — a
decision, a rejected idea and why, a corrected assumption, a guardrail. Not
details: if it's greppable in a terminal, it doesn't belong here; if it took a
dedicated script/agent exploration to learn, it does. Keep entries terse and
correct stale claims in place rather than appending contradictions.
