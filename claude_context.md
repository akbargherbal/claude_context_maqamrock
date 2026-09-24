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

**Resolved, session 10:** each alpha value requires baking a separate merged
file — not a live runtime slider. audio.cpp and ai-toolkit each load exactly
one LoRA network per expert (`yue2.ar_lora`/`yue2.nar_lora`, one file + one
scale each — `docs/yue2-gguf-lora-findings.md`, `DECISIONS.md`); confirmed
still current by reading the audio.cpp GitHub repo directly session 10
(PR #586, PR #614, releases through today) — no multi-adapter-per-stage
support has been added. `docs/FUTURE_PRONUNCIATION_LORA.md` §3 had already
worked this out back in September, just never cross-referenced into this
question. Method: rank-concatenation (v2 rank 32 + pron rank 8 → rank 40),
not dense-delta materialization; alpha folds into pron's B (the up-projection,
the side ai-toolkit's own scale convention lands on). Fully implemented,
tested, and documented — see "Session 10 result" below and
`docs/PRON_LORA_MERGE.md`.

## Explicitly open / not yet decided

- ~~**Dataset scale**~~ — resolved session 5: **350 ayat, dual-script, 9
  reciters.** See "Session 5 result" below. (My original ~10-20-ayat pilot
  proposal was superseded, not followed — worth remembering next time a
  "small pilot" instinct gets proposed here without re-checking it against
  what's actually cheap to get.)
- **Sessions 2–9 (condensed) — shortlist → dataset build → gatekeeping → training pipeline → real A100 run, all done and verified:**
  - **Sessions 2–3:** built initial 40-ayah shortlist (25 type-A/15 type-B, all four letters ≥16 ayat), dual-script JSONL captions, `strip_pause_marks()`. Locked the pron-caption template: positive-only, no melisma/vibrato mention even negated (negation primes the concept in gen-music models).
  - **Session 4:** confirmed the bucket's own metadata has no style/type field at all — reciter classification falls back to name-keyword matching + a curated 26-name `KNOWN_MURATTAL` map (documented, not a hard data source). Measured real duration data on 3 shortlisted ayat × 2 reciters, confirming `Husary_Muallim` recites ~2x slower than `Minshawy_Murattal` — a real pacing gap, not an assumption. **Standing lesson:** the agent's plain-text chat reports don't survive the push intact — verify from commits/files, not prose, every time.
  - **Session 5 — dataset built and verified complete:** **9 reciters**, user's own pick — `Husary_128kbps` (standard, **not** the earlier-flagged `Husary_Muallim`), `Abdul_Basit_Murattal_192kbps`, `Abu_Bakr_Ash-Shaatree_128kbps`, `Minshawy_Murattal_128kbps`, `Hudhaify_128kbps`, `Muhammad_Ayyoub_128kbps`, `Yaser_Salamah_128kbps`, `aziz_alili_128kbps`, `Abdullah_Basfar_192kbps` — × **350 ayat** (up from 40) = **8.31 hrs donor audio** (3,150 mp3s, 0 missing/corrupt at ffprobe level), **6,300 dual-script training pairs**, 700 caption files. Kept Tanzil script pairing at 2 editions (uthmani + simple), deliberately not expanded to all 6 variants (the other 4 are diacritic-reduced, working against the pronunciation goal). One anomaly surfaced: 37/3,150 files at unexpected bitrate, 14 of them (`Abu_Bakr_Ash-Shaatree`) genuinely low quality (11025Hz/mono/24kbps). User reframed the follow-up as a **broader low-quality gatekeeping pass**, not just a patch on those 14.
  - **Session 6 — gatekeeping pass:** data integrity independently re-verified (local MD5 vs GCS object MD5, 3150/3150 match — don't trust an untracked-data "nothing modified" self-report, verify against the source). Bandwidth/rolloff check retired as non-discriminative for this content (quiet recitation never carries much energy above ~9.5kHz regardless of true quality). Found **2 genuinely corrupt files at the source** (`Hudhaify_128kbps/023005`, `aziz_alili_128kbps/086008` — libsndfile decode fails even after a fresh re-download, same MD5). **Policy locked in: flag audio-quality issues for a human decision, never auto-edit** — a non-destructive silence-trim was built and demoed, then rejected on principle by the user even at tiny scale (0.68% of runtime); **don't re-propose audio editing (trim/normalize/etc.) without re-litigating this first.** New silence check (longest continuous silent run ≥3.0s) flagged 19 files; new loudness check (integrated LUFS >3dB from that *same reciter's* own median) flagged 312 files, read as normal vocal-dynamics variation, not a defect — not treated as an exclusion candidate without more scrutiny. A ≥8h duration-floor / "backfill ayat to compensate exclusions" idea was raised but not acted on preemptively (exclusions are a (reciter, ayah)-pair-level lever, cheaper than dropping whole ayat).
  - **Session 7 — gatekeeping CLOSED, dataset final:** user listened to all flagged audio — **kept all 19 silence-flagged and all 312 loudness-flagged files** (loudness/pause-length are performance traits, not exclusion grounds; **the only valid exclusion criteria are defects that damage the consonant signal** — corrupt/truncated audio or audio/text mismatch; don't propose new quality checks that measure performance). The 2 "corrupt" files were **false positives** — they decode fully under ffmpeg/torchaudio/librosa (torchaudio is what the training loader actually uses), only libsndfile fails — reinstated. The 14 low-format Abu_Bakr files were compared against Abu_Bakr's own normal files on HF-energy/consonant-proxy metrics with a rule fixed in advance: **10 disqualified, 4 kept** (017105, 041003, 041013, 041035). **Final dataset, locked: 3,140 files (Abu_Bakr 340 ayat, other 8 reciters 350 each) = 6,280 dual-script pairs**, 10 pair-level exclusions, all Abu_Bakr. Assembly design agreed: flat folder of `.mp3`+matching `.txt`, stem `<reciter>_<key>_<script>`, audio copied not symlinked, blank `trigger_word` (else v2's `arabmaqamrock` prepends to every caption), `network_kwargs.ignore_if_contains: ["transformer.nar"]`, rank well below v2's 32, steps set by epoch (not v2's raw step count), ~10 ayat held out at the ayah level for validation. Declined a duration-per-word audio/text-mismatch check — the user's own sampling never found wrong text, and that's enough for them.
  - **Session 8 — pipeline built, smoke test queued:** confirmed epoch-based step count, ayah-level ~10-ayat hold-out; landed on **rank 8** (linear/alpha 8/8), **1 epoch with quarter-epoch checkpoints** (doubles as the alpha-sweep stopping rule), **in-training sampling off**. Dataset assembled and uploaded to GCS: 6,100 train / 180 val / 16 smoke pairs (unique train audio 8.005h, val 0.26h; 12,200/360/32 files) as a **sibling GCS prefix** to v2's dataset — never inside it, since v2's job rsyncs that folder wholesale. All pron-LoRA work lives on a new branch, **`pron-lora-ar-only`** (main repo) — never on `main`. Key technical findings, verified from source: saved-weight key names differ from training names (`transformer.ar.*`→**`text_encoders.*`**, `transformer.nar.*`→**`diffusion_model.*`** — this is the smoke-pass criterion); `trigger_word: ""` prepends nothing; **no audio validation loss exists in ai-toolkit** (an offline loss-replay script is needed instead, not built yet); the AR LoRA trains only on `ar_ce`+`ar_kl` (NAR forward still runs every step, wasted compute). Runtime plan (user's CU-driven call): **L4 smoke test → A100 real run**.
  - **Session 9 — L4 smoke PASSED, A100 real run completed cleanly:** smoke test: `text_encoders.*` 224 keys, `diffusion_model.*` 0, rank 8 everywhere, all losses finite — PASS. Real run: 6,100 steps, no OOM, VRAM flat ~10.5GB; `loss/ar_ce` fell 5.20→4.40 (diminishing returns after ~step 1500); `loss/ar_kl` rose but stayed bounded (max 2.19, same shape as v1/v2 — not a blow-up, but the lever to pull later if AR over-drift shows up). 4 checkpoints (1525/3050/4575/6100) + final, all backed up locally + GCS. **Real bug found and fixed live:** `backup_to_gcp.py`'s settle-check never excluded the continuously-rewritten TensorBoard event file, so no checkpoint reached GCS for ~1h despite everything else looking synced — fixed by excluding tensorboard files from the settle watcher (generalizable: any settle/sync watcher on a training output folder must exclude continuously-rewritten log files). **A100 was overkill for this job** (22% median GPU util, only 10.5/80GB VRAM, barely faster than the L4 smoke) — this workload is data/CPU-bound, not compute-bound. **Guardrail: default to L4, not A100, for future short-clip jobs on this dataset shape** — don't assume the A100 throughput advantage (only seen on v1/v2's whole-song clips) transfers here.
- **Session 10 result (alpha question resolved; Task 15 written, sent, and completed):**
  - This was a CPU-only session by design (user's call, made before any work started — matches
    items 6-8 being CPU-only in "Natural next steps"; no Colab/GPU touched at all).
  - **Alpha bake-in-vs-runtime resolved**, closing the open question from session 9: each alpha
    value needs its own baked+converted file, confirmed two ways — (a) the answer was already
    sitting in `docs/FUTURE_PRONUNCIATION_LORA.md` §3, written back in September, just never
    cross-referenced into the session-9 question; (b) re-verified live against the audio.cpp
    GitHub repo (PR #586, PR #614, releases) — still exactly one `ar_lora`/`nar_lora` slot each,
    no multi-adapter-per-stage support added since. **Lesson worth repeating: check whether an
    answer already exists in the repo's own docs before treating a question as open** — this one
    had been sitting unresolved for two sessions while already answered elsewhere.
  - **Task 15 sent as one complete prompt** (rank-concatenation method, alpha folded into
    pron's B, precise sha256-based bit-for-bit invariant definition, CPU-only, no base-GGUF
    touch) — user's agent completed it same session, commits `e654c4f` (merge tool) and
    `32ffc02` (e2e invariant test), branch `pron-lora-ar-only`.
  - **Real gotcha the agent found, not in the spec:** the converter
    (`convert_aitoolkit_yue2_lora.py`) refuses mixed LoRA ranks across AR/NAR branches. A merged
    file is AR rank 40 (32+8) but NAR stays rank 32, so NAR needs zero-padding to rank 40 at any
    nonzero alpha (numerically a no-op — padded rows/cols are zero on one side of the product).
    At alpha=0 no pron block is concatenated, so no padding fires either, both branches stay
    rank 32 — which is *why* alpha=0 output is v2 verbatim. Documented with file:line in
    `docs/PRON_LORA_MERGE.md`.
  - **Exact ai-toolkit scaling convention, verified against source** (also in that doc, pinned
    to `ostris/ai-toolkit@460c29b`): a module's delta is `multiplier * (alpha/rank) * (B @ A)`;
    both v2 and pron were trained `alpha == rank`, so each saved file's delta is exactly `B @ A`.
    The merge folds the user's `--alpha` dial into `B_pron` (the up-projection — the side
    ai-toolkit's own runtime scale lands on), not `A_pron`; algebraically equivalent either way,
    chosen to mirror the existing convention.
  - **Verified directly this session, not just from the agent's commit message/doc** (standing
    project lesson: agent reports have failed to survive intact multiple times before): read
    `merge_pron_lora.py` end to end — the alpha=0 short-circuit and the NAR padding logic are
    correct on inspection. Installed CPU torch + safetensors in a fresh sandbox and **actually
    ran** `tests/test_merge_pron_lora.py`: **11 passed, 1 skipped**. The skip is
    `test_alpha_zero_reproduces_live_converted_v2` — it needs the real ~100+ MB v2/pron
    `.safetensors` files and the audio.cpp converter script, neither available outside the
    project's own GCS-connected VM, so it skips cleanly on a bare clone by design.
  - **One thing NOT independently verified, flagged for the user rather than silently trusted:**
    the actual "alpha=0 sha256 byte-match — PASS" claim in `docs/PRON_LORA_MERGE.md` (matching
    the live converted v2 adapters' hashes `747d5cfe…`/`ad2c8d86…`) came from the agent running
    against real files on its own box; no GCS access existed in the verification sandbox to
    re-check those specific hashes independently. This is exactly the class of claim this
    project's history says to verify rather than trust prose on (sessions 2-4). **Next action:**
    have the agent re-run that one skipped test live, on the VM, in front of the user, before
    treating the no-regression invariant as fully trusted — cheap (~3.5s per the doc) and it's
    the one guarantee the whole two-LoRA design depends on. Not yet done as of session 10 end.
  - Also fixed in passing: the user's GCS bucket path is `OSTRIS_Arabic_Suno_Finetuning`
    (**with** "Arabic") — Claude wrote it without "Arabic" in an early draft of the Task 15
    prompt this session and caught/corrected it before sending, after the user asked to
    double-check env vars. Worth double-checking this exact string again if it's ever retyped
    by hand rather than copied.

- **Session 11 result (Task 16 done in full: merge invariant re-verified live PASS; AR-loss replay script built + CPU-benchmarked, sweep deliberately NOT run):**
  - CPU-only session, using a new personal VS Code-anywhere/opencode(DeepSeek) workflow notebook instead of a
    Claude-run terminal — same underlying repo/branch, different way of relaying commands to the agent.
    **Notebook gotcha caught before use, worth re-checking if this notebook is reused:** it cloned the main
    repo but never ran `git checkout pron-lora-ar-only`, and never exported `GCP_PRON_DATASET_PATH` — both
    are silent-failure risks (stays on `main`, or `/content/pron_dataset` never restores) that were only
    caught by inspection this session, not by the notebook itself.
  - **Task 16A — the one untrusted claim from session 10 is now independently confirmed, live, PASS.**
    Re-ran `pytest tests/test_merge_pron_lora.py -v -s` on a fresh VM with real v2+pron `.safetensors` staged
    from GCS: all 12 tests ran (none skipped this time), including
    `test_alpha_zero_reproduces_live_converted_v2`. Both hashes matched the doc's table exactly: converted AR
    `747d5cfe…`, converted NAR `ad2c8d86…`. **The no-regression invariant the whole two-LoRA merge design
    depends on is now verified, not just agent-reported.** Commit `c02e37d`.
  - **Task 16B — `offline_ar_loss_replay.py` built** (reuses the real per-item AR forward/loss path,
    `yue2_model.py`'s `_prefix_segment`/`_item_prefix_and_abc`/`_ar_inputs`/`_ar_losses`; forward-only, no
    trainer/backward/optimizer/NAR flow). 9 new tests. Commit `b087bac`.
  - **Real, load-bearing finding: CPU has no `torch._int_mm` kernel**, so the int8-quantized base checkpoint
    can't run quantized math on CPU — it silently falls back to dequantized bf16 (W8A16) matmuls. Output is
    numerically correct, but there is **no speedup from quantization on CPU**, which is most of why one
    forward pass is so heavy. This is a hardware-capability gap, not a tuning problem — it doesn't shrink
    with a bigger/faster CPU box, only with a CUDA GPU (L4 or better) where the kernel actually engages.
  - **8-item CPU benchmark (final checkpoint), verbatim numbers, committed:** mean `loss/ar_ce = 4.0652`,
    `loss/ar_kl = 1.3636` — plausible next to the training-time final-checkpoint figures (`ar_ce` ~4.16-4.40,
    `ar_kl` ~1.53), no red flags. Per-item: tokenize 3.70s + AR-loss forward 272.79s = **276.49s/item**
    (model load 72.6s, one-time, not per-item).
  - **Extrapolation for the full sweep (180 val pairs × 5 checkpoints = 900 forward passes): ≈68.2 hours
    on this CPU box** (272.79s × 900 = 245,511s), plus ~0.9h of (cacheable) tokenization. Recorded in
    `docs/PRON_LORA_VERIFICATION.md` §A3b.
  - **Script correctly stopped exactly where instructed:** no 900-pass sweep, no checkpoint comparison, no
    pronunciation-quality conclusion drawn — commit message explicitly hands the CPU-vs-GPU call back to the
    user. It reported the `torch._int_mm` fact but did **not** itself draw the inference that GPU should give
    a *qualitative* (kernel-engages) speedup rather than just a proportional one from more cores — that
    reasoning happened in this WebUI session, not in the agent's own output, and needs to be stated explicitly
    in whatever prompt is written for the GPU/L4 step rather than assumed already known to the agent.
  - **Given the ~68h number, CPU is not a realistic option for the full sweep — this settles item 9 below
    as effectively mandatory, not just "probably faster on GPU."** Still L4, not A100 (unchanged reasoning
    from session 9: this workload is data/CPU-bound, not compute-bound).
  - **`PROGRESS.md` was not updated with a Task 16 milestone entry this session** (the detail lives in
    `docs/PRON_LORA_VERIFICATION.md` §A3b and the two commit messages instead, both git-preserved and
    sufficient) — worth a one-line PROGRESS.md entry next session if it's ever missed when skimming history.
  - **`agent_notes/current.md` is git-ignored by design** (see AGENTS.md) and only reaches GCS if
    `backup_to_gcp.py`'s daemon is running; this was a script-dev/benchmark session with no training job, so
    that daemon likely was never started. Don't assume it survived — verify or re-request it fresh at the
    start of the GPU session rather than relying on a carryover that was never confirmed pushed anywhere.

- **Session 12 result (offline AR-loss sweep DONE on L4; checkpoint choice still open):**
  - Ran on a fresh Colab L4. **Only 4 trainable checkpoints exist** (1525/3050/4575/`final`=6100), so the full
    sweep is 180 val pairs x 4 = **720 passes** (the "900 / 5 checkpoints" in session 11 + `PRON_LORA_VERIFICATION.md`
    §A3b was wrong; agent fixed the doc). Agent commit `8f6c57e` (`results/*.json`, branch `pron-lora-ar-only`).
  - **Kernel-engages prediction confirmed:** 0.61 s/item on L4 vs 272.79 s/item on CPU (~450x, qualitative not
    proportional); no W8A16-fallback warning in the sweep log. Whole sweep ~23 min, zero errors. Verified from the raw
    JSONs (n_items=180, device=cuda), not just commit prose.
  - **Mean losses (180 val pairs):** 1525: ar_ce 5.1157 / ar_kl 0.7180; 3050: 4.6202 / 1.2715; 4575: 4.5053 / 1.4241;
    final: 4.4507 / 1.4828. ar_ce gains: -0.50 (1525->3050), then only -0.11, -0.05; ar_kl rises the whole way.
    So **3050 or 4575 may be a better pronunciation/drift trade than `final`** — but ar_ce on val is NOT a
    pronunciation measure by itself; only the held-out-lyrics alpha sweep + ear-check can decide.
  - Preflight note: `backup_to_gcp.py`/`gpu_logger.py` were NOT running when the agent checked; it started them.
    Check manually before any job that writes files worth keeping.
  - **Session 12 ended prematurely (max session tokens) right after this result** — the log
    (`session_logs/SESSION_12.json`) ends on Claude asking which checkpoint x alpha grid to try. Not answered.
  - **Watch-item for the next step:** `docs/INFERENCE.md` says the prebuilt `audiocpp_cli` in GCS is **sm_75 (T4)**;
    L4 is sm_89 -> may need the per-arch build in `docs/audiocpp_gpu_arch_builds.md` (or a prebuilt sm_89 binary
    already in GCS — check first). Don't assume the T4 binary runs/performs right on L4.

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
6. ~~Resolve the alpha bake-in-vs-runtime open question~~ — done, session 10. See
   "Session 10 result" below and the resolved note under "Control knobs for
   pronunciation strength" above.
7. ~~Write Task 15, the merge tool~~ — done, session 10 (commits `e654c4f`, `32ffc02`,
   branch `pron-lora-ar-only`). `merge_pron_lora.py` + `docs/PRON_LORA_MERGE.md` +
   12 unit tests (11 run + verified passing this session; 1 real end-to-end test
   skips without the live GCS artifacts — see "Session 10 result" for the one
   thing still worth watching run live before fully trusting it).
8. ~~Write + CPU-benchmark the offline AR-loss replay script~~ — done, session 11 (Task 16B,
   commit `b087bac`). CPU benchmark: ~68.2h extrapolated for the full 900-pass sweep — CPU
   confirmed **not viable** for the full run (no `torch._int_mm` on CPU, dequant fallback, not
   just slow). See "Session 11 result" above. Also done, session 11: Task 15's alpha=0
   no-regression invariant re-verified live, PASS (Task 16A, commit `c02e37d`) — no longer an
   open trust item.
9. ~~Offline AR-loss replay sweep on L4~~ — done, session 12 (commit `8f6c57e`); see "Session 12 result".
10. **Start session 13 here (L4, not A100).** Merge (`merge_pron_lora.py`) at chosen checkpoint x alpha, generate the
    4 held-out songs (`INFERENCE/yue2_eval_heldout/`) per config via `INFERENCE/run_one.sh`/`generate.py`, always
    including alpha=0 (= v2) as the baseline, same seed. Then the letter-substitution scorecard (method not yet designed —
    sung-vocal ASR is noisy; the user's ear is the primary judge) and listen for recitation-cadence bleed (stretched
    vowels, wrap-up-early). Preflight: `git checkout pron-lora-ar-only`, all 3 GCP env exports, sidecars running,
    audiocpp_cli arch (see session 12 watch-item).

## Maintaining this file

Update it proactively (no need to be asked) when a real *insight* lands — a
decision, a rejected idea and why, a corrected assumption, a guardrail. Not
details: if it's greppable in a terminal, it doesn't belong here; if it took a
dedicated script/agent exploration to learn, it does. Keep entries terse and
correct stale claims in place rather than appending contradictions.
