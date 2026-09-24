# SESSION 08 — prompt log (continuity for session 9)

State at end of session 8: Task 13 done and verified (secondary repo). Task 14 CPU phase
done and pushed (main repo, branch `pron-lora-ar-only`, commit `0220aa8`). Dataset
uploaded to GCS and counts verified. User was about to start an **L4** GPU VM to run the
smoke test (Task 14b, L4 version below); after it passes, switch to an **A100** VM for the
real 6,100-step run (Task 14c below). Merge tool (Task 15) not yet written.

Runtime plan (user's call, CU-budget driven): smoke on L4 (1.52 CU/h), real run on A100
(6.77 CU/h). Claude's view was "one A100 VM for both" (roughly break-even, less fuss); the
user chose L4-first anyway. Each switch wipes /content, so the A100 VM needs its own full
setup.sh + dataset restore. Smoke is a pure preflight; nothing from it is reused.
The smoke run's step time / VRAM are L4 numbers: do NOT use them as the A100 ETA.

GCS layout: v2 dataset `gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/dataset/`
(-> /content/yue2_dataset via $GCP_DATASET_PATH). Pron dataset sibling prefix
`.../OSTRIS_Arabic_Suno_Finetuning/pron_dataset/{train,val,smoke}/` (-> /content/pron_dataset via
opt-in $GCP_PRON_DATASET_PATH). Run backups: `<base>/<run-name>/output/`.

Launch block for a fresh GPU VM (plus the usual exports: GCP_DATASET_PATH, GCP_BACKUP_BASE, tokens):

```
git clone https://github.com/akbargherbal/maqamrock-yue2-lora-finetuning.git
cd maqamrock-yue2-lora-finetuning
git checkout pron-lora-ar-only
export GCP_PRON_DATASET_PATH=gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/pron_dataset
bash bootstrap/setup.sh > /content/logs/setup.log 2>&1 &
```

---

## Task 13 (sent, completed: secondary repo, commit df3193d)

```
Task 13 — Assemble the pronunciation training set (secondary repo, branch pron-lora-prep; no GPU, no GCS writes)

Context: the dataset is final (3,140 donor mp3s, 10 pair-level exclusions in data/pron/training_pair_exclusions.json, 700 captions in data/pron/captions/). Build the flat folders ai-toolkit will read. Do NOT modify donor audio or captions, re-run earlier pipeline stages, re-litigate the exclusions, or add new quality checks.

Spec
1. New script scripts/pron_assemble_training_set.py plus tests. Inputs: pron_shortlist_v2.csv, donor_audio/, captions/, training_pair_exclusions.json, donor_audio_manifest.json.
2. Pairs: every (reciter, key, script) for the 9 reciters x 350 keys x {uthmani, simple}, minus exclusions (an excluded (reciter,key) drops BOTH scripts). Stem = <reciter>_<key>_<script>. Write <stem>.mp3 = byte-for-byte copy of donor_audio/<reciter>/<key>.mp3 (no re-encode/resample/trim/normalize, no symlinks) and <stem>.txt = byte-for-byte copy of captions/<key>_<script>.txt. The same audio therefore appears under two stems, one per script.
3. Hold-out, ayah-level, seed=42, deterministic: 10 ayat. All their pairs (9 reciters x 2 scripts = 180) go to val/, never train/. Constraints: 6 type A + 4 type B; together cover all four letters (>=1 chosen ayah with count>0 for each of ح خ ع ض); none appear in the exclusion list; the ayah's simple-script text is unique among all 350 shortlisted ayat (repeated refrains would leak into train). Record the chosen keys and the rationale in holdout_manifest.json.
4. Also build a smoke/ folder: 16 pairs drawn from train (4 ayat x 2 reciters x 2 scripts), for a cheap pipeline test (latent caching runs over the whole folder, so smoke must be tiny).
5. Outputs (gitignored): data/pron/training_set/{train,val,smoke}/. Committed: data/pron/training_set_manifest.json (per pair: split, reciter, key, script, stem, audio MD5, source path), data/pron/holdout_manifest.json, data/pron/training_set_report.json.
6. Idempotent; refuse to write into a non-empty output dir without --force.

Verification: compute everything below by RE-READING the output folders, not from the build loop's own counters.
- Counts: train pairs, val pairs, smoke pairs; per reciter and per script. Expected train = 6,280 - 180 = 6,100, val = 180; report actuals if different.
- Every mp3 MD5 equals its donor source MD5; every txt is byte-identical to its source caption.
- 1:1 mp3<->txt stems, no orphans, no symlinks.
- No excluded pair present (either script); no held-out key in train/ under any reciter/script; smoke is a subset of train.
- No txt contains "arabmaqamrock", "..." or "…".
- Total audio duration for train and val (from the existing manifest, no new ffprobe); total disk size of the three folders.
Put ALL of these numbers in the committed report AND the commit message. The existing test suite (452) must still pass.
```

---

## Task 14 — CPU phase (sent, completed: main repo, branch pron-lora-ar-only, commit 0220aa8)

```
Task 14 (CPU phase) — Pronunciation LoRA: config, smoke-test preparation, persistence (main repo: /content/maqamrock-yue2-lora-finetuning)

THIS VM HAS NO GPU (Colab CPU runtime). Do not attempt to run any training, smoke run, model load, or nvidia-smi-dependent step. Everything below except the final smoke execution is doable on CPU. The smoke run itself happens LATER on a different (GPU) VM, in a separate task.

BRANCH: create a new branch `pron-lora-ar-only` off `main` and do ALL work there. Never commit to `main`.

RULES (from AGENTS.md, still binding): do NOT edit config/akbar_arabic_rock_lora.yml, do NOT touch /content/yue2_dataset or the v2 dataset's GCS prefix, do NOT start any run or upload anything yourself. You ARE authorized to create the new files listed below. For anything the user must run, hand over the exact command per the command-handover skill (foreground/detached, log path, stop command). Report results in committed files and the commit message, not in chat.

BACKGROUND (you have no memory of earlier sessions):
- Goal: train a SEPARATE small AR-only LoRA on Quran verse recitation (audio + fully diacritized text) to sharpen consonant articulation (ح خ ع ض). It will later be merged offline with the existing v2 style LoRA: W = W_base + 1.0*dW_style + alpha*dW_pron. The v2 checkpoint is never retrained or modified. It must NOT learn recitation style, so: AR-only scope (transformer.nar excluded), low rank, no trigger word, no style caption words.
- The training set was built by Task 13 in the secondary repo (/content/arabic-phoneme-difficulty-quran, branch pron-lora-prep, commit df3193d). It lives at /content/arabic-phoneme-difficulty-quran/data/pron/training_set/{train,val,smoke}/ (gitignored, ~1.07 GB, currently NOT in GCS): flat folders of <reciter>_<key>_<script>.mp3 + same-stem .txt. train = 6,100 pairs, val = 180 pairs (10 held-out ayat x 9 reciters x 2 scripts), smoke = 16 pairs (a subset of train). Caption layout: "Solo male voice, unaccompanied. Clear precise Arabic diction, measured pace.\n[Lyrics]\n[Verse]\n<one ayah>", no trigger word. Do NOT modify these files or re-run any dataset build.
- GCS layout (bucket already in use): the v2 dataset is gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/dataset/ (restored to /content/yue2_dataset by bootstrap/setup.sh job_dataset via $GCP_DATASET_PATH). Run backups go to <base>/<run-name>/output/ where <base> = $GCP_BACKUP_BASE = gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning. The pron dataset gets its OWN sibling prefix, gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/pron_dataset/ , and must never be placed inside dataset/ (job_dataset rsyncs all of dataset/ into /content/yue2_dataset and would contaminate v2's training folder).

A0. DATASET PLACEMENT. Make the Task 13 output available at /content/pron_dataset/train, /content/pron_dataset/val and /content/pron_dataset/smoke on this VM. A symlink of each folder to the existing location is fine; say in the report which you used, and verify ls counts: 12,200 files in train (6,100 mp3 + 6,100 txt), 360 in val, 32 in smoke.

A. READ-ONLY VERIFICATION against the ai-toolkit source (cite file:line for each). If ai-toolkit is not already present on this VM, clone it read-only into /content/ai-toolkit-src at the same repo/commit that bootstrap/setup.sh uses (read setup.sh to see how it clones), and do NOT run any pip install or heavy setup. Commit the results as docs/PRON_LORA_VERIFICATION.md:
 1. With network_kwargs.ignore_if_contains: ["transformer.nar"], the saved adapter contains only transformer.ar.* keys (static reading only; the empirical confirmation is PENDING and will be added after the GPU smoke run; leave a clearly marked "PENDING" section for it).
 2. The exact trigger_word value ("" / null / omitted) that guarantees NOTHING is prepended to captions (v2's injector only prepends when the trigger appears zero times in a caption).
 3. Whether diffusion_trainer (yue2 arch) supports a validation / eval dataset. If not, say so and propose the smallest offline alternative: a script that loads each checkpoint and computes loss on /content/pron_dataset/val through the same loss code path as training. Do not build it.
 4. What loss/ar_kl is measured against; whether ar_kl_weight and content_or_style still mean the same thing when the NAR has no trainable params; whether the NAR forward/loss still runs (cost). Report only, change nothing.
 5. Where the latent cache is written, and that a run on /content/pron_dataset/* cannot collide with or reuse v2's cache.
 6. Whether train.steps counts optimizer steps or micro-batches (matters under gradient_accumulation), and whether conv / conv_alpha have any effect on an AR-only linear LoRA.
 7. Whether bootstrap/setup.sh in training mode still hard-requires $GCP_DATASET_PATH (v2's dataset) even when the user only wants the pron dataset. Report it and what it costs (size of the extra v2 download if you can read it from the repo docs); do NOT change job_dataset.

B. config/pron_lora_ar_only.yml — a copy of config/akbar_arabic_rock_lora.yml with ONLY these changes (list them in a header comment):
 - config name, log_dir, and run name: pron_lora_ar_only_r8 (training_folder stays /content/ai-toolkit/output)
 - trigger_word: the blank form determined in A2
 - network: linear 8, linear_alpha 8 (v2 = 32/32); network_kwargs.ignore_if_contains: ["transformer.nar"]
 - datasets[0].folder_path: /content/pron_dataset/train
 - train.steps: 6100 (= 1 epoch = 6,100 train pairs / (batch_size 1 x grad_accum 1)); show the arithmetic in a comment. If A6 shows steps counts something other than optimizer steps, adjust so it is still exactly 1 epoch and say so in the report.
 - save.save_every: 1525 (quarter epoch); save.max_step_saves_to_keep: 12 (v2's value) so the 1525 / 3050 / 4575 / 6100 checkpoints all survive
 - sample.disable_sampling: true (evaluation will be a merged-alpha sweep on held-out lyrics, not in-training samples); remove the long sample.samples prompt block
 Everything else stays identical to v2 (lr 1e-4, adamw8bit, EMA 0.999, cot off, ar_kl_weight 0.2, train_window_frames 0, whole-clip, batch_size 1, grad_accum 1, dtype bf16, quantization).
 Validate the YAML parses (python -c "import yaml,sys; yaml.safe_load(open(...))") and show a unified diff of the two configs in the commit message.

C. SMOKE CONFIG (prepare only, DO NOT RUN). Create config/pron_lora_ar_only_smoke.yml = B with folder_path /content/pron_dataset/smoke, steps 10, save_every 10, run/config name pron_lora_ar_only_smoke. Record in docs/PRON_LORA.md the exact command the user will run later on the GPU VM: python run.py config/pron_lora_ar_only_smoke.yml -l /content/logs/train_smoke.log (foreground). Also record what to inspect afterwards: total safetensors key count, count of transformer.ar.* keys, count of transformer.nar.* keys (must be 0), LoRA rank from tensor shapes (must be 8), per-step time and peak VRAM.

D. PERSISTENCE (no uploads by you):
 1. backup_to_gcp.py: confirm TARGETS / --run-name cover the new output folder so checkpoints of run pron_lora_ar_only_r8 land at <base>/pron_lora_ar_only_r8/output/. Change the script only if they do not, and report which.
 2. bootstrap/setup.sh: add a SEPARATE function job_pron_dataset() that restores $GCP_PRON_DATASET_PATH into /content/pron_dataset using the same idempotent rsync pattern as job_dataset, with the completion marker at /content/pron_dataset/.bootstrap_complete (NOT inside train/, val/ or smoke/). It must be opt-in: it runs only if GCP_PRON_DATASET_PATH is set, so v2 training bootstraps behave exactly as before. Do NOT modify job_dataset, DATASET_LOCAL, or GCP_DATASET_PATH. Run `bash -n bootstrap/setup.sh` to syntax-check it.
 3. The intended value is GCP_PRON_DATASET_PATH=gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/pron_dataset . Give the user, in the runbook AND the commit message: (a) the exact upload command to run NOW from this VM using `gcloud storage rsync -r` for each of /content/pron_dataset/train, /val and /smoke to the matching subfolder of that prefix (verify syntax against the installed gcloud version with --help, and make sure symlinked source folders are followed/handled correctly; test with --dry-run first and show that output), (b) the exact `gcloud storage ls -r` command to verify the object counts afterwards (expect 12,200 in train, 360 in val, 32 in smoke), (c) the exact export line for the launching notebook / /root/.secrets.env.

E. DOCS. Write docs/PRON_LORA.md, a short runbook containing: what this run is and why it is AR-only; the CPU-phase upload command and verification from D3; the GPU-VM restore sequence (fresh VM: git clone the repo, git checkout pron-lora-ar-only, export GCP_PRON_DATASET_PATH plus the usual exports, run bootstrap/setup.sh, confirm /content/pron_dataset has the expected counts as real files); the smoke command and inspection checklist from C; the real-run command (python run.py config/pron_lora_ar_only.yml -l /content/logs/train_pron.log, foreground, user-typed only); the backup command with --run-name pron_lora_ar_only_r8; the four expected checkpoints. Add a two-line pointer to it in PROGRESS.md. Do not edit DECISIONS.md.

DELIVERABLES: everything committed on branch pron-lora-ar-only and pushed (the user supplies GitHub auth via bootstrap/github_auth.sh; ask if push fails; the branch MUST be on GitHub before the user leaves this VM). The commit message must state: the exact list of files created/changed; the A0 result (symlink or copy, file counts); the A1-A7 answers in one line each with file:line (A1's empirical confirmation marked PENDING); the diff summary versus the v2 config; the upload command (D3a) and verification command (D3b) verbatim. Verify by re-reading the committed files (git show --stat, and the config diff) before claiming anything is done. Stop after pushing and handing over the upload command; do not wait for the smoke run.
```

Result notes (verified by Claude from the commit): all deviations as specified; agent found
train.disable_sampling (not sample.disable_sampling) is the real field; A1 caveat (saved keys are
text_encoders.* / diffusion_model.*); fixed a latent backup_to_gcp.py --run-name bug. NOT delivered:
A7 (Claude answered it: setup.sh line 93 hard-requires GCP_DATASET_PATH), dry-run output; runbook
omitted `git checkout pron-lora-ar-only` (fixed by 14b step 4).

---

## Task 14b — GPU smoke test, **L4 version** (this is what the user runs first, on an L4 VM)

```
Task 14b (GPU phase, L4 smoke only) — smoke test and verdict (repo: /content/maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only; commit and push on that branch only, never main)

RULES (AGENTS.md still binding): do not edit config/akbar_arabic_rock_lora.yml, do not touch /content/yue2_dataset, do not start any training run yourself. The user types every training command in their own terminal; hand over exact commands (foreground/detached, log path, stop command per the command-handover skill). Report results in committed files and the commit message, not in chat.

CONTEXT: this is a fresh L4 GPU VM used ONLY for the smoke test. The real 6,100-step run will happen later on a different A100 VM, so DO NOT hand over a real-run command here. Task 14 (CPU phase, commit 0220aa8) created config/pron_lora_ar_only.yml (real run, 6,100 steps = 1 epoch, run name pron_lora_ar_only_r8), config/pron_lora_ar_only_smoke.yml (10 steps, run name pron_lora_ar_only_smoke), docs/PRON_LORA.md and docs/PRON_LORA_VERIFICATION.md. The pron dataset was uploaded to gs://akbar-december-2024-backup/OSTRIS_Arabic_Suno_Finetuning/pron_dataset/{train,val,smoke} and setup.sh's opt-in job_pron_dataset() should have restored it to /content/pron_dataset.

1. PREFLIGHT (report each result in the verification doc):
 a. setup.sh finished (check /content/logs/setup.log for completion and failures).
 b. /content/pron_dataset/train has 12,200 files (6,100 .mp3 + 6,100 .txt), val 360, smoke 32, as REAL files (not broken symlinks), and /content/pron_dataset/.bootstrap_complete exists.
 c. /content/ai-toolkit exists and `nvidia-smi` shows the GPU is free. Record the GPU model and total VRAM.
 d. Check that gpu_logger.py is running (pgrep -af); if not, give the user the exact start command from AGENTS.md. The backup daemon is NOT needed for the smoke run.
 If any preflight item fails, stop and diagnose; do not proceed.

2. SMOKE COMMAND. Hand the user this exact foreground command and wait for them to say it is done:
   cd /content/ai-toolkit
   python run.py /content/maqamrock-yue2-lora-finetuning/config/pron_lora_ar_only_smoke.yml -l /content/logs/train_smoke.log

3. INSPECT the smoke result (output/pron_lora_ar_only_smoke/*.safetensors, /content/logs/train_smoke.log, /content/logs/gpu_usage.csv). NOTE the saved key naming: on save, ai-toolkit renames AR keys to text_encoders.* and NAR keys to diffusion_model.* (yue2_model.py convert_lora_weights_before_save). Report:
 - total tensor count in the safetensors file, and counts grouped by key prefix (text_encoders.*, diffusion_model.*, transformer.*, anything else)
 - the set of distinct LoRA ranks read from the A/B (down/up) tensor shapes
 - tensor dtype(s) and file size
 - whether all 10 training losses in the log are finite (no NaN/inf), including loss/ar_ce and loss/ar_kl values
 - seconds per step and peak VRAM (from the log and the GPU CSV); label these as L4 numbers, warmup-inflated, NOT to be used as an A100 estimate
 - where the latent cache was written (must be under /content/pron_dataset/smoke/_latent_cache, and /content/yue2_dataset must be unmodified)
 PASS iff ALL hold: text_encoders.* count > 0; diffusion_model.* count == 0; no unexpected prefixes; every LoRA rank == 8; all losses finite; the run completed with no OOM or crash.

4. DOCS. In docs/PRON_LORA_VERIFICATION.md: replace the PENDING smoke section with these results and the PASS/FAIL verdict; add a section A7 with this finding: bootstrap/setup.sh line 93 hard-requires GCP_DATASET_PATH in training mode and `start_job dataset job_dataset` always runs, so a GPU VM also downloads the v2 dataset in the background (harmless; job_dataset deliberately not modified). In docs/PRON_LORA.md: fix the fresh-VM restore steps to include `git checkout pron-lora-ar-only` after the clone, and correct the expected-key wording to text_encoders.* / diffusion_model.*. Commit and push (the user supplies GitHub auth via bootstrap/github_auth.sh; ask if push fails). The pushed verdict is what the A100 VM will read, so the push MUST succeed before the user disconnects this VM.

5. VERDICT. Do NOT hand over any real-run command or ETA.
 - If PASS: state PASS, confirm the verdict commit is pushed (give the commit hash), and tell the user they can disconnect this L4 VM and start the A100 VM.
 - If FAIL: write the diagnosis and your proposed fix into the verification doc, commit, push, and stop.
 Also write agent_notes/current.md with the current state (overwrite, per AGENTS.md; verify the write before saying so).
```

---

## Task 14c — A100 VM: preflight + real-run handover (send on the A100 VM after smoke PASS on L4)

```
Task 14c (A100 VM) — preflight and real-run handover for the AR-only pronunciation LoRA (repo: /content/maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only; commit and push on that branch only, never main)

RULES (AGENTS.md still binding): do not edit config/akbar_arabic_rock_lora.yml, do not touch /content/yue2_dataset, do not start any training run yourself, and do NOT change config/pron_lora_ar_only.yml. The user types every training command in their own terminal; hand over exact commands (foreground/detached, log path, stop command per the command-handover skill). Report results in committed files and the commit message, not in chat.

CONTEXT: the smoke test already ran on a separate L4 VM and its verdict is committed in docs/PRON_LORA_VERIFICATION.md on this branch. This is a fresh A100 VM used for the real run: config/pron_lora_ar_only.yml, run name pron_lora_ar_only_r8, 6,100 steps = 1 epoch over 6,100 train pairs, checkpoints at steps 1525 / 3050 / 4575 / 6100 plus a final no-step file. Do NOT rerun the smoke test.

1. GATE: read docs/PRON_LORA_VERIFICATION.md. If the smoke section does not show an explicit PASS verdict, STOP and tell the user; do not proceed.

2. PREFLIGHT (append results to docs/PRON_LORA_VERIFICATION.md under an "A100 preflight" heading):
 a. setup.sh finished (/content/logs/setup.log complete, no failures).
 b. /content/pron_dataset/train has 12,200 files (6,100 .mp3 + 6,100 .txt), val 360, smoke 32, as REAL files, and /content/pron_dataset/.bootstrap_complete exists. No _latent_cache from the smoke VM should exist (fresh VM); latents will be built at the start of the real run, so note that the first minutes before step 1 are caching, not a hang.
 c. /content/ai-toolkit exists; `nvidia-smi` shows the GPU is free; record GPU model and total VRAM (expect A100).
 d. Start/confirm the backup daemon and the GPU logger. Give the user these exact commands if not running (verify paths against AGENTS.md first):
    cd /content/maqamrock-yue2-lora-finetuning
    setsid nohup python backup_to_gcp.py --run-name pron_lora_ar_only_r8 > /content/logs/gcp_backup_stdout.log 2>&1 & disown
    setsid nohup python gpu_logger.py --out /content/logs/gpu_usage.csv > /content/logs/gpu_logger_stdout.log 2>&1 & disown
    Confirm GCP_BACKUP_BASE is exported (echo it) and that the backup log is fresh.
 e. Confirm the config on disk is byte-identical to the one committed at the branch tip (git status clean for config/).
 If any item fails, stop and diagnose.

3. REAL-RUN HANDOVER. Give the user, exact and foreground (Ctrl+C stops it; do not detach):
    cd /content/ai-toolkit
    python run.py /content/maqamrock-yue2-lora-finetuning/config/pron_lora_ar_only.yml -l /content/logs/train_pron.log
 plus: the stop command; the exact RESUME command for this run (read docs/PAUSE_RESUME.md and AGENTS.md; use the same config and run name); how to check progress (python monitor_loss.py /content/ai-toolkit/output/pron_lora_ar_only_r8/loss_log.db and tail of the log); the expected checkpoint filenames. Do NOT state a wall-clock estimate up front: no A100 step time has been measured for these short clips (the L4 smoke number does not transfer, and the old whole-song 3.10 s/step A100 figure is for full songs). Instead tell the user to run monitor_loss.py after about 50 steps and report the measured step rate and ETA then.

4. AFTER THE RUN (when the user tells you it finished): verify the four checkpoints + final file exist locally, verify they exist in GCS under <base>/pron_lora_ar_only_r8/output/ (gcloud storage ls), report file sizes and the last logged loss/ar_ce and loss/ar_kl, and commit a short results note to docs/PRON_LORA_VERIFICATION.md (push). Do not do anything else (no merging, no inference, no sweep): those are the next session's tasks.

Write agent_notes/current.md with the current state and the exact next command (overwrite, per AGENTS.md; verify the write before saying so).
```

---

## Open for session 9 (not sent)

- **Task 15, merge tool** (main repo, same branch `pron-lora-ar-only`), with the alpha = 0 must reproduce v2 bit-for-bit invariant. Not yet written. Design questions to settle first:
  (1) Saved LoRA files use `text_encoders.*` (AR) and `diffusion_model.*` (NAR) key names, not `transformer.ar/nar.*` (verified in ai-toolkit `yue2_model.py convert_lora_weights_before_save`). v2's file has both prefixes; the pron file has only `text_encoders.*`.
  (2) Ranks differ (v2 = 32, pron = 8): raw A/B pairs cannot be summed; options are dense-delta materialization or rank-concatenation (exact sum, alpha = 0 leaves the v2 delta numerically unchanged). Base weights are quantized (int8 convrot), so a LoRA-file-level merge is probably the practical route. Unverified: check how the v2 LoRA reaches audio.cpp inference (`/content/converter/out/convert_aitoolkit_yue2_lora.py`, `docs/INFERENCE.md`) before deciding.
  (3) Define "bit-for-bit" precisely (file bytes vs. effective weight delta vs. converted GGUF).
- Then: alpha sweep on `INFERENCE/yue2_eval_heldout/` lyrics, letter-substitution scorecard; use the 0.25/0.5/0.75/1.0-epoch checkpoints as the stopping-rule data.
- Offline val-loss script (proposed by A3, not built): diffusion_trainer has no audio validation support.
