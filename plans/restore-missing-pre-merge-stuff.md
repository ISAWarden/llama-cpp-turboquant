# Restore Missing Pre-Merge Behavior

## Goal

Restore pre-merge behavior that was lost or materially changed by commit `763379392`
relative to `c6e32b949`, with priority on regressions that affect `llama-cli`
quality when using TurboQuant KV cache.

Primary reproduction:

```bash
TURBO_AUTO_ASYMMETRIC=0 ./build/bin/llama-cli \
  -m "$HOME/Applications/MachineLearning/Models/Qwen3.6-35B-A3B-Uncensored-HauhauCS-Aggressive-IQ4_NL.gguf" \
  --mmproj /home/peter/Applications/MachineLearning/Models/Qwen3.6-35B-A3B-mmproj-bf16.gguf \
  -fa on -c 512000 \
  --spec-type ngram-mod \
  --seed 80085 --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.05 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  --jinja --ctx-checkpoints 16 --reasoning off \
  -ctk turbo4_0 -ctv turbo4_0 \
  --prompt "Write a story about a dragon named Rex"
```

Expected behavior should be comparable to pre-merge `c6e32b949`, where the model
keeps the requested name `Rex` and produces a stable story opening.

## Scope

Restore behavior only where the fork had deliberate pre-merge functionality.
Avoid broad upstream refactors while repairing these paths.

Do not commit automatically.

## Work Items

### 1. Restore `llama-cli` checkpoint cadence option

Regression:

- Pre-merge had `common_params::checkpoint_every_nt`.
- `--checkpoint-every-n-tokens` and `-cpent` were available for both
  `LLAMA_EXAMPLE_SERVER` and `LLAMA_EXAMPLE_CLI`.
- The merge replaced this with `checkpoint_min_step`, made
  `--checkpoint-every-n-tokens` an alias, and scoped it to server only.

Plan:

1. Reintroduce a CLI-visible `--checkpoint-every-n-tokens` / `-cpent` option.
2. Preserve the merged `--checkpoint-min-step` server option if needed.
3. Decide whether to use one shared field or separate fields:
   - `checkpoint_every_nt`: pre-merge checkpoint cadence during prefill.
   - `checkpoint_min_step`: merged server spacing guard.
4. Ensure `llama-cli --help` shows the restored option.
5. Verify the original command accepts `--checkpoint-every-n-tokens 1024`.

Acceptance:

- `./build/bin/llama-cli --help | rg "checkpoint-every|cpent"` shows the option.
- The original pre-merge command line parses without removing the checkpoint flag.

### 2. Restore TurboQuant KV inverse output rotation

Regression candidate:

- Pre-merge `src/llama-graph.cpp` applied `ggml_turbo_wht(..., 1, ...)` after
  attention when V cache type was turbo.
- Merged branch removed `GGML_OP_TURBO_WHT`, `ggml_turbo_wht`, and
  `llama_kv_cache_context::get_turbo_rotation*` hooks.
- `ggml-turbo-quant.c` still documents that turbo4 dequant leaves values in the
  rotated domain and that older graph paths applied inverse WHT to attention
  output.

Plan:

1. Compare pre-merge `ggml_turbo_wht` implementation and graph call sites against
   merged attention rotation infrastructure.
2. Choose the smallest restoration path:
   - restore `GGML_OP_TURBO_WHT` and graph call sites, or
   - adapt the merged `attn_rot_v` infrastructure to provide equivalent inverse
     output rotation for turbo V.
3. Cover both FA and non-FA paths in `build_attn_mha`.
4. Confirm MLA handling still uses the correct group source and output dimensions.

Acceptance:

- With `-fa on -ctv turbo4_0`, attention output is inverse-rotated before the
  downstream projection.
- A debug build or graph dump shows the inverse turbo/WHT operation in the V turbo
  path.
- Turbo V output no longer remains in rotated space at the model layer boundary.

### 3. Restore TurboQuant KV padding assumptions

Regression candidate:

- Pre-merge padded turbo K/V head dimensions to the next 128-wide WHT group.
- Merged branch allocates raw `n_embd_k_gqa` / `n_embd_v_gqa` dimensions.
- Pre-merge validation allowed turbo types to satisfy block-size constraints after
  padding.

Plan:

1. Restore pre-merge effective dimensions for turbo K/V cache tensors:
   - pad K per head to 128 when needed.
   - pad V per head to 128 when needed and not MLA.
2. Restore validation logic that accounts for turbo padding.
3. Ensure state read/write uses actual tensor widths where padding matters, not
   only unpadded hparams.
4. Verify no regression for already 128-aligned models.

Acceptance:

- Startup logs show turbo zero-padding when the model head dimension requires it.
- `-ctk turbo4_0 -ctv turbo4_0` does not reject models that pre-merge accepted.
- KV state serialization does not truncate padded turbo tensors.

### 4. Restore compatibility aliases for turbo cache type names

Regression:

- Pre-merge type names accepted `turbo2`, `turbo3`, `turbo4`.
- Merged branch names are `turbo2_0`, `turbo3_0`, `turbo4_0`.

Plan:

1. Keep canonical merged names if desired.
2. Add CLI parsing aliases so old commands keep working:
   - `turbo2` -> `GGML_TYPE_TURBO2_0`
   - `turbo3` -> `GGML_TYPE_TURBO3_0`
   - `turbo4` -> `GGML_TYPE_TURBO4_0`
3. Ensure help text lists both canonical names and aliases or documents aliases
   nearby.

Acceptance:

- Both `-ctk turbo4 -ctv turbo4` and `-ctk turbo4_0 -ctv turbo4_0` parse.
- Type logging remains clear about the actual enum used.

### 5. Isolate speculative decoding changes

Risk:

- `common/speculative.cpp` changed substantially.
- The reproduction uses `--spec-type ngram-mod`.
- Bad output could be affected by speculative accept/reject behavior even if KV is
  the primary suspect.

Plan:

1. Run the reproduction with `--spec-type none`.
2. Run the same seed with f16 KV and turbo4 KV.
3. If bad behavior only appears with ngram-mod, compare pre-merge and merged
   `common_speculative_impl_ngram_mod` logic.
4. Add a small deterministic speculative test if a concrete bug is found.

Acceptance:

- We can say whether the quality drop reproduces with speculation disabled.
- If speculation is implicated, fix it separately from turbo KV math.

### 6. Validate prompt formatting did not change

Current assessment:

- `tools/cli/cli.cpp` still applies Jinja chat formatting with
  `add_generation_prompt=true`.
- Both pre-merge and merged CLI paths force `COMMON_REASONING_FORMAT_DEEPSEEK`.
- Prompt formatting looks less suspicious than turbo KV math, but should be
  ruled out.

Plan:

1. Run the reproduction with `--verbose-prompt` on both pre-merge and merged code.
2. Diff the rendered prompt text and token IDs.
3. If different, identify whether `--reasoning off`, preserved tokens, or Jinja
   caps detection changed the generation prompt.

Acceptance:

- Rendered prompt and tokenization are identical or differences are understood.

## Validation Matrix

Run enough combinations to isolate the failing subsystem:

| KV type | Flash Attention | Speculative | Expected signal |
| --- | --- | --- | --- |
| f16/f16 | on | none | Baseline model behavior |
| f16/f16 | on | ngram-mod | Speculative-only effect |
| turbo4/turbo4 | on | none | Turbo KV math effect |
| turbo4/turbo4 | on | ngram-mod | Full reproduction |
| turbo4/turbo4 | off if supported | none | FA-specific turbo effect |

For each run, record:

- first 100 generated tokens,
- whether `Rex` is preserved,
- prompt t/s and generation t/s,
- startup logs for cache type, padding, attention rotation, and FA kernel choice.

## Suggested Implementation Order

1. Restore CLI parsing compatibility for `--checkpoint-every-n-tokens` and turbo
   aliases. These are low-risk user-facing regressions.
2. Restore or adapt turbo inverse output rotation.
3. Restore turbo KV padding and validation assumptions.
4. Run the validation matrix.
5. Only then inspect speculative decoding if the turbo fixes do not explain the
   quality gap.

## Files Likely Involved

- `common/arg.cpp`
- `common/common.h`
- `src/llama-graph.cpp`
- `src/llama-kv-cache.cpp`
- `src/llama-kv-cache.h`
- `src/llama-context.cpp`
- `ggml/include/ggml.h`
- `ggml/src/ggml.c`
- `ggml/src/ggml-cpu/ops.cpp`
- `ggml/src/ggml-turbo-quant.c`
- `common/speculative.cpp`

## Notes

- The output regression looks like degraded attention quality rather than a simple
  prompt template failure.
- The most important technical invariant to restore is: if turbo V is stored or
  dequantized in a rotated domain, the attention output must be transformed back
  before the rest of the layer consumes it.
- Keep upstream-compatible merged code where it does not conflict with the fork's
  TurboQuant behavior.
