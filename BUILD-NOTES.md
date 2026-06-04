# Build notes — llama.cpp + MTP + TurboQuant for Windows / Blackwell sm_120

**For:** RTX 50-series consumer GPUs (5060 Ti, 5070, 5070 Ti, 5080, 5090) on Windows 11.

This document is a complete reproducible build recipe — anyone with the same hardware can rebuild from scratch and verify the same results.

If you just want the prebuilt binary without compiling — see `README.md` and the release zip.

---

## What this build provides

A llama.cpp Windows x64 binary set with:

- **MTP (Multi-Token Prediction)** speculative decoding via `--spec-type draft-mtp` for models shipping MTP heads (e.g. Unsloth Qwen3.6 MTP GGUFs)
- **TurboQuant KV cache** (`turbo2/turbo3/turbo4`) — Google DeepMind's ICLR 2026 algorithm, up to 6.4× compression
- **Native sm_120 + FP4 tensor cores** via `CMAKE_CUDA_ARCHITECTURES=120a-real` (no JIT-PTX fallback)
- All cross-cache-type combinations available

Combined, this enables Qwen3.6-27B at **256K context on 16 GB VRAM** at ~47 t/s decode. Compare to TheTom's `tqp-v0.1.1` prebuilt which delivers only ~19 t/s on the same hardware due to a stuck `FORCE_CUBLAS=ON` in CMake cache.

---

## Hardware requirements

| Component | Minimum | Recommended |
|---|---|---|
| GPU | RTX 50-series Blackwell (sm_120) | RTX 5060 Ti 16GB or better |
| RAM | 32 GB DDR5 | 32-64 GB |
| Free disk | ~25 GB for tools + source | NVMe SSD for build performance |
| OS | Windows 10/11 x64 | Windows 11 |
| NVIDIA Driver | R555 (for CUDA 13.x runtime) | R596 or newer |

---

## Software prerequisites

| Tool | Version used | Why |
|---|---|---|
| CUDA Toolkit | **12.8.61** | Minimum for native `sm_120` (`120a-real` architecture). **12.4 is NOT enough** (toolkit doesn't know sm_120). 12.9+ also fine. 13.x not tested with TurboQuant kernels — likely incompatible. |
| Visual Studio Build Tools 2022 | **17.14.33** with Desktop C++ workload | Provides MSVC `cl.exe` 14.44+ and the Windows 11 SDK |
| Ninja | 1.13.2 | Fast multi-config build generator |
| CMake | 4.2.1 | Build configuration |
| Git | 2.37+ | Source clone |

### Optional: custom install paths

To save space on C:, both VS Build Tools and CUDA Toolkit support custom install locations:

**VS Build Tools** GUI installer → "Installation locations" tab → set Product to e.g. `E:\BuildTools\VS2022`. Note: ~2 GB of Windows SDK still goes to C: (Microsoft-mandated, can't be changed).

**CUDA Toolkit** GUI installer → Custom (Advanced) → expand CUDA components → set install path to e.g. `E:\NVIDIA\CUDA\v12.8`. Uncheck Documentation, Samples, Nsight Compute/Systems/VSE to save ~1.5 GB. Always uncheck Driver components — don't downgrade your existing driver.

---

## Source code

Clone the NJannasch fork on the `mtp-turboquant` branch:

```cmd
git clone --branch mtp-turboquant --single-branch ^
  https://github.com/NJannasch/llama.cpp.git ^
  src-njannasch
```

This is **upstream llama.cpp's `am17an/mtp-clean` branch** (PR #22673 = MTP integration) with one cherry-picked patch from `TheTom/llama-cpp-turboquant` (commit `d1cfe5766`, 113 files changed, +24642 lines) that adds TurboQuant KV cache types.

Latest commit on the branch as of writing: `d1cfe5766` — "Add TurboQuant KV cache compression (turbo2/turbo3/turbo4)", May 14, 2026.

---

## Build commands

### `configure.bat`

```cmd
@echo off
call "E:\BuildTools\VS2022\VC\Auxiliary\Build\vcvars64.bat"
if errorlevel 1 (
  echo Failed to activate MSVC environment
  exit /b 1
)

REM Add CUDA to PATH (vcvars does not inherit system PATH fully)
set "CUDA_PATH=E:\NVIDIA\CUDA\v12.8"
set "CUDAToolkit_ROOT=E:\NVIDIA\CUDA\v12.8"
set "PATH=%CUDA_PATH%\bin;%CUDA_PATH%\libnvvp;%PATH%"

cd /d <path_to_src-njannasch>

REM CRITICAL: clean build dir to avoid sticky CMakeCache.txt values like FORCE_CUBLAS=ON
if exist build rmdir /S /Q build

cmake -S . -B build -G "Ninja Multi-Config" ^
  -DCUDAToolkit_ROOT="E:/NVIDIA/CUDA/v12.8" ^
  -DGGML_CUDA=ON ^
  -DGGML_CUDA_FA_ALL_QUANTS=ON ^
  -DGGML_NATIVE=ON ^
  -DGGML_CUDA_FORCE_MMQ=ON ^
  -DGGML_CCACHE=OFF ^
  -DLLAMA_BUILD_SERVER=ON ^
  -DLLAMA_BUILD_TOOLS=ON ^
  -DLLAMA_BUILD_EXAMPLES=OFF ^
  -DLLAMA_BUILD_TESTS=OFF
```

### `build.bat`

```cmd
@echo off
call "E:\BuildTools\VS2022\VC\Auxiliary\Build\vcvars64.bat"
cd /d <path_to_src-njannasch>
cmake --build build --config Release -j 11
```

(`-j 11` = number_of_logical_threads − 1, adjust for your CPU)

### Expected output

From `configure.bat`:

```
-- Replacing 120-real in CMAKE_CUDA_ARCHITECTURES_NATIVE with 120a-real
-- Using CMAKE_CUDA_ARCHITECTURES=120a-real
-- Found CUDAToolkit: ... (found version "12.8.61")
-- CUDA Toolkit found
-- ggml commit: d1cfe5766
-- Configuring done
```

If you see `120-real` instead of `120a-real`, your build is **missing FP4 tensor core support** — check that you cleaned `build/` and didn't override `CMAKE_CUDA_ARCHITECTURES` manually.

From `build.bat`:

```
[520/520] Linking CXX executable bin\Release\llama-server.exe
=== Build exit code: 0 ===
```

Wall time on Ryzen 5 9600X (6c/12t): **~30 minutes**. Higher-core CPUs will scale near-linearly with `-j N`.

---

## Deploy

After successful build, deploy to a clean directory:

```cmd
mkdir E:\AI-models\llm-server-njannasch
copy /Y src-njannasch\build\bin\Release\* E:\AI-models\llm-server-njannasch\
copy /Y E:\NVIDIA\CUDA\v12.8\bin\cudart64_*.dll E:\AI-models\llm-server-njannasch\
copy /Y E:\NVIDIA\CUDA\v12.8\bin\cublas64_*.dll E:\AI-models\llm-server-njannasch\
copy /Y E:\NVIDIA\CUDA\v12.8\bin\cublasLt64_*.dll E:\AI-models\llm-server-njannasch\
```

This makes the deploy folder fully self-contained — users do not need a separate CUDA Toolkit install.

---

## Verification (smoke test)

```cmd
cd E:\AI-models\llm-server-njannasch
.\llama-bench.exe ^
  -m path\to\Qwen3.6-27B-UD-IQ3_XXS.gguf ^
  -ngl 999 -fa 1 -ctk turbo3 -ctv turbo3
```

Expected on RTX 5060 Ti 16GB with **Qwen3.6-27B-UD-IQ3_XXS** (Unsloth dynamic 3.0625 bpw, MTP variant):

```
| qwen35 27B IQ3_XXS - 3.0625 bpw | CUDA | ngl 999 | type_k turbo3 | type_v turbo3 | fa 1 | pp512 | ~918 t/s |
| qwen35 27B IQ3_XXS - 3.0625 bpw | CUDA | ngl 999 | type_k turbo3 | type_v turbo3 | fa 1 | tg128 | ~31 t/s |
```

If decode is ~31 t/s — build is healthy. If you see ~15-20 t/s — your binary likely got `FORCE_CUBLAS=ON` stuck in CMake cache, disabling MMQ kernels. Clean `build/` directory and reconfigure from scratch.

---

## Run with MTP for maximum decode speed

```cmd
.\llama-server.exe ^
  -m path\to\Qwen3.6-27B-UD-IQ3_XXS.gguf ^
  --ctx-size 262144 ^
  --n-gpu-layers 999 ^
  --flash-attn on ^
  --cache-type-k turbo3 --cache-type-v turbo3 ^
  --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-p-min 0.75 ^
  --threads 5 --batch-size 1024 --ubatch-size 512 ^
  --jinja ^
  --host 127.0.0.1 --port 8080
```

Expected: ~47 t/s decode on short context, ~30-40 t/s at depth 100K-200K, 256K context loads in ~14.7 GB of VRAM (very tight on a 16 GB card — ~430 MB free margin).

**Note 1:** `--spec-draft-n-max 2` is optimal for `turbo3` cache because TurboQuant dequantization on each MTP acceptance check is more expensive than on `q8_0`. With `q8_0` cache, `n_max=5` would be more efficient.

**Note 2:** `--mmproj` (vision) is currently incompatible with `--spec-type draft-mtp` due to llama.cpp issue #22867 (slot position corruption). For vision use a separate server invocation without `--spec-type`.

---

## Pitfalls and gotchas

### `FORCE_CUBLAS=ON` stuck in CMakeCache

The single most common reason for slow Blackwell inference (~50% performance loss). Symptom: decode t/s about half of what it should be.

Cause: in the early Blackwell support era (before llama.cpp `b8157`), forks added `-DGGML_CUDA_FORCE_CUBLAS=ON` as a workaround for MMQ kernel crashes. That flag **persists in CMakeCache.txt** and disables the MMQ kernel on Blackwell where it is now the primary path. `cmake -B build` does NOT overwrite cached values.

**Fix:** delete `build/` directory before reconfigure. Our `configure.bat` does this automatically.

### `llama-memory-recurrent.cpp:173` assert without `--spec-type`

```
GGML_ASSERT(rollback >= 1 && rollback <= (llama_pos) n_rs_seq) failed
```

Triggers when running NJannasch fork without `--spec-type draft-mtp` at non-trivial `-c` values. Workaround: always run with MTP activated when context is large. For vision-only tasks (which can't use MTP), use upstream llama.cpp prebuilt instead — it does not have this assert.

### `--mmproj` + `--spec-type draft-mtp` crashes

Tracked as [llama.cpp issue #22867](https://github.com/ggml-org/llama.cpp/issues/22867). Not specific to this fork — affects all builds with both features active. Workaround: run two separate server processes if you need both vision and MTP.

### CUDA 13.x compatibility

CUDA 13.0-13.2 produces gibberish output on Blackwell consumer GPUs (Unsloth issue #4849). CUDA 13.3 fixes that for upstream llama.cpp. But TurboQuant code in this fork is **not tested on CUDA 13.x** — stay on CUDA 12.8 or 12.9 for safety.

### CUDA 12.4 lacks `sm_120` knowledge

If you accidentally build with CUDA Toolkit 12.4 — nvcc will not include `120a-real` in the architecture list. The build will use older arches (89 / 90) with JIT-PTX fallback at runtime. Use 12.8+ for proper native compilation.

---

## Credits

This build is a composition of work from several MIT-licensed projects:

- **ggml-org/llama.cpp** — upstream inference engine
- **am17an/llama.cpp `mtp-clean` branch** — PR #22673 MTP integration
- **TheTom/llama-cpp-turboquant** — original TurboQuant kernel port for llama.cpp
- **NJannasch/llama.cpp `mtp-turboquant` branch** — cherry-pick combining MTP and TurboQuant
- **Google DeepMind** — original TurboQuant algorithm (Zandieh et al., ICLR 2026, "Walsh-Hadamard-rotated polar codebook quantization")

See `LICENSE` for full text.

This build is provided AS-IS with no warranty and no support guarantee.

---

## Step-by-step TL;DR

1. Install Windows 11 + NVIDIA Driver R555 or newer
2. `choco install ninja git -y`
3. Download and install **VS Build Tools 2022** with Desktop C++ workload (optionally set install path to `E:\BuildTools\VS2022`)
4. Download and install **CUDA Toolkit 12.8** Custom mode (uncheck Driver, Documentation, Samples, Nsight; optionally set toolkit path to `E:\NVIDIA\CUDA\v12.8`)
5. Reboot
6. `git clone --branch mtp-turboquant --single-branch https://github.com/NJannasch/llama.cpp.git src-njannasch`
7. Save the `configure.bat` and `build.bat` from this document (adjust paths if you used different install locations)
8. Run `configure.bat` — should complete in ~30 seconds, must report `CMAKE_CUDA_ARCHITECTURES=120a-real`
9. Run `build.bat` — ~30 minutes on Ryzen 5 9600X
10. Deploy: copy binaries + CUDA DLLs as shown in the Deploy section
11. Smoke test with `llama-bench` and verify decode ~31 t/s on Qwen3.6-27B
12. Done — use with any GGUF model. For MTP speedup use models with MTP heads (e.g. Unsloth's MTP-GGUFs)

Total time investment from scratch: ~2-3 hours including downloads and build.
