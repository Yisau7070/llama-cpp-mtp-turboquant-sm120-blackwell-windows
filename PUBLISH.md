# Publish workflow — step-by-step

This file is for you (the maintainer). Not part of the public release.

After local git init + commit (done automatically by the assistant), follow
these steps to publish to GitHub. Estimated total time: 10-15 minutes.

---

## Prerequisites

1. **GitHub account** — must exist before starting
2. **GitHub CLI (`gh`)** — install once:
   ```powershell
   choco install gh -y
   ```
3. **Authenticate** (one-time, interactive):
   ```powershell
   gh auth login
   ```
   - Choose: GitHub.com
   - HTTPS
   - Login via browser (opens default browser, follow prompts)

---

## Step 1: Create the GitHub repo

From PowerShell, in the `E:\AI-models\release-en` folder:

```powershell
cd E:\AI-models\release-en

gh repo create llama-cpp-mtp-turboquant-sm120-blackwell-windows `
  --public `
  --description "Windows prebuilt of llama.cpp combining Multi-Token Prediction (MTP) + TurboQuant KV cache compression + native sm_120 (Blackwell consumer GPU, FP4 tensor cores). For RTX 5060 Ti / 5070 / 5080 / 5090." `
  --source=. `
  --remote=origin
```

This creates the repo on GitHub AND sets `origin` to point at it.

---

## Step 2: Push the initial commit

```powershell
git push -u origin main
```

If your branch ended up named `master` (older git default), rename first:

```powershell
git branch -M main
git push -u origin main
```

---

## Step 3: Create the v1.0 Release with binary asset

```powershell
gh release create v1.0 `
  "llama-cpp-mtp-turboquant-sm120-v1.0-windows-x64.zip" `
  --title "v1.0 — Initial release (MTP + TurboQuant + native sm_120)" `
  --notes-file CHANGELOG.md
```

This:
- Creates tag `v1.0` on the current commit
- Creates a GitHub Release page for that tag
- Uploads the 589 MB zip as a release asset (downloadable on the release page)
- Uses `CHANGELOG.md` content as the release notes

Upload of 589 MB will take a few minutes depending on your upload bandwidth.

---

## Step 4 (optional): Set repo topics for discoverability

```powershell
gh repo edit --add-topic llama-cpp,turboquant,mtp,blackwell,sm-120,rtx-50,rtx-5060ti,rtx-5090,cuda-12-8,prebuilt,windows
```

Topics make the repo appear in GitHub topic search pages — useful for community discovery.

---

## Step 5 (optional): Community visibility

After the repo is live, post in places where the target audience lives:

1. **Reddit r/LocalLLaMA** — short post explaining the gap this fills:
   - TheTom prebuilt gives ~19 t/s on Blackwell due to stuck FORCE_CUBLAS
   - Upstream lacks TurboQuant
   - AmesianX has sm_120 but no MTP
   - NJannasch has both but no Windows release
   - This: filling the gap, RTX 5060 Ti 47 t/s with 256K context
2. **HuggingFace Discussion** — on `unsloth/Qwen3.6-27B-MTP-GGUF` discussions, note the speedup with this build
3. **Issue #22867 / Discussion #20969** on `ggml-org/llama.cpp` — relevant threads where TurboQuant + MTP community gathers

For each post: link to the GitHub repo, mention the 5060 Ti / RTX 50 hardware
benchmark numbers, and the `BUILD-NOTES.md` for those who want to compile
themselves.

---

## After publication

If you make changes (e.g. rebuild against newer NJannasch commit, or fix
issue #22867 once it's resolved upstream):

1. Update `CHANGELOG.md` with the new version section
2. `git commit -am "v1.1: <what changed>"`
3. `git push`
4. Rebuild + zip + `gh release create v1.1 ...`

For text-only doc updates (typo fixes etc), just `git push` — no new release
needed.

---

## Things to NOT do

- Don't commit the zip into git — it's already in `.gitignore`. Releases handle binaries.
- Don't push API keys or paths from your local machine into committed text files.
- Don't take down v1.0 if you publish v1.1 — users may link to the v1.0 download URL. Keep old releases around (you can mark them as "pre-release" or "outdated" in the GitHub UI if needed).
