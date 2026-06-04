# Changelog

## v1.0 — 2026-06-04 (initial release)

First public build combining MTP + TurboQuant + native sm_120 for Windows.

### Included

- llama.cpp source: NJannasch/llama.cpp `mtp-turboquant` branch, commit `d1cfe5766` (2026-05-14)
- Built with CUDA Toolkit 12.8.61
- Built with MSVC 14.44.35207 (Visual Studio Build Tools 2022 17.14.33)
- Built with `CMAKE_CUDA_ARCHITECTURES=120a-real` (native Blackwell consumer GPU + FP4 tensor cores)
- Built with `GGML_CUDA_FA_ALL_QUANTS=ON` (all KV cache type cross-combinations)
- Built with `GGML_CUDA_FORCE_MMQ=ON` (safe MMQ kernel selection)
- Bundles CUDA 12.8 runtime DLLs (cudart, cublas, cublasLt) for zero-install deployment

### Cache types available

`f16`, `bf16`, `q8_0`, `q4_0`, `q5_0`, `q5_1`, `q4_1`, `iq4_nl`, plus TurboQuant `turbo2`, `turbo3`, `turbo4` and all cross-combinations.

### Speculative decoding

- `--spec-type draft-mtp` (Multi-Token Prediction, for models with MTP heads)
- `--spec-type draft-simple` (standard draft model)
- `--spec-type draft-eagle3` (EAGLE-3)
- `--spec-type ngram-*` (n-gram variants)

### Verified on

- RTX 5060 Ti 16GB (sm_120) with Qwen3.6-27B-UD-IQ3_XXS:
  - Without MTP, turbo3 KV: 31 t/s decode at short ctx
  - With MTP n_max=2, turbo3 KV: 47 t/s decode, loads 256K context in ~15 GB VRAM

### Known issues

- `--mmproj` + `--spec-type draft-mtp` triggers llama.cpp issue #22867 (not specific to this build)
- `--spec-type none` + large `-c` triggers `llama-memory-recurrent.cpp:173` assert (specific to NJannasch fork)
- CUDA 13.x runtime not tested with TurboQuant kernels

### Files included

- `llama-server.exe` — HTTP server with OpenAI-compatible API
- `llama-cli.exe` — interactive CLI
- `llama-bench.exe` — benchmarking tool
- `llama-quantize.exe` — GGUF quantization tool
- `llama-imatrix.exe` — importance matrix calculation
- `llama-perplexity.exe` — perplexity evaluation
- `llama-tokenize.exe` — tokenizer test
- `llama-completion.exe` — one-shot completion
- Other tools from llama.cpp standard set
- `ggml-cuda.dll` — CUDA backend with TurboQuant kernels (~44 MB)
- `ggml-cpu.dll`, `ggml-base.dll`, `ggml.dll`, `llama.dll` — core libraries
- `cudart64_12.dll`, `cublas64_12.dll`, `cublasLt64_12.dll` — CUDA 12.8 runtime
- `mtmd.dll`, `llama-common.dll`, etc.

Total archive size: ~870 MB.

### Future versions

Will rebuild when:
- llama.cpp issue #22867 is fixed (mmproj + MTP compatibility)
- NJannasch rebases on newer upstream master
- CUDA 12.9+ becomes meaningfully different for our target
- Significant TurboQuant kernel improvements appear in any upstream
