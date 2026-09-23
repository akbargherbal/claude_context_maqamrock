Claude Last Prompt in this session to AI agent:

```
# Task 5: build the 350-ayah training dataset

Repo: arabic-phoneme-difficulty-quran, branch `pron-lora-prep` (continue on
it; commit and push). This is a real build, not analysis-only — but still no
training, no model repos, no merges.

## Step 1 — rebuild the shortlist at 350 scale
python scripts/pron_shortlist_v2.py \
    --scores data/pron/v2/ayah_scores.csv \
    --out data/pron/pron_shortlist_v2.csv \
    --type-a-count 219 --type-b-count 131 --min-letter-coverage 12
Overwrite the existing 40-row file (it's superseded). Confirm in the commit
message: method used (should be density-within-type, no coverage-floor
fallback needed), final letter coverage per ح/خ/ع/ض, and the excluded-marks
list (should still be the same 2 special-reading ayat: 011041, 012011).

## Step 2 — re-check reciter coverage at the NEW 350-ayah scale
python scripts/audio_audit.py \
    --shortlist data/pron/pron_shortlist_v2.csv \
    --out data/pron/reciter_audit.json
This is a listing-only pass (no download), ~30s. The 9 reciters in
reciters_shortlist/shortlist.txt (main branch) previously had full coverage
of the OLD 40-ayah set only — this re-checks all 350. If any reciter comes
back short of full coverage on any of the 350, STOP and report which
reciter(s) and which keys before proceeding to Step 4 — don't silently drop
ayat or reciters to route around a gap.

## Step 3 — redo the dual-script join and captions at 350 scale
python scripts/pron_dualscript.py \
    --shortlist data/pron/pron_shortlist_v2.csv \
    --out data/pron_shortlist_dualscript.jsonl
python scripts/pron_captions.py \
    --dualscript data/pron_shortlist_dualscript.jsonl \
    --out-dir data/pron/captions
Expect 350 JSONL records, 700 caption .txt files (350 × uthmani/simple).
Run the existing suite (pytest tests/ -q) — it doesn't hardcode the old
40/80 counts, so it should pass unchanged; report the actual pass count.

## Step 4 — download the donor audio
For each of the 9 reciters in reciters_shortlist/shortlist.txt, download all
350 shortlisted SSSAAA.mp3 files from
gs://sheikh-fitzgerald-backup/ARABIC_DATA/QURAN_VERSE_BY_VERSE_RECITATIONS_DATASETS/
— 3,150 files total. Do NOT commit the audio to git (binary, too large);
place it wherever the actual training run will read from and tell me that
path explicitly in your report. Verify count on disk == 3,150 before
declaring done.

## Step 5 — verify what actually landed
Run ffprobe across the full 3,150 files (not just a sample this time — this
is the real corpus). Commit a summary (not prose) to
data/pron/donor_audio_manifest.json: per-reciter file count, total duration,
any files that failed to download or came back 0-duration/corrupt. Flag
anything unexpected rather than averaging it away.

## Report
Put every number above (final letter coverage, coverage-audit result, JSONL/
caption counts, test pass count, download counts, total duration, any
failures) into the commit message bodies — chat reports have not survived
the push in 3 of the last 4 sessions on this repo.
```
The AI agent is now working on the request.