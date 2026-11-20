# Genjustsu Android/Termux Audit — Reports

Compatibility audits and research reports produced on a rootless Android phone running Termux.

- **Device:** OPPO CPH2565 (A78 4G), Android 15 / API 35, SM6225, arm64, 7.4 GB RAM, no root
- **Environment:** Termux 0.118.3, Python 3.14.8 (`android-24-arm64_v8a`), no paid services
- **Genjustsu verdict:** **PARTIALLY** compatible — the media/UI shell runs locally, but the AI paths are hosted (network-only) or paid, and the face-mesh path needs a MediaPipe replacement.

## Reports

| Report | Summary |
|---|---|
| [ANDROID-TERMUX-AUDIT.md](ANDROID-TERMUX-AUDIT.md) | Audit of [sirioberati/Genjustsu-Open-Source-Workflow](https://github.com/sirioberati/Genjustsu-Open-Source-Workflow) on Termux: FFmpeg/Rubber Band + FastAPI UI work, PyAV/MediaPipe do not, Depth/SAM3/Seedance are hosted or paid |
| [ANDROID-GUEST-COMPATIBILITY-REPORT.md](ANDROID-GUEST-COMPATIBILITY-REPORT.md) | Rootless Android-guest platforms (VineOS, RootlessAndroid15VM, etc.) on the same phone: VineOS BLOCKED (`unshare` → EPERM, no `CAP_SYS_ADMIN`), 708 MB rootfs not downloaded, stub-guest validation |
| [HACKATHON-RESEARCH-REPORT.md](HACKATHON-RESEARCH-REPORT.md) | "Build With AI: Basics" hackathon research report (devpost.com, deadline Oct 26 2026 5:00 pm EDT), evidence convention VERIFIED/INFERRED/UNKNOWN |

## TL;DR — Genjustsu on Termux

| Area | Result |
|---|---|
| FFmpeg + Rubber Band pipeline | Works (installed, verified) |
| FastAPI web UI | Works with dependency changes (pure Python wheels) |
| NumPy / Pillow / SciPy / OpenCV | Works with relaxed pins |
| PyAV (`av`) | Not installable — no Termux package, no matching wheel |
| MediaPipe face mesh | Not natively runnable — needs OpenCV/ONNX rewrite |
| Depth / SAM3 | Hosted Hugging Face Spaces — network only |
| Seedance 2.5 / Replicate | Paid APIs |
| Cloudflared webhook | Works |
