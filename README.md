# Genjustsu Open Source Workflow — Android/Termux Audit

Compatibility audit of [sirioberati/Genjustsu-Open-Source-Workflow](https://github.com/sirioberati/Genjustsu-Open-Source-Workflow) on a rootless Android phone running Termux.

- **Device:** OPPO CPH2565 (A78 4G), Android 15 / API 35, SM6225, arm64, 7.4 GB RAM, no root
- **Environment:** Termux 0.118.3, Python 3.14.8 (`android-24-arm64_v8a`), no paid services
- **Verdict:** **PARTIALLY** compatible — the media/UI shell runs locally, but the AI paths are hosted (network-only) or paid, and the face-mesh path needs a MediaPipe replacement.

## TL;DR

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

Full report: [ANDROID-TERMUX-AUDIT.md](ANDROID-TERMUX-AUDIT.md)
