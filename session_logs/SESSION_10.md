In Flight Prompt to AI agent from Claude:

```
Task 16 — (A) re-verify Task 15's live invariant; (B) build + BENCHMARK ONLY the offline AR-loss replay script (main repo, branch pron-lora-ar-only)

## Part A — re-run Task 15's real end-to-end test live (do this first, it's cheap)

Session 10 built merge_pron_lora.py and reported (in docs/PRON_LORA_MERGE.md and the commit message) that the alpha=0 output, through the converter, reproduces the live converted v2 adapters byte-for-byte. That claim has not been independently re-run since — do it now, live, and paste real output.

1. Confirm whatever merge_pron_lora.py/convert_aitoolkit_yue2_lora.py need is staged (the real v2 + pron .safetensors, the converter script itself) — same inputs docs/PRON_LORA_MERGE.md already lists paths for.
2. Run: `pytest tests/test_merge_pron_lora.py -v -s` (the whole file, not just the one test) and paste the ACTUAL terminal output verbatim into your report and the commit message — not a paraphrase, not "all passed."
3. Confirm test_alpha_zero_reproduces_live_converted_v2 ran (not skipped) and passed. Read the two full sha256 hashes it compared against out of docs/PRON_LORA_MERGE.md yourself (don't retype from memory) and confirm the match explicitly in your report.
4. Do NOT touch merge_pron_lora.py itself unless this genuinely fails. If it fails, STOP — do not patch around it — report exactly which tensor/file differs and how, and treat the whole two-LoRA design as unverified until that's resolved.
5. Commit a short dated note to docs/PRON_LORA_MERGE.md's Verification section: "Re-verified live, session 11, <date>: pass/fail, both full sha256 hashes."

## Part B — build the offline AR-loss replay script, but STOP after a small timed benchmark — do not run the full sweep

Context: diffusion_trainer's validation path is image-only (docs/PRON_LORA_VERIFICATION.md, section A3) — there is no in-training validation on /content/pron_dataset/val. That doc already scopes the smallest correct fix: for each checkpoint, load the adapter via `convert_lora_weights_before_load` (yue2_model.py:795-802), run the exact per-item forward/loss path training used (yue2_model.py:630-704, `_ar_losses` at 506-574) over the 180 val pairs, report `loss/ar_ce` and `loss/ar_kl`. Reuse that real code path — do not reimplement the loss math yourself.

Spec
1. New script offline_ar_loss_replay.py (repo root) + tests. Forward passes only, no backward/optimizer.
2. Assets needed (forward-only, no trainer, no training dataset): base checkpoint `Comfy-Org/YuE2/checkpoints/yue2_3b_int8_convrot.safetensors`, MERT-v2 semantic head (`m-a-p/MERT-v2-FullSong`), tokenizer/head assets (`Mothersuperior/yue2-mothersuperior-realaudio-tokenizer-v4`) — same set `bootstrap/setup.sh --training` pre-warms into the HF cache (DECISIONS.md). Stage only what's needed for a forward pass; report exactly what you staged and from where, since this environment's layout may not match a training VM's.
3. The 180 val pairs live at /content/pron_dataset/val or wherever they're actually staged this session — locate them and report the real path used, don't assume.
4. Given one checkpoint (one of the 4 pron checkpoints or final), load base + adapter, iterate val pairs, compute per-item loss/ar_ce and loss/ar_kl via the real code path, report the mean per checkpoint.
5. CLI: `python offline_ar_loss_replay.py --checkpoint final --limit N` (`--limit` caps how many val pairs run, for a benchmark slice).
6. Do not assume CPU works for this — if any op in the forward path is CUDA-only and genuinely can't run on CPU, say so explicitly, report what specifically blocks it, and stop there rather than silently switching hardware or reimplementing around it.

BENCHMARK — the only run you're authorized to do in this task
7. Run once: `python offline_ar_loss_replay.py --checkpoint final --limit 8` on CPU. Report wall-clock time per item and total, and the actual loss/ar_ce, loss/ar_kl values for sanity (session 9 logged ar_ce ~4.16-4.40 and ar_kl ~1.53 at the final checkpoint during training — these should land in a plausible neighborhood, not wildly different or NaN; if they're wildly different, flag it as a possible bug rather than reporting it as a normal result).
8. Extrapolate explicitly: full job = 180 val pairs x 5 checkpoints (4 saved + final) = 900 forward passes. State it plainly: "measured X s/item -> ~Y hours total for all 900."
9. STOP HERE. Do not run the full 900-pass sweep. Do not compare checkpoints. Do not draw any conclusion about pronunciation quality. That decision (run the full 900 on this CPU box vs. wait for the L4 step) is the user's to make once they see the extrapolated time.

Verification (put all of this in the commit + report, numbers not prose-only)
- Part A: full pytest output, both hashes, match confirmed.
- Part B: what assets were staged and from where; the 8-item benchmark's per-item and total wall time; the extrapolated 900-pass estimate; the 8 items' actual loss/ar_ce and loss/ar_kl values.

Do NOT: run the full replay sweep, run any inference, start the merge sweep, touch Colab GPU pricing/runtime decisions, or make any judgment about whether pronunciation improved — all of that waits for the benchmark number to come back to the user.

Write agent_notes/current.md with current state and the exact next command (overwrite, per AGENTS.md; verify the write before saying so).
```

Results are pushed into github.