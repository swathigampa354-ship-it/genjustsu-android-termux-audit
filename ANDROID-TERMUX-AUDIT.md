# Genjustsu Open Source Workflow — Android/Termux Compatibility Audit

Audited repository: https://github.com/sirioberati/Genjustsu-Open-Source-Workflow
Audit date: 2026-11-20
Constraint: audit only (no modification of the original project), no root, zero budget.

## Device Summary

| Item | Value |
|---|---|
| Device | OPPO CPH2565 (A78 4G) |
| OS | Android 15 (API 35), build type `user`, `ro.debuggable=0` |
| SoC | Qualcomm SM6225 (`bengal`), 8 cores, arm64-v8a |
| GPU | Adreno (Vulkan/OpenCL only, no CUDA) |
| RAM | 7.4 GB total, ~2.0 GB available, 5.5 GB swap (2.7 GB used) |
| Storage | 10 GB free of 106 GB |
| Root | No — uid `u0_a1078`, SELinux `untrusted_app_27`; `su` present but unusable |
| Termux | 0.118.3, repos: main, tur, x11, glibc |
| Python | 3.14.8, pip 26.2.2, platform tag `android-24-arm64_v8a` |
| Thermal | zone0 58.7 °C at idle-ish baseline |

Key platform fact: Termux pip only accepts `android_24_arm64_v8a` tags. There is no manylinux/musllinux compatibility, so PyPI aarch64 Linux wheels cannot be installed, and PyPI has no Android wheels.

## Compatibility Table

| Component | Compatible? | Tested? | Modification needed? | Free? |
|---|---|---|---|---|
| FFmpeg (composite/encode) | Yes | Yes — 9.0.2 with `--enable-librubberband`, `--enable-mediacodec` | No | Yes |
| Rubber Band CLI (+3 semitones pitch) | Yes | Yes — 4.0.0-1 | No | Yes |
| Cloudflared webhook tunnel | Yes | Yes — 2026.10.6 | No | Yes |
| FastAPI + Uvicorn + static UI | Yes (pure wheels) | Import tested, server not run | Relax/extend `requirements.txt` | Yes |
| gradio_client (HF Spaces) | Yes (pure wheel 2.7.2) | pip dry-run OK | Add to requirements | Yes (network) |
| NumPy | Yes (2.5.4 via pkg) | Imported | Pin `2.4.3` not satisfiable — relax | Yes |
| Pillow | Yes (13.0.0 via pkg) | Imported | Pin `12.1.1` not satisfiable — relax | Yes |
| SciPy | Yes (1.18.1 via pkg) | Imported | No | Yes |
| OpenCV (`cv2` 5.1.0 via x11 pkg) | Yes | Imported | Package name differs from `opencv-python-headless` | Yes |
| PyAV (`av`) | **No** | No — no Termux package, no matching wheel | Replace with FFmpeg subprocess or build from source | Yes |
| MediaPipe face mesh | **No** | No — no Android/musllinux wheel, no Termux package | Rewrite with OpenCV/ONNX Runtime | Yes |
| PyTorch | Importable only | Imported 2.15.0, not exercised | Unverified for this workload | Yes |
| ONNX Runtime | Importable only | Imported 1.31.1; warns `Unsupported platform (android)` | Execution unverified | Yes |
| Depth Anything / SAM3 video (HF Spaces) | Yes via network | No live call (needs HF token/queue) | No code change | Yes (network) |
| Replicate raw masks (SAM3) | No (paid) | No | Remove/replace paid dependency | **No** |
| Replicate HD upgrade / vocal separation | No (paid) | No | Remove/replace paid dependency | **No** |
| Enhancor Seedance 2.5 | No (paid) | No | Requires API key + billing | **No** |
| tmpfiles / catbox media hosting | Yes (network) | No live upload | No | Yes |

## Verdict

**PARTIALLY.**

What genuinely runs on-device for free:
- The FFmpeg/Rubber Band compositing and audio pipeline.
- The FastAPI/static web shell (after installing pure-Python wheels).
- Cloudflared callback tunneling.

What does not run as shipped:
- `av` cannot be installed → `tests/test_media.py` and any PyAV import path fail.
- MediaPipe face-mesh path has no installable wheel for Android/arm64 Termux.
- The advertised AI features depend on hosted Spaces (network) or paid APIs (Enhancor Seedance, Replicate SAM3/HD/separation).

## Required Modifications (exact files)

1. `requirements.txt` — relax compiled pins (`numpy==2.4.3`, `Pillow==12.1.1`, `opencv-python-headless==5.0.0.93`) or split into a Termux-specific requirements file using system packages.
2. `setup-face-mesh.sh` — pins `mediapipe==1.1.0` and `av==19.0.1`; neither is installable here. Needs an alternative face-mesh implementation.
3. `audio_workflow.py` — vocal separation calls paid Replicate; no free local equivalent wired in.
4. `raw_masks.py` — SAM3 mask generation calls paid Replicate (`408f82fc…f81269`).
5. `seedance_bridge.py` — requires paid Enhancor Seedance 2.5 API key and a public webhook.
6. `face_mesh.py` / `mesh_worker.py` — MediaPipe-dependent; rewrite with OpenCV or ONNX Runtime.
7. `doctor.py` — extend checks to the Termux package names (`python-numpy`, `python-pillow`, `python-opencv-python`) and to missing `av`/`mediapipe`.
8. `docs/AGENT-OPERATIONS.md` — documents `workflow.py`, which does not exist in the repository.

## Evidence Highlights

- `pip install --dry-run --only-binary=:all:` → `gradio_client==2.7.2` "Would install"; `numpy==2.4.3` and `mediapipe==1.1.0` → "No matching distribution found".
- `pkg` has no `python-av`, `mediapipe`, `python-fastapi`, `python-uvicorn`, or `python-multipart`.
- `find_spec` results: True for `numpy`, `PIL`, `scipy`, `cv2`, `torch`, `onnxruntime`; False for `av`, `mediapipe`, `fastapi`, `uvicorn`, `gradio_client`.
- ONNX Runtime emits `Unsupported platform (android)` from `onnxruntime_validation.py:66`.
- Repo contains no Demucs code despite `AGENTS.md`; `audio_workflow.py` uses paid Replicate separation.
- All neural weights are hosted per `docs/MODELS.md`; nothing is downloaded locally.

## Concise Copy-Paste Version

```
Genjustsu on OPPO CPH2565 / Android 15 / Termux (no root): PARTIALLY compatible.
Works free locally: FFmpeg 9.0.2 + Rubber Band 4.0.0, FastAPI UI (pure wheels), cloudflared.
Works with changes: numpy/Pillow/OpenCV (relax pins), HF Spaces via gradio_client (network).
Does not work: PyAV (no package/wheel), MediaPipe face mesh (no Android wheel).
Paid, not free: Enhancor Seedance 2.5, Replicate SAM3 masks, Replicate HD/vocal separation.
Required edits: requirements.txt, setup-face-mesh.sh, audio_workflow.py, raw_masks.py,
seedance_bridge.py, face_mesh.py/mesh_worker.py, doctor.py, docs/AGENT-OPERATIONS.md.
```
