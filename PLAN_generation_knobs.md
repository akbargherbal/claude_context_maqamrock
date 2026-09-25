# Plan — generation-knobs & checkpoint follow-through

**Purpose:** the modular breakdown of the work opened by
`docs/investigation_generation_knobs.md` (main repo, branch
`pron-lora-knobs-investigation`, commit `6478a43`, verified accurate as of
session 16 — see `claude_context.md`). That report is now **frozen**
(docs-reconciler excluded it as an external-citing research report); it is
not touched again by this plan. All *new* work below lands on
`pron-lora-ar-only`, same as every prior task in this project.

**Why this file exists:** each step below is a self-contained unit — question,
dependency state, exact copy-pasteable dispatch prompt, and a done-when/
decision rule. A session can stop after any one step and the next session
(agent or human) can resume from that step alone, without re-reading the
others. Steps are not required to happen in separate WebUI sessions — the
coding agent may run several in one sitting if it has the time/GPU budget;
the modularity is about *dependency correctness*, not session boundaries.

**How to use this file each session:**
1. Check `claude_context.md`'s "Where things stand" for which step is
   current.
2. Open that step's section here, note its **Compute** line before doing
   anything else — it decides whether a GPU box needs to be spun up at all.
3. Copy the dispatch prompt verbatim (fill in the one bracketed value if the
   step says to), send it to the coding agent.
4. When the agent reports a commit hash, pull `pron-lora-ar-only`, verify
   against the step's "Done when" criteria, then update `claude_context.md`
   (not this file — this file is the stable plan, `claude_context.md` is the
   live status) with the outcome and which step is next.
5. Do not skip a step whose "Depends on" is unresolved.

**Compute policy — GPU is scarce/costed, CPU is not, but don't over-optimize:**
the project has a free-standing CPU box (52 GB RAM / 8 vCPU — enough for
tensor concat/reshape, numpy/pandas analysis, text/tokenizer inspection, log
parsing) plus a paid GPU session (Colab T4/L4) reserved for `audio.cpp`
**generation** runs. The split that actually matters is at the **step**
level, not within a step:
- **Steps 0 and 4 never need the GPU at all** — dispatch those on the CPU
  box (or even this sandbox) and skip provisioning a GPU session entirely.
- **Steps 1-3 are GPU steps, full stop.** Each has a small CPU-shaped piece
  inside it (a merge, a shell-script edit) that's technically CPU-only, but
  it's minutes of work immediately followed by a GPU render — splitting
  that off to a separate CPU session to save a few minutes of idle GPU time
  costs more in context-switching (re-provisioning, re-cloning, re-verifying
  state) than it saves. Provision the GPU box once per step and let the
  agent do the whole thing there.
Don't chase savings finer than this — one provisioning decision per step is
the right grain.

---

## Step map (dependency order)

```
Step 0 (bake)                    [CPU only] ──► ships independently, anytime
Step 1 (checkpoint 1525/4575)    [GPU required] ──► Step 2
Step 2 (sampling knob probe)     [GPU required] ── re-anchored on Step 1
Step 3 (Kurd lyric-swap)         [GPU required] ──► fully decoupled
Step 4 (في ذمة الله lyric text)  [CPU only]     ──► fully decoupled, low priority
```

Only Step 2 has a real dependency (on Step 1's outcome, not merely on Step 1
having *run*). Steps 0, 3, 4 can be dispatched in any order, any session,
before/after/interleaved with 1 and 2. **Steps 0 and 4 never need the GPU box
at all** — dispatch those on a plain CPU runtime (or even this sandbox) and
save the GPU session for Steps 1-3.

---

## Step 0 — Bake the production merge

**Question this answers:** ship the already-decided result (session 15) as
the production artifact. Not an open question — no new evidence needed.

**Depends on:** nothing. Unblocked since session 15.

**Compute: CPU only.** `merge_pron_lora.py` loads safetensors and does
`torch.cat`/scalar-multiply on LoRA A/B tensors — no model forward pass, no
`audiocpp_cli` invocation, nothing that touches a GPU. Runs fine on the
52 GB/8-vCPU box (or this sandbox). **Do not spin up a GPU session for this
step.**

**Dispatch prompt:**

```
Repo: maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only.

Bake two production merges with merge_pron_lora.py, checkpoint 3050:
  - c3050_a0.5 (primary), alpha=0.5
  - c3050_a0.3 (fallback), alpha=0.3

Follow the existing merge + sidecar conventions (DECISIONS.md's sidecar
policy — full command, option values, adapter sha256, checkpoint step,
audio.cpp commit — one sidecar per output file). Do not regenerate audio;
this is a merge-only task, no GPU inference run. Commit the merged
safetensors (or a manifest + regeneration script if the files are too
large to commit — match whatever convention results/pron_sweep/ already
uses for large binaries) to pron-lora-ar-only. Report back: commit hash,
files added, both output sha256 hashes.
```

**Done when:** two merged adapter files (or their manifest) exist on
`pron-lora-ar-only` with sidecars, sha256 recorded.

**Decision rule:** none — just confirm and mark shipped in
`claude_context.md`.

---

## Step 1 — Checkpoint axis: render 1525 and 4575

**Question this answers:** is `step 3050` actually the best pron-LoRA
checkpoint, or was it never compared against the low-drift (1525) and
high-drift (4575) alternatives? (`investigation_generation_knobs.md` §5:
1525/4575 have **never been rendered**, only had `ar_ce`/`ar_kl` measured.)

**Depends on:** nothing technically — this can run before or after Step 0.
Sequenced first among the two open-question steps because it is cheaper and
more evidence-grounded (tests the repo's own named risk, `ar_kl` drift)
than the sampling probe, and its outcome changes what Step 2 should anchor
to.

**Compute: GPU required — whole step, one provisioning.** This step renders
actual audio (4 tracks) via `audiocpp_cli` — full AR+NAR inference. The merge
step (folding 1525/4575 into the adapter) is technically CPU-cheap on its
own, but it's a ~seconds-long precursor to a GPU render that follows
immediately after — provision the GPU box once for the whole step rather
than splitting the merge onto CPU first.

**Dispatch prompt:**

```
Repo: maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only.

Merge and render pron-LoRA checkpoints 1525 and 4575 at alpha=0.5 (the
already-shipped alpha), same Hijaz + Kurd maqams, same seed (20260924) and
auto cap (7250/6500) as results/pron_fine_sweep/. Reuse that sweep's staged
prompts. No run_one.sh changes needed — LORA_AR/LORA_NAR env overrides
already exist (Task 17). Strictly sequential renders (concurrent runs OOM
the NAR graph, docs/PRON_LORA_SWEEP.md).

4 tracks total: c1525_a0.5 x {Hijaz, Kurd}, c4575_a0.5 x {Hijaz, Kurd}.
Full sidecar per track (DECISIONS.md policy). Blind the filenames the same
way results/pron_fine_sweep/ was blinded (new random labels, key file kept
separate) so the human listener isn't biased. Commit tracks + sidecars +
blinding key to pron-lora-ar-only. Report back: commit hash, files added,
per-track sidecar summary (checkpoint, config, wall time).
```

**Done when:** 4 blinded tracks + sidecars + key committed; you (human)
have listened and scored each on the existing 1-5 + bleed-flag protocol
(`PRON_FINE_SWEEP_INPUT/`'s format) against the already-known `c3050_a0.5`
baseline scores.

**Decision rule (fill into `claude_context.md` after listening):**
- If 1525 or 4575 beats `c3050_a0.5`'s score with equal-or-fewer bleed
  flags → that checkpoint becomes the new anchor for Step 2, and for the
  production bake in Step 0 if not already shipped.
- If 3050 still wins → Step 2 proceeds anchored at 3050 as originally
  designed. No re-bake needed.
- If results are ambiguous (small n, no clear winner) → note it as
  inconclusive per the "n=1 track is noise" lesson, keep 3050 as the
  working anchor, and record the open question rather than forcing a call.

---

## Step 2 — Sampling knob probe (guidance_scale, temperature, repetition_penalty)

**Question this answers:** do any of the three cheapest, never-tried
sampling knobs move pronunciation or bleed at all?
(`investigation_generation_knobs.md`'s TL;DR + "Proposed minimal test
design".)

**Depends on:** Step 1's decision rule. **Before dispatching, read
`claude_context.md` for which checkpoint Step 1 settled on** and substitute
it for `<ANCHOR_CHECKPOINT>` and `<ANCHOR_ALPHA>` below (default
`3050`/`0.5` if Step 1 hasn't run yet or was inconclusive).

**Compute: GPU required — do the whole step on the GPU box, don't pre-stage
the script edit separately.** The `run_one.sh` diff and `pron_knob_probe.sh`
driver take the agent a couple minutes either way; splitting that off to a
CPU session first just to save a few minutes of idle GPU time isn't worth
the context-switch overhead (re-provisioning, re-cloning, re-verifying
state). Provision the GPU box once and let the agent write the script and
render all 8 tracks in the same session.

**Dispatch prompt:**

```
Repo: maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only.

First, apply this diff to INFERENCE/run_one.sh (adds an EXTRA_REQUEST_OPTS
env hook, additive only, no existing behavior changed):

--- a/INFERENCE/run_one.sh
+++ b/INFERENCE/run_one.sh
@@
 status="$OUT/_runs_status.log"
+
+# Extra request options, space-separated key=value pairs (e.g.
+# EXTRA_REQUEST_OPTS="guidance_scale=1.5 semantic_temperature=0.8").
+extra_request=()
+for kv in ${EXTRA_REQUEST_OPTS:-}; do
+  extra_request+=(--request-option "$kv")
+done
@@
   --request-option style="$(cat "$style")" \
   --request-option cot=off \
   --request-option semantic_max_tokens="$CAP" \
+  "${extra_request[@]}" \
   --seed "$S" \

Then write INFERENCE/pron_knob_probe.sh mirroring pron_fine_sweep.sh
(resumable on WAV + "Exit status: 0", failures to _failed.log, one folder
per config, full sidecar with sha256 of adapters/prompts/binary).

Render 8 tracks at checkpoint <ANCHOR_CHECKPOINT>, alpha=<ANCHOR_ALPHA>,
Hijaz + Kurd, same seed/cap/prompts as the fine sweep, one knob changed
per config from the anchor:
  - g1.0: guidance_scale=1.0
  - g1.5: guidance_scale=1.5
  - t0.8: semantic_temperature=0.8
  - rp1.4: semantic_repetition_penalty=1.4

Strictly sequential. Blind the filenames (new seed/labels, key kept
separate), matching the fine sweep's blinding convention. Commit the
run_one.sh diff, pron_knob_probe.sh, tracks, sidecars, and blinding key to
pron-lora-ar-only. Report back: commit hash, files added, per-config
sidecar summary.
```

**Done when:** `run_one.sh` diff + driver script + 8 blinded tracks +
sidecars + key committed; human has scored each against the reused
`c<ANCHOR>_a<ANCHOR>` reference on the same 1-5 + bleed-flag protocol.

**Decision rule:** per the report's "Verdict protocol" — decide *does this
knob move the needle at all* before picking a best value; do not call a
winner from 2 tracks per knob. If a knob shows a clear, replicated
direction, it becomes a candidate for the next production bake; log it as
an open thread rather than shipping on n=2.

---

## Step 3 — Kurd lyric-swap test

**Question this answers:** is Kurd's persistent underperformance
maqam-specific, or an artifact of the one held-out Kurd song?
(`claude_context.md` "Open threads".)

**Depends on:** nothing. Fully decoupled from Steps 0-2.

**Compute: GPU required.** 4 tracks of `audiocpp_cli` inference (alpha=0
and c3050_a0.5, swapped lyrics) — same as any other render step.

**Dispatch prompt:**

```
Repo: maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only.

Generate 4 tracks, seed and cap matching the existing fine-sweep
conventions, swapping held-out lyrics across maqams:
  - Kurd's caption+maqam tag with the Hijaz held-out lyric, at alpha=0
    and at c3050_a0.5
  - Hijaz's caption+maqam tag with the Kurd held-out lyric, at alpha=0
    and at c3050_a0.5

No run_one.sh or tooling changes needed (style/lyrics files already
overridable). Full sidecar per track. Blind filenames, key kept separate.
Commit tracks + sidecars + key to pron-lora-ar-only. Report back: commit
hash, files added.
```

**Done when:** 4 blinded tracks + sidecars + key committed; human has
listened and compared each against the existing Kurd/Hijaz baseline
scores.

**Decision rule:** if Kurd stays weak even with the Hijaz (easy) lyric →
maqam-specific weakness, worth its own investigation. If the weakness
follows the lyric instead → it's a test-song artifact, drop it as a
concern and stop treating Kurd's low scores as a maqam signal in future
sweeps.

---

## Step 4 — `في ذمة الله` lyric-text check

**Question this answers:** why does this specific line come out wrong in
every non-collapsed config, including the untouched baseline?

**Depends on:** nothing. Fully decoupled, low priority, and notably **not
a GPU task** — this is a source-text/tokenization investigation, not a
generation run.

**Compute: CPU only.** Reading the lyric source file, checking Unicode
normalization/diacritics, grepping the corpus for the same phrase, and even
running the yue2 text tokenizer in isolation (`Yue2TextTokenizer::encode`)
are all CPU operations — no model weights loaded, no `audiocpp_cli`
inference. **One caveat, not a blocker:** if the investigation wants to
actually invoke the compiled `audiocpp_cli` binary to see tokenizer output
(rather than reading tokenizer source/vocab directly), that binary was built
CUDA-linked, so it may only run on a CUDA-capable box even for a
tokenizer-only call. Prefer reading the tokenizer source/vocab file directly
on CPU first; only fall back to invoking the binary (and only then consider
whether that needs the GPU box) if source-reading doesn't resolve it.

**Dispatch prompt:**

```
Repo: maqamrock-yue2-lora-finetuning, branch pron-lora-ar-only.

Investigate why the lyric line "في ذمة الله" renders incorrectly across
every tested config, including alpha=0 (untouched v2 baseline) — this
predates the pron-LoRA and is not an alpha/checkpoint issue. This is a
CPU-only investigation — no audio generation, no GPU session needed. Check:
the source lyric text's encoding/diacritics in whatever file holds it, how
the yue2 text tokenizer handles it (read the tokenizer source/vocab
directly rather than invoking the CUDA-linked audiocpp_cli binary, unless
that's genuinely insufficient), and whether the same phrase or its
individual words render correctly elsewhere in the corpus. Propose a fix
(corrected source text, tokenization workaround) but do not apply it
without confirmation. Report findings inline in chat or as a short doc —
your call given the scope.
```

**Done when:** a diagnosis exists (even if "root cause unclear, here's what
was ruled out") — this does not require a merge, render, or listening
pass, so "done" here means a written finding, not a track.

**Decision rule:** if a clear text/encoding bug is found, decide whether to
fix the source lyric file (separate from any pron-LoRA work) or leave it
as a known limitation.

---

## What NOT to do (guardrails carried over from the investigation)

- Do not test alpha ≥ 0.55 again (session 15 finding, closed).
- Do not treat a single-track score delta as a peak (n=1 lesson).
- Do not run two GPU renders concurrently (NAR graph OOM).
- Do not re-open the frozen `pron-lora-knobs-investigation` branch/report —
  new findings from Steps 1-2 get written into `claude_context.md` or a new
  doc on `pron-lora-ar-only`, not back into the frozen file.
- Do not let Step 2 dispatch before checking `claude_context.md` for
  Step 1's outcome — the anchor checkpoint/alpha it substitutes in matters.
