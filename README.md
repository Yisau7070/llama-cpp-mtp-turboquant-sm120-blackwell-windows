# llama.cpp + MTP + TurboQuant — Windows Prebuilt for RTX 50-series (Blackwell sm_120)

**A Windows x64 build of llama.cpp combining upstream Multi-Token Prediction (MTP) with TurboQuant KV cache compression, compiled with native sm_120 (Blackwell consumer GPU) and FP4 tensor cores.**

This solves a specific gap as of June 2026: no public prebuilt of llama.cpp combines MTP + TurboQuant + native sm_120 for Windows. Existing builds either lack TurboQuant (upstream prebuilt), lack MTP integration (TheTom's `tqp-v0.1.1` from April), or target server GPUs (DGX Spark sm_121, RTX 5090 binaries from AmesianX), or simply ship without the `120a-real` architecture flag (resulting in JIT-PTX fallback). This build addresses all of those at once for consumer Blackwell.

## Hardware support

Native `sm_120 (120a-real)` build. Targets all RTX 50-series consumer GPUs:

| GPU | VRAM | Tested |
|---|---|---|
| RTX 5060 Ti 16GB | 16 GB | ✅ primary test platform |
| RTX 5060 Ti 8GB | 8 GB | — should work, low VRAM |
| RTX 5070 | 12 GB | — should work |
| RTX 5070 Ti | 16 GB | — should work |
| RTX 5080 | 16 GB | — should work |
| RTX 5090 | 32 GB | — should work, plenty of headroom |

Older NVIDIA cards (Ampere, Hopper, Ada) — not targeted by this build. Use upstream prebuilts.

## What's included

- All llama.cpp tools (`llama-server`, `llama-cli`, `llama-bench`, `llama-quantize`, etc.)
- `ggml-cuda.dll` with TurboQuant kernels (turbo2/turbo3/turbo4)
- CUDA 12.8 runtime DLLs (cudart, cublas, cublasLt) — self-contained, no separate install
- MTP speculative decoding (`--spec-type draft-mtp`) from upstream PR #22673
- All cross-combinations of cache types: turbo2/3/4 × q8_0 × f16

## Software prerequisites

- Windows 10 / 11 x64
- NVIDIA Driver **R555 or newer** (R596+ recommended for CUDA 13.x runtime, but this build uses CUDA 12.8 runtime which works with R555+)
- No separate CUDA toolkit install needed — runtime DLLs are bundled

## Quick start

1. Download the zip from releases
2. Extract anywhere (e.g. `C:\llama-cpp-mtp-turboquant\`)
3. Download a GGUF model (recommended: [Unsloth's Qwen3.6-27B-MTP-GGUF](https://huggingface.co/unsloth/Qwen3.6-27B-MTP-GGUF) for full feature use)
4. Run from the extracted folder:

```cmd
.\llama-server.exe ^
  -m path\to\Qwen3.6-27B-UD-IQ3_XXS.gguf ^
  --ctx-size 262144 ^
  --n-gpu-layers 999 ^
  --flash-attn on ^
  --cache-type-k turbo3 --cache-type-v turbo3 ^
  --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-p-min 0.75 ^
  --host 127.0.0.1 --port 8080
```

This runs Qwen3.6-27B with 256K context, TurboQuant KV (~5× compression vs f16), and MTP speculative decoding, all in ~15 GB of VRAM.

## ✅ Verified — works out of the box (2026-06-05)

The release zip was downloaded from **this very GitHub release page**, extracted into a clean folder, and benchmarked end-to-end on RTX 5060 Ti 16GB with `Qwen3.6-27B-UD-IQ3_XXS.gguf` (Unsloth MTP variant), 256K context, turbo3 KV, MTP with `n_max=2`:

```
[ Prompt: 77.9 t/s | Generation: 45.1 t/s ]
build: b9150-d1cfe5766
```

- SHA256 of the downloaded asset matches the original local build
- All binaries and CUDA 12.8 runtime DLLs ship inside the zip — **no separate CUDA Toolkit install needed**
- Decode speed within 3% of the local source build (~46.7 t/s) — variance is normal run-to-run noise
- 256K context loads correctly, no asserts, no OOM

**No external dependencies.** Download → Extract → Run.

## Performance benchmarks (RTX 5060 Ti 16GB)

Model: `Qwen3.6-27B-UD-IQ3_XXS.gguf` (Unsloth dynamic 3.0625 bpw, MTP variant).

| Configuration | Prompt t/s | Decode t/s | Max context |
|---|---|---|---|
| Upstream b9495 prebuilt (no TurboQuant, no MTP) | 1032 | 34 | 128K (q8_0 KV) |
| Upstream b9495 + MTP (no TurboQuant) | 109 | 54 | 128K |
| This build (turbo3 + MTP n_max=2) | 97 | **47** | **256K** |
| This build (turbo3, no MTP, llama-bench) | 918 | 31 | 256K |

For reference: TheTom's `tqp-v0.1.1` prebuilt on the same hardware gives only **19 t/s decode** due to `GGML_CUDA_FORCE_CUBLAS=ON` stuck in the CMake cache. This build avoids that pitfall.

For 35B-A3B MoE models (e.g. `Qwen3.6-35B-A3B-MTP-GGUF` with `--n-cpu-moe`), the original patch author reported **125 t/s at 98K context** on the same RTX 5060 Ti.

## Cache type options

Recommended combinations for Qwen3.6-27B (head_dim=128) on 16 GB VRAM:

| `-ctk` / `-ctv` | Compression vs f16 | Quality loss | Typical use |
|---|---|---|---|
| `q8_0` / `q8_0` | 2× | minimal | Default for ≤128K context (no TurboQuant needed) |
| `turbo4` / `turbo4` | 3.8× | very small | Best decode speed of TurboQuant variants |
| `turbo3` / `turbo3` | 4.9× | small | Maximum context (256K on 16GB) |
| `turbo2` / `turbo2` | 6.4× | noticeable | Extreme compression, quality-sensitive |
| `turbo4` / `turbo3` | hybrid | small | K precision, V compression — good for attention quality |

## MTP tuning

`--spec-draft-n-max` controls how many tokens are drafted per cycle:

- `n_max=2` — best when paired with `turbo3` KV (dequant cost on acceptance check is heavy)
- `n_max=5` — best with `q8_0` KV (cheap dequant, more acceptance opportunity)
- `n_max=8+` — only with strong draft model; can be net-negative on low acceptance rate

`--spec-draft-p-min 0.75` is a sensible default (threshold for draft confidence).

## Known issues

### 1. `--mmproj` (vision) incompatible with `--spec-type draft-mtp`

Tracked as [llama.cpp issue #22867](https://github.com/ggml-org/llama.cpp/issues/22867). Slot position corruption and OOM during mmproj restore when both MTP and vision are active. For vision tasks, omit `--spec-type` — the model still works as a multimodal text+image model, just without the MTP speedup.

### 2. NJannasch `llama-memory-recurrent.cpp:173` assert without MTP

Running with `-c >small` and **no** `--spec-type draft-mtp` triggers `GGML_ASSERT(rollback >= 1 && rollback <= n_rs_seq)`. Workaround: always activate MTP when using long context with this build. For vision (which can't use MTP), use upstream llama.cpp prebuilt instead — it does not have this assert.

### 3. Quality degradation on 3-bit quants at long context

Independent research (arxiv 2505.02214, "An Empirical Study of Qwen3 Quantization") shows: «as bit-width decreases to 3 bits, most of the original model's advantages are lost», and «4-bit methods lead to substantial losses for long-context inputs (up to 59%)». If you use UD-IQ3_XXS at 200K+ context for retrieval-heavy tasks, expect occasional misses. Use UD-IQ4_XS or higher quant at 128K for better quality at the cost of context length.

### 4. VRAM is tight at 256K

256K context + turbo3 KV + MTP draft cache + compute buffers ≈ 14.7 GB on Qwen3.6-27B IQ3_XXS, leaving only ~430 MB headroom on a 16 GB card. If you see OOM during long-prompt prefill, try `--ubatch-size 256` instead of 512, or reduce ctx to 192K.

## Why this build exists

The combination of MTP + TurboQuant + native sm_120 on Windows is currently scattered across:

- **upstream llama.cpp** — has MTP (PR #22673 merged May 2026), but no TurboQuant
- **TheTom/llama-cpp-turboquant** — has TurboQuant, but `tqp-v0.1.1` (Apr 21, 2026) predates MTP merge, ships CUDA 12.4 build, has `FORCE_CUBLAS=ON` stuck in cache
- **AmesianX/TurboQuant** — has Windows sm_120 binaries (v1.7.0 with RTX5090 build), but no MTP integration
- **NJannasch/llama.cpp** branch `mtp-turboquant` — has both MTP and TurboQuant, but ships no binaries

This build is **NJannasch's source compiled with the correct CUDA + CMake flags for native consumer Blackwell**, with CUDA runtime DLLs bundled for zero-install deployment.

## License and credits

This build is composed of MIT-licensed work from multiple upstream projects. See `LICENSE` for the full text.

Credit chain:

- **ggml-org/llama.cpp** — upstream inference engine (MIT)
- **am17an/llama.cpp** `mtp-clean` branch — MTP integration (PR #22673 merged May 2026)
- **TheTom/llama-cpp-turboquant** — original TurboQuant kernel port (MIT)
- **NJannasch/llama.cpp** branch `mtp-turboquant` — combined MTP + TurboQuant fork
- **Google DeepMind** — TurboQuant algorithm (Zandieh et al., ICLR 2026, Walsh-Hadamard-rotated polar codebook quantization)

Build by an independent contributor as a community service. **AS-IS, no warranty, no support guarantees.** If something breaks, file an issue — but expect community-quality response, not enterprise support.

## Building from source

See `BUILD-NOTES.md` for the full reproducible build instructions if you want to compile yourself instead of using this prebuilt.

## Compatibility

This build is a snapshot from **NJannasch/llama.cpp `mtp-turboquant` branch, commit `d1cfe5766` (May 14, 2026)**. It does not include upstream llama.cpp changes merged after that date. If you need newer features (e.g. issue #22867 fix when it lands), you'll need to rebuild from a newer source.
