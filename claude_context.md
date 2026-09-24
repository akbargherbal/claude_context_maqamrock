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
- **Branches (session 8):** secondary repo work is on `pron-lora-prep`; main repo pron-LoRA work
  (configs, runbook, setup.sh/backup changes, later the merge tool) is on **`pron-lora-ar-only`**
  (off `main`; never commit pron work to `main`). Prompt history: `session_logs/SESSION_08.md`
  holds the exact Task 13/14/14b/14c prompts (sent + queued).

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
- **Final reciter set — 9, user's own pick (session 5), committed directly to
  `reciters_shortlist/shortlist.txt` on the secondary repo's `main` branch**:
  `Husary_128kbps` (note: standard Husary, **not** the earlier-flagged
  `Husary_Muallim`), `Abdul_Basit_Murattal_192kbps`,
  `Abu_Bakr_Ash-Shaatree_128kbps`, `Minshawy_Murattal_128kbps`,
  `Hudhaify_128kbps`, `Muhammad_Ayyoub_128kbps`, `Yaser_Salamah_128kbps`,
  `aziz_alili_128kbps`, `Abdullah_Basfar_192kbps`. Verified directly against
  `reciter_audit.json`: all Murattal, none excluded, none duplicates, all had
  full coverage of the (then-)40-ayah shortlist. Caveat worth keeping: 7/9 are
  classified via the common-knowledge fallback, not a hard bucket field — only
  Abdul_Basit_Murattal and Minshawy_Murattal matched on name-keyword. Going to
  9 (not the originally-discussed 2-3) is a deliberate go-wide choice — a
  different bet than the original "avoid single-voice memorization" reasoning,
  trading more download/storage for more voice diversity.
- **Real pacing data (session 5, agent Task 4, commit `369a299`, secondary
  repo)**: ffprobe'd a fixed random sample (seed=42; 3 type-A + 3 type-B ayat
  × all 9 reciters = 54 files; audio deleted after probing, only JSON numbers
  committed — verified directly, no mp3s in the diff). Pooled: type A mean
  9.81s, type B mean 12.27s, overall 11.04s. `Husary_128kbps` is a real pacing
  outlier — ~15-17s/ayah, 40-75% slower than the other 8 reciters (9.0-11.4s
  band). Confirms the session-4 teaching-style-vs-standard pacing gap, now
  measured on the actual 9 picked reciters.
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

## Control knobs for pronunciation strength (post-training, session 9)

Revisited the original scope line ("only crisp pronunciation, nothing else — no
recitation flavor") now that the real run finished, to lay out what will
actually be tunable. Three things, not one:

1. **Alpha — the primary, purpose-built dial.** `W = W_base + 1.0·dW_style +
   alpha·dW_pron`, continuous, includes `alpha = 0` (must reproduce v2
   bit-for-bit). This is what the two-LoRA split was designed to give us.
2. **Checkpoint choice — a second, independent dial, only visible now that
   training telemetry exists.** 4 checkpoints (steps 1525/3050/4575/6100) +
   final. `loss/ar_ce` (pronunciation signal) improved steeply early then
   flattened after ~step 1500; `loss/ar_kl` (drift from base AR behavior) rose
   the *entire* run, never plateauing (session 9 result, above). So an earlier
   checkpoint may give most of the pronunciation gain with less accumulated
   drift than the final one — it's really **checkpoint × alpha**, a 2D space,
   not alpha alone.
3. **A structural guardrail, not a runtime knob, but the reason the design is
   safe at all.** The pron LoRA is AR-only by construction
   (`ignore_if_contains: ["transformer.nar"]`) — the NAR/flow path, where
   musical style/timbre/melisma/cadence actually live, is architecturally
   untouched at *any* alpha value. This is why alpha can be pushed toward 1.0
   without the merge risking style bleed: the adapter never had access to
   that part of the model. Caveat: this guarantees the *style/timbre* path is
   safe, not that stronger AR conditioning can't carry audible
   cadence/delivery artifacts of its own — which is exactly why the
   letter-substitution scorecard + an ear-check for recitation-cadence bleed
   stay in the eval plan, not skippable.

**Open question this creates, to settle before/while writing Task 15:**
whether the merge tool makes alpha a *live* multiply-at-load-time scalar, or
whether — because v2 (rank 32) and pron (rank 8) can't have their raw A/B
pairs summed — it's forced into rank-concatenation, meaning **each alpha
value requires baking a separate merged file**, not a cheap runtime slider.
This is design question (2) from `session_logs/SESSION_08.md` (line ~195),
now sharper: it decides whether sweeping 4 alpha values × up to 5 checkpoints
is ~20 cheap inference runs or ~20 separate merge+convert jobs. Resolve by
reading the actual merge code path and how v2's LoRA currently reaches
`audio.cpp` inference (`docs/INFERENCE.md`,
`/content/converter/out/convert_aitoolkit_yue2_lora.py`) before writing Task 15.

## Explicitly open / not yet decided

- ~~**Dataset scale**~~ — resolved session 5: **350 ayat, dual-script, 9
  reciters.** See "Session 5 result" below. (My original ~10-20-ayat pilot
  proposal was superseded, not followed — worth remembering next time a
  "small pilot" instinct gets proposed here without re-checking it against
  what's actually cheap to get.)
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
  **Gap found (resolved session 4):** the agent's own script docstring said
  `_catalog.json`/`_metadata`/`_registry` were checked and "carry riwayah,
  verse counts and provenance but no performance-style field" — so
  classification fell back to directory-name matching for all 32 dirs, and
  27 of them ended up `unknown`. See "Session 4 result" below for the
  confirmed resolution.
  **Also not yet confirmed (resolved session 4):** Part C step 8 (download 6
  shortlisted mp3s across 2 full-coverage reciters, report duration/sample-
  rate/channels) — no output for this was committed and no report text came
  through either. See "Session 4 result" below.
- **Session 4 result:** verified session-3's two gaps directly from git
  history (agent's chat reports keep not surviving the push — verify from
  commits, not prose, every time). Two commits on the same branch:
  - `816628b` — Gap 1, resolved cleanly. Confirmed directly (now recorded in
    both a code comment and a new `style_metadata_note` key inside
    `reciter_audit.json` itself, not just a docstring): `_catalog.json`,
    `_registry/registry.json`, `_registry/registry_history/*.json`, and all
    31 `_metadata/*/reciter.json` files genuinely carry no style/type/
    category field, and every `note` field is empty — the catalog really
    doesn't have this data, the agent hadn't missed it. Added a
    `classification_source` field (`name-keyword`/`common-knowledge`/`none`)
    and a curated 26-name `KNOWN_MURATTAL` fallback map, name-match still
    taking precedence. All 32 dirs now classified except `warsh` (stays
    `unknown`, already excluded anyway). Sudais/Shuraim/Hudhaify/Ayyoub/
    Basfar — the names flagged for sanity-checking — all landed Murattal as
    expected. Caveat worth keeping in mind: the common-knowledge fallback is
    the agent's own general knowledge asserting these 26 are Murattal, same
    epistemic weight as our own naming sanity-check, not an independent data
    source — reasonable and well-documented, but not a hard fact pulled from
    the bucket. 112 tests still pass (metadata-only change).
  - `852c6fe` — Gap 2, resolved after a nudge (pushed shortly after being
    asked about; wasn't in the first commit at all — not a "report lost on
    push" case like sessions 2-3, this time the work itself came later).
    New `pron_ffprobe_scratch.py` + committed `ffprobe_scratch.json` (numbers
    only, verified no mp3s in the diff). 3 keys from `pron_shortlist_v2.csv`
    (`068030`, `026148` type A; `023041` type B — confirmed against the CSV
    directly) × 2 full-coverage reciters (`Husary_Muallim_128kbps`,
    `Minshawy_Murattal_128kbps`). All 6 files: 44100Hz, stereo, mp3, 128kbps.
    Durations: Husary_Muallim 068030=12.83s, 026148=10.71s, 023041=24.19s;
    Minshawy_Murattal 068030=7.22s, 026148=6.04s, 023041=14.35s. Notable for
    later: same ayah runs ~2x longer under Husary_Muallim than
    Minshawy_Murattal (12.8s vs 7.2s on 068030) — a real pacing difference
    between "measured teaching" and "standard Murattal" style, worth
    remembering if cross-reciter pacing consistency ever matters.
  **Working-mode lesson reinforced:** this repo's agent's plain-text chat
  reports have now failed to survive 3 sessions running. Stop asking for
  numbers "in the reply" — the prompt template should require committing any
  requested numbers/analysis into a small file, the way Gap 2's fix finally
  did, even for read-only/analysis asks.
  **Session 4 end (cut off by max-token, not recorded until session 5):**
  user declined to commit to reciters yet ("want to think about reciter
  choice more"). Pivoted to dataset scale (open since session 1) — reframed
  it as two separate smaller levers (reciter count; whether to run a
  throwaway pipeline-shakedown slice before the full run) rather than one big
  "how much" question, since the 40-ayah shortlist itself was sunk cost
  either way. Session ended on an unanswered clarifying question about what
  was actually driving the scale hesitation. See "Session 5 result" for how
  this actually resolved (differently — user reopened the 40-ayat number
  itself, which the session-4 framing had assumed was fixed).
- **Session 5 result:** picked up the dangling thread directly. Two real
  decisions landed:
  1. **Reciters: 9**, user's own pick, done outside any agent task — see the
     new entry under "Data/methodology decisions" above for the list and
     verification. Session 4's "2-3 reciters" framing didn't hold; the user
     went wider instead of narrower.
  2. **Ayat count: 350** (up from 40), full dual-script pairing kept. This
     reopened an assumption session 4 had treated as settled — the 40-ayah
     count was never a principled limit, just leftover from an early
     "small pilot" instinct (`pron_shortlist_v2.py`'s `--type-a-count
     25 --type-b-count 15` are CLI defaults, not a hard cap) plus the
     letter-coverage floor mechanics. The actual scored pool is 2,911 ayat
     (1,184 type-A / 1,727 type-B candidates in the 4-12 word band), so 350
     is well inside it. **Verified complete, session 5 (Task 5, agent commit
     `8cac7a6`)**: rebuilt for real at `--type-a-count 219 --type-b-count 131`
     — same numbers as the trial run. `method: density-within-type`, no
     coverage-floor fallback. Letter coverage: ح 166, خ 113, ع 277, ض 86, all
     comfortably above the floor of 12. **Step 2's re-check of reciter
     coverage against the NEW 350-key shortlist came back clean** — all 9
     reciters, 350/350, 0 missing (this was flagged explicitly as unverified
     at session-4-era assumptions and is now confirmed, not assumed). Dual-
     script join: 350/350 matched, 3 minor word-count mismatches between
     editions (020083 u5/s6, 072016 u7/s8, 012039 u9/s10 — expected
     occasional orthographic variance, not an error). 700 caption files.
     Test suite: 422 passed (up from 112).
  Real measured total (not the estimate): **350 ayat × 9 reciters = 8.31
  hours of donor audio** (29,904.8s), **6,300 training pairs** after dual-
  script join. Agent commit `8c8c9a1`: 3,150/3,150 mp3s downloaded, 0
  missing, 0 corrupt, full-corpus ffprobe (not a sample) committed to
  `data/pron/donor_audio_manifest.json`. Audio itself lives at
  `data/pron/donor_audio/<reciter>/<key>.mp3`, gitignored, NOT committed —
  verified directly, no mp3s in either commit's diff. **One real anomaly
  found, not yet resolved:** 37/3,150 files (~1.2%) came back at unexpected
  format vs. their directory-implied bitrate — most are harmless (mislabeled
  bitrate, still full quality), but **14 files in
  `Abu_Bakr_Ash-Shaatree_128kbps` are genuinely lower quality: 11025Hz,
  mono, 24kbps**, not just a labeling mismatch. Exact keys are in
  `unexpected_format_files` inside `donor_audio_manifest.json`. Open
  decision for next session: keep these 14 as-is, or exclude them (and
  decide whether that leaves `Abu_Bakr_Ash-Shaatree` materially short of
  its 350, or is a rounding error against 3,150 total).
  **Also found this session:** the secondary repo carries its own
  `PROGRESS_pron_lora_prep.md` (agent-written, commit `15e3559`, on
  `pron-lora-prep`) — a reproduce-from-scratch doc with exact commands and
  file locations. Useful as a second reference alongside this file; skim it
  too when resuming repo-mechanics questions this file doesn't cover in
  command-level detail.
  **Tanzil script-variety question (asked and resolved, same session):** all
  6 Tanzil script variants (uthmani, uthmani-min, simple, simple-plain,
  simple-min, simple-clean) were already downloaded back at corpus setup
  (`download_corpus.py`, commit `a17f833`) — confirmed download params match
  what the user later screenshotted from tanzil.net exactly: `marks=true`
  (pause marks), `sajdah=true`, `tatweel=true`, no `rub` param. But the
  caption pipeline (`pron_dualscript.py`) only ever consumed 2 of the 6:
  full `uthmani` + full `simple`. **Decided to keep it at 2, not expand** —
  the other 4 are all diacritic-reduced variants of one or the other
  (`simple-clean` is skeleton-only, no vowels at all), and diacritics are
  exactly the signal a pronunciation adapter needs, so adding them would
  work against the goal, not add useful variety. Don't re-raise this as an
  open gap unless the reasoning above changes.
  **Task 5 (dataset build) fully verified complete, session 5** — see the
  reciter/ayat-count entries above (agent commits `8cac7a6`, `8c8c9a1`) for
  the full numbers. The 350-ayah training dataset (text + captions + donor
  audio) now exists and is verified; the one open item it left behind is the
  14-file low-quality anomaly noted above.
  **Framed by the user at session-5 close, for next session:** the task
  isn't just "resolve the 14 flagged files" — do a broader low-quality
  gatekeeping pass across the corpus. The 37/3,150 format anomalies were
  only caught because they were *labeled* wrong (bitrate/sample-rate
  mismatch); there could be quality issues ffprobe's format fields wouldn't
  catch at all (clipping, background noise, silence padding, truncated
  audio). Treat the 14 as the known floor of the problem, not the whole of
  it — plan an actual filtering pass, not just a patch on the flagged set.
- **Session 6 result (in progress, gatekeeping pass — Tasks 6-8, secondary
  repo, `pron-lora-prep`, commits `df963c7`, `fa56df8`, `dcec1a5`):**
  - **Data integrity independently verified, not just claimed**: local MD5 of
    all 3,150 donor mp3s checked against GCS's own object metadata MD5 (no
    re-download needed) — 3150/3150 match, checked twice (before and after
    the analysis steps). Donor audio is confirmed untouched throughout the
    whole gatekeeping pass. Worth repeating this pattern any time a task
    claims "nothing was modified" on ungit-tracked data — verify against the
    source, don't take the self-report on faith (this came from the user
    pushing back on exactly that).
  - **Bandwidth/rolloff check retired as a criterion (Task 6→7)**: spectral
    rolloff for this content sits ~2.5-9.5 kHz regardless of true recording
    quality (quiet unaccompanied recitation just doesn't carry much energy
    higher than that), so it can't separate the known-bad 11025Hz files from
    clean 44100Hz ones — hundreds of legitimately fine files score as "low
    bandwidth" too. ffprobe's `sample_rate` field remains the correct signal
    for under-sampling; content-based rolloff isn't a useful proxy for it
    here. Raw per-file values kept in the report, not treated as actionable.
  - **2 files are genuinely corrupt, confirmed at the source**:
    `Hudhaify_128kbps/023005`, `aziz_alili_128kbps/086008` fail libsndfile
    decode; fresh re-download from the GCS bucket came back byte-identical
    (same MD5) and still fails — the corruption is in the bucket object
    itself, not a bad local copy, so re-downloading again won't fix it.
    Recorded in `data/pron/training_pair_exclusions.json` (drops just those
    2 (reciter, ayah) pairs, not the ayat themselves — the other 8 reciters'
    clips for those same ayat are untouched).
  - **Policy decided, locked in: flag audio quality issues for a human
    decision, never auto-edit.** Task 7 built a non-destructive silence-trim
    manifest (offsets only) and demoed actual trims in a throwaway QA zip to
    prove it was non-destructive. The user reviewed the before/after samples
    and rejected the whole trim-then-apply approach on principle, even
    though the actual trim was tiny (290/3,150 files, 203.6s total off
    29,904.8s = 0.68%) — reasoning: this project's own stated guardrail is
    crisp articulation, not performance, and the model is independently
    known to be literal/grapheme-sensitive (see `عمرو` note above), so
    editing donor audio adds a real risk of introducing something unintended
    for very little upside. `silence_trim_manifest.json` stays in git as a
    record but is retired — never applied downstream. **Don't re-propose
    audio editing (trimming, gain normalization, or otherwise) in a future
    session without re-litigating this decision first.**
  - **Reworked silence check (Task 8, replaces Task 6's 20%-of-frames rule)**:
    single unified metric — longest continuous silent run anywhere in the
    file (start/middle/end, position reported), flagged if >=3.0s. Much
    tighter than the old rule: 19/3,150 flagged (was 113), 17 of those in
    `Husary_128kbps` (consistent with its already-known slow/teaching pace),
    worst case 5.78s. Short, clean, explainable list — treated as a real
    exclude-candidate list, not yet acted on (next step: zip the 19
    originals, untouched, for the user to listen to before deciding).
  - **New loudness-outlier check (Task 8, `pyloudnorm`, LUFS/BS.1770)**:
    flag if a file's integrated loudness deviates >3dB from that *same
    reciter's own* median (not a cross-reciter comparison — different
    reciters are expected to differ). 312/3,150 flagged, spread across all 9
    reciters with no single dominant outlier (unlike silence) — per-reciter
    std is 0.93-2.51dB against a 3dB threshold, a fairly loose net. Current
    read: this is probably mostly normal verse-to-verse vocal-dynamics
    variation (a reciter reciting some ayat with more emphasis/volume),
    *not* a quality defect the way the silence and corrupt-file findings
    are — don't treat this list as an exclusion candidate without more
    scrutiny. Side-finding, informational only, not flagged: reciters' own
    median LUFS spans a 7.4dB range (`aziz_alili` -13.8, loudest;
    `Abdul_Basit` -21.2, quietest) — expected from different recording
    setups, may matter later if loudness-matching ever becomes relevant at
    actual clip-assembly time, not now.
  - **Dataset-scale/duration-floor thread (raised by user, not yet a
    decision)**: user proposed a ≥8-hour floor for the corpus (current raw
    total: 8.31 hrs) and suggested any quality exclusions should be
    compensated by adding more ayat so the corpus doesn't shrink. Pushed
    back gently: (a) real exclusion counts so far are tiny relative to
    8.31 hrs (even the corrupt+silence lists combined are ~21 files, seconds
    not hours), so backfilling preemptively means guessing at a number for a
    problem that hasn't materialized; (b) excluding a bad file is a
    (reciter, ayah)-pair-level decision, not an ayah-level one — dropping
    one reciter's take for one ayah (cheap, doesn't touch letter coverage)
    is a different, cheaper lever than dropping the whole ayah to preserve
    uniform 9-reciter coverage. Agreed to revisit "add more ayat" only if a
    future finding turns out concentrated/large (e.g. a whole reciter
    turning out systematically bad), not for the current scattered findings.
  - **Still open, not yet decided**: what to do with the 19 silence-flagged
    files and the 312 loudness-flagged files; whether/how to fold
    `training_pair_exclusions.json` into whatever eventually assembles real
    training pairs. **Resolution mechanism decided at session close: human
    listening, not further automated analysis.** Task 9 sent (confirmed complete in
    session 7, commit `6a20446`) asking the agent to zip,
    at `/content/human_review.zip`, the original/untouched audio for: all 19
    `silence_long_gap` files, plus a seed=42 proportional sample (~30 files,
    capped ~4/reciter) of the 312 `loudness_outlier` files, each with a
    `MANIFEST.txt` line stating exactly what to listen for (gap
    duration+position, or dB deviation from that reciter's own median) so
    the reviewer doesn't have to cross-reference the JSON by hand. **Next
    session starts with the user relaying their listening results** — treat
    that as the actual decision input for exclude/keep on both lists, not
    something to re-derive from the numbers alone.
  - **Also touched this session (read-only)**: cloned
    `maqamrock-yue2-lora-finetuning` (main repo, still untouched — only
    `main` branch, nothing built yet) and skimmed
    `docs/FUTURE_PRONUNCIATION_LORA.md` ahead of the merge-tool task. It
    already documents the merge mechanism (`ai-toolkit`'s
    `toolkit/lora_special.py`: `merged = base_weight + merge_weight*delta`,
    a literal linear sum) and how to scope an adapter to AR-only
    (`network_kwargs.ignore_if_contains: ["transformer.nar"]`) — both
    directly relevant when the merge-tool task actually starts. User's own
    sequencing call: do the merge tool *after* the dataset is fully settled,
    on a new branch (not `main`) on that repo, same pattern as the secondary
    repo's `pron-lora-prep`.

- **Session 7 result (gatekeeping CLOSED, dataset final):**
  - Task 9 confirmed complete (agent commit `6a20446`; the commit message
    carried the full report this time). `/content/human_review.zip` was built
    (49 audio + MANIFEST, all byte-identical to source) and lives on the
    agent's box, not in git.
  - **User listened; decisions final:** loudness-flagged files (312) all
    acceptable, **keep all**. Silence-flagged files (19) **keep all** (no
    editing/trimming, per the session-6 policy). The 2 corrupt files
    (`Hudhaify_128kbps/023005`, `aziz_alili_128kbps/086008`) play fine in a
    normal media player, but they still fail libsndfile. Initially left
    excluded, later reversed (see below). **Task 10 decode check was then run after all** (agent commit `3f493ff`,
    `data/pron/corrupt_file_decode_check.json`): both files decode fully and
    identically under ffmpeg, torchaudio and librosa (same frame counts, within
    0.14% of ffprobe duration); **only libsndfile fails**. MD5s unchanged.
    The training loader is documented (main repo `verification.md:22,27`) as
    using torchaudio; caveat: ai-toolkit's loader code isn't in either repo, so
    that is from docs, not a verified line. **User called the exclusion a false
    positive; reinstated (Task 11, agent commit `1773dda`, verified from the
    diff):** `training_pair_exclusions.json` is now `[]` (file kept), both mp3s
    present, MD5s match, torchaudio.load ok. 445 tests pass. Lesson: a libsndfile-only failure is not evidence of a bad file; check
    the decoder the training loader actually uses before excluding anything.
  - **The 14 low-format Abu_Bakr files (open since session 5) resolved,
    Task 12, agent commit `fd0d118`:** 11025 Hz / mono / 24 kbps (keys 010014,
    010058, 010082, 017021, 017022, 017049, 017075, 017105, 017106, 017109,
    041002, 041003, 041013, 041035). User's rule: verify them, and disqualify
    if verified low quality. Agent compared each against Abu_Bakr's own 336
    normal files (torchaudio decode, no audio modified, 14/14 MD5 match) on
    4-5.5 kHz HF energy ratio and a 2-5 kHz consonant proxy, with a rule
    fixed in advance (no threshold tuning). **10 DISQUALIFIED, 4 KEPT, 0
    inconclusive**; the 10 went into `training_pair_exclusions.json`
    (pair-level, Abu_Bakr only). Kept: 017105, 041003, 041013, 041035 (inside
    the reference 5th-95th on both consonant-relevant metrics despite the low
    bitrate). Honest caveats, verified from the JSON: the bandwidth-cutoff
    metric is trivially "below p05" for every 11 kHz file, so the rule in
    effect reduced to "HF ratio or consonant proxy below p05"; `010082` is
    marginal (HF -23.8 dB vs the reference p05 of -23.76 dB, consonant proxy at
    p50), and `041002` was driven by HF alone. Cost is 1 pair either way, so
    left as decided rather than re-litigated. 452 tests pass.
  - **Final dataset state (after Tasks 11-12): 3,140 files (Abu_Bakr 340
    ayat, other 8 reciters 350) = 6,280 dual-script pairs**, 10 pair-level
    exclusions, all Abu_Bakr. Each excluded take drops both script variants.
  - **Assembly design agreed in principle, not yet built** (the two starred proposals were confirmed in session 8): flat folder of `.mp3` + same-stem
    `.txt` (`caption_ext: txt`), one stem per `<reciter>_<key>_<script>`, audio
    copied not symlinked; pron config must blank `trigger_word`
    (v2's `arabmaqamrock` would otherwise be prepended to every caption) and set
    `network_kwargs.ignore_if_contains: ["transformer.nar"]` with rank well
    below v2's 32; steps set by epochs, not copied from v2's 3000 (~0.5 epoch
    on 6,280 pairs)*; hold out ~10 ayat across all reciters/scripts for
    validation loss, split at ayah level*. Assembly script goes in the
    secondary repo (output gitignored); config goes on a new main-repo branch
    with the merge tool.
  - **Guardrail restated by the user twice this session (same one as the
    goal section):** this is crisp-pronunciation training, not performance
    training. Corollary for data quality: the only valid exclusion criteria
    are defects that damage the consonant signal (corrupt/truncated audio,
    audio/text mismatch). Loudness dynamics and pause length are performance
    traits and are **not** exclusion grounds. Don't propose new quality
    checks that measure performance.
  - **Text/audio match check declined by the user:** proposed a duration-per-
    word outlier check (catching basmala prepended on ayah-1 clips,
    truncation), and the user said no. Their own random sampling of ayat
    never turned up wrong text, and that is enough for them. Don't re-raise.

- **Session 8 result (dataset assembled, pron LoRA config + persistence done; smoke test pending):**
  - **Confirmed by the user:** epoch-based step count; ayah-level ~10-ayat hold-out. Claude's three
    further proposals were not objected to and are baked into the configs (user's call to change):
    **rank 8** (linear/alpha 8/8), **1 epoch first with quarter-epoch checkpoints** (so the alpha
    sweep doubles as the stopping rule), **in-training sampling off** (eval = merged alpha sweep).
    Note: 1 epoch = 18 exposures per ayah text (9 reciters x 2 scripts); not a "light" run.
  - **Task 13 done, verified (secondary repo, commit `df3193d`):** 6,100 train / 180 val / 16 smoke
    pairs. Hold-out (seed 42): type A 007078, 020005, 030017, 037061, 069046, 079025; type B 006042,
    007052, 012039, 043067; covers all four letters (ض in only 2 held-out ayat, so val is weak for ض;
    the real ض eval is the held-out-lyrics sweep). Unique train audio = 8.005 h (barely clears the
    user's session-6 >=8 h floor); val 0.26 h. Claude re-verified counts, exclusions, hold-out
    constraints and manifest MD5s vs the GCS-verified MD5s from the committed JSON; the folders
    themselves (txt byte-identity, forbidden-substring scan) are agent-reported only.
    Folders live at `<secondary>/data/pron/training_set/{train,val,smoke}` (gitignored), stems
    `<reciter>_<key>_<script>`; caption text depends on (key, script) only, not reciter.
  - **Task 14 CPU phase done (main repo, `pron-lora-ar-only`, commit `0220aa8`), verified from the
    diff:** `config/pron_lora_ar_only.yml` = v2 config with only: name/log_dir `pron_lora_ar_only_r8`,
    `trigger_word: ""`, rank 8/8, `ignore_if_contains: ["transformer.nar"]`, folder
    `/content/pron_dataset/train`, steps 6100, `save_every: 1525` (checkpoints 1525/3050/4575/6100 +
    final), `train.disable_sampling: true`, sample prompts removed. `..._smoke.yml`: folder
    `.../smoke`, 10 steps. Also: opt-in `job_pron_dataset()` in `bootstrap/setup.sh`; a real latent
    bug fixed in `backup_to_gcp.py` (`--run-name` used to mirror v2's local output folder under any
    run's prefix); runbook `docs/PRON_LORA.md`; source-cited `docs/PRON_LORA_VERIFICATION.md`.
  - **Source findings worth keeping (agent-cited, Claude spot-checked in ai-toolkit):**
    - **Saved key names differ from training names:** on save, AR keys `transformer.ar.*` become
      **`text_encoders.*`** and NAR keys `transformer.nar.*` become **`diffusion_model.*`**
      (`yue2_model.py convert_lora_weights_before_save`). This corrects
      `docs/FUTURE_PRONUNCIATION_LORA.md` ("saved keys are transformer.ar.*") and the session-6 note;
      the smoke pass criterion is `text_encoders.*` > 0, `diffusion_model.*` == 0, rank 8 everywhere.
    - `trigger_word: ""` prepends nothing (`prompt_utils.py` guard `trigger.strip() != ""`).
    - `validation_config` is image-only: **no audio validation loss in ai-toolkit**. Offline val-loss
      script proposed, not built.
    - `ar_kl` = KL(base || lora) on AR tokens (base = network off); the flow loss sees a detached AR
      KV cache and NAR is frozen, so the AR LoRA trains only on ar_ce + ar_kl; NAR forward still runs
      every step (wasted compute). `content_or_style` is inert here. `conv`/`conv_alpha` have no
      effect (yue2 targets are Linear). `steps` counts loop iterations (= optimizer steps at
      batch 1 x accum 1). Latent cache is per dataset folder (`<folder>/_latent_cache`), no collision
      with v2. `disable_sampling` lives under `train:`, not `sample:` (Claude's spec was wrong).
    - `setup.sh` training mode **hard-requires `GCP_DATASET_PATH`** (v2 dataset) and always runs
      `job_dataset`, so any GPU VM also pulls the v2 dataset in the background (harmless; left as is).
      Agent's report skipped this (A7); Claude answered it from the script.
  - **GCS layout:** v2 dataset `gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/dataset/`
    (= `/content/yue2_dataset`); pron dataset is a SIBLING prefix `.../pron_dataset/{train,val,smoke}`
    (never inside `dataset/`, which `job_dataset` rsyncs wholesale); run backups
    `<base>/<run-name>/output/`, base inferred = `.../OSTRIS_Arabic_Suno_Finetuning`. **Upload done by
    the user, counts verified: 12,200 / 360 / 32.**
  - **Runtime plan (user's call, CU-driven; Claude had recommended one A100 VM, roughly break-even):**
    smoke test on **L4** (1.52 CU/h), then real run on **A100** (6.77 CU/h). Smoke is a pure
    preflight (own run name, nothing reused). Each switch wipes `/content`, so the A100 VM needs its
    own full `setup.sh` + dataset restore. L4 smoke step-time/VRAM do NOT transfer to the A100 ETA
    (v1/v2 whole-song: ~13.4 s/step L4 vs ~3.10 A100, `docs/GPU_L4_VS_A100.md`; short clips unmeasured).
  - **Agent-reporting pattern held again:** the commit message carried the report, but omitted A7, the
    upload dry-run evidence, and the fresh-VM `git checkout` step (runbook gap, patched in Task 14b).
    Verify from the diff, as always.
  - **Where session 8 stopped:** user starting the L4 VM to run Task 14b (L4 version, in
    `session_logs/SESSION_08.md`); on PASS, switch to A100 and send Task 14c (same file). Task 15
    (merge tool) not yet written; open design questions are listed at the end of that log
    (rank 32 vs 8 cannot be summed as raw A/B; base is int8-quantized, so a LoRA-file-level merge is
    likely; define "bit-for-bit" precisely; check how v2 reaches audio.cpp inference first).
- **Session 9 result (L4 smoke PASSED, A100 real run completed):**
  - Task 14b (L4): PASS. `text_encoders.*` 224 keys, `diffusion_model.*` 0, all ranks 8, all losses
    finite. Commit `258a098`. One preflight catch worth remembering: `GCP_DATASET_PATH` and
    `GCP_PRON_DATASET_PATH` got crossed on the VM (pron data landed under `/content/yue2_dataset`
    instead of `/content/pron_dataset`); agent caught it before training, re-pulled correctly. Cause
    was a user env-export mixup, not a script bug — **re-check both exports on every fresh VM**.
  - Task 14c (A100): preflight PASS (`1100c84`), then 6,100-step real run completed cleanly. No
    tracebacks, no OOM, VRAM flat ~10.5 GB. `loss/ar_ce` (the only term this AR-only adapter
    actually optimizes) fell steadily 5.20→4.40 (500-step window means); `loss/ar_kl` rose
    monotonically but bounded (max 2.19), same shape as v1/v2 — not a blow-up, but the lever to pull
    (`ar_kl_weight` / fewer steps / lower LR) if offline eval later shows AR over-drift. Diminishing
    returns after step ~1500 (per-500-step gains dropped from −0.41 to low single digits). 4
    checkpoints (1525/3050/4575/6100) + 1 final file, all local + GCS. Analysis:
    `TRAINING_ANALYSIS/pron_lora_ar_only_r8/ANALYSIS.md`.
  - **Real bug found mid-run, fixed live:** `backup_to_gcp.py`'s `wait_for_settle` never excluded the
    TensorBoard event file, which is rewritten every step (`log_every: 1`) — so the settle check
    never completed and NO checkpoint reached GCS for ~1 h, even though logs/agent_notes looked
    synced (false confidence). Fixed by adding `tensorboard/` and `events.out.tfevents*` to
    `SETTLE_IGNORE_NAMES` (commit `5d76d89`). **Generalizable gotcha:** any settle/sync watcher on a
    training output folder must exclude continuously-rewritten log files, or it will starve on them.
  - **A100 was overkill for this job.** Median GPU util 22%, VRAM only 10.5/80 GB, step time 0.886 s
    — barely faster than the L4 smoke's ~1.0 s/step (itself warmup-inflated). This run is
    data/CPU-bound (short-clip audio/VAE path), not compute-bound. **Guardrail for future short-clip
    jobs on this dataset shape: default to L4, not A100** — the A100's throughput advantage only
    showed up for v1/v2's whole-song clips (`docs/GPU_L4_VS_A100.md`), not these short recitation
    pairs. Don't assume that number transfers to a different clip length/dataset without remeasuring.
  - **Next:** offline AR-loss replay over the 180 `val` pairs per checkpoint (`PRON_LORA_VERIFICATION.md`
    A3) — loss curves alone don't tell us if pronunciation improved (`disable_sampling: true`, no
    validation split during training). Then Task 15 (merge tool, open questions still unresolved,
    see session 8 bullet above).

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

**User preferences learned session 8 (follow these):** (1) Hand over prompts as ONE complete,
copy-pasteable block with no placeholders or "replace step X" edits. (2) Give an exact, ordered
to-do (what to do now, when to switch runtimes, when to come back) rather than options; the user
dislikes back-and-forth and constant context switching. If a runtime switch would wipe state,
say what to persist first. (3) State clearly what the agent has finished vs. what only the user can
do (runs, uploads, auth). (4) Runtime/cost trade-offs are the user's call once laid out; don't
relitigate after they've chosen.

## Natural next steps (whenever resumed)

1. ~~Resolve the two session-3 gaps~~ — done, session 4 (commits `816628b`,
   `852c6fe`). See "Session 4 result" above.
2. ~~Pick final reciters~~ — done, session 5: 9 reciters, user's own pick.
   ~~Dataset scale~~ — done, session 5: 350 ayat, dual-script. See "Session 5
   result" above.
3. ~~Dataset build~~ — done and verified, session 5 (Task 5, agent commits
   `8cac7a6`, `8c8c9a1`). 350 ayat × 9 reciters, 700 captions, 3,150 mp3s,
   8.31 hrs audio, 6,300 training pairs (6,280 after gatekeeping, see session 7).
   ~~Broader low-quality gatekeeping pass~~ — done, session 6 (Tasks 6-8,
   commits `df963c7`/`fa56df8`/`dcec1a5`). Integrity independently verified
   (3150/3150 MD5-match GCS), bandwidth check retired as non-discriminative,
   2 files confirmed corrupt at the source, policy locked to flag-only/never
   auto-edit. See "Session 6 result" above for full detail. Task 9 review done, session 7: all flagged
   files kept, and the 2 "corrupt" files reinstated as false positives. **Gatekeeping is closed; dataset is final (3,140 files / 6,280 pairs).**
   See "Session 7 result".
4. ~~Draft assembly + training-config specs~~ — done, session 8 (Tasks 13, 14; see "Session 8
   result"). Dataset built and uploaded; configs and persistence on `pron-lora-ar-only`.
5. ~~L4 smoke + A100 real run~~ — done, session 9 (Tasks 14b/14c). Smoke PASS (`258a098`); 6,100-step
   real run completed cleanly, 4 checkpoints + final in local + GCS. See "Session 9 result" above for
   the loss trend, the backup-daemon settle bug (fixed), and the A100-overkill finding.
6. **Start session 10 here (CPU, no GPU needed yet):** resolve the alpha
   bake-in-vs-runtime open question — read the merge code path and how v2's
   LoRA currently reaches `audio.cpp` inference (`docs/INFERENCE.md`,
   `convert_aitoolkit_yue2_lora.py`) — see "Control knobs for pronunciation
   strength" above. This decides whether the alpha × checkpoint sweep is cheap
   or expensive, so settle it before writing Task 15, not after.
7. Write **Task 15, the merge tool** (main repo, branch `pron-lora-ar-only`; CPU-only). Settle the
   remaining design questions at the end of `SESSION_08.md` too; test `alpha = 0` reproduces v2 with
   a precisely defined invariant. Note v2's LoRA covers AR and NAR, so pron's AR-only deltas are
   additive with v2's AR part.
8. Also CPU-only, can be done same session as 6-7: write the offline AR-loss replay script over the
   180 `val` pairs per checkpoint (`PRON_LORA_VERIFICATION.md` A3) — loss curves alone don't
   establish whether pronunciation actually improved.
9. **Only once 6-8 are done, switch to L4** (not A100 — session 9 found this workload
   data/CPU-bound, L4 gets ~equal throughput for less CU/h) to actually run: the val-loss replay,
   the merge, and the alpha sweep on `INFERENCE/yue2_eval_heldout/` lyrics using checkpoint × alpha
   combinations (stopping rule), plus the letter-substitution scorecard; listen for
   recitation-cadence bleed (stretched vowels, wrap-up-early).

## Maintaining this file

Update it proactively (no need to be asked) when a real *insight* lands — a
decision, a rejected idea and why, a corrected assumption, a guardrail. Not
details: if it's greppable in a terminal, it doesn't belong here; if it took a
dedicated script/agent exploration to learn, it does. Keep entries terse and
correct stale claims in place rather than appending contradictions.
