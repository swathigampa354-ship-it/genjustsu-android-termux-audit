# ANDROID_GUEST_COMPATIBILITY_REPORT.md

**Rootless Android-guest platforms on one specific phone: OPPO A78 4G (CPH2565), Android 15 / ColorOS, ARM64, Termux, no root**

Audit date: 2026-10-22 · Method: source inspection of live `main` branches + GitHub release metadata + on-device read-only diagnostics + one minimal on-device smoke test. No root, no bootloader unlock, no SELinux change, no large downloads (the 708 MB rootfs was NOT downloaded; its runtime was validated instead with a stub guest).

---

## A. Executive verdict

**None of the three repositories can boot an Android guest with a runnable target app on this device today.**

- **VineOS** — the only project aimed at exactly this (a real separate Android OS in userspace). Its current engine is hard-gated behind `CAP_SYS_ADMIN` (root): `unshare()` + `mount()` + loop devices + cgroups. Verified on-device: `unshare(CLONE_NEWNS|NEWPID|NEWUTS|NEWIPC)` → **EPERM** (no capabilities, CapEff=0). Its own README FAQ states the root requirement. The no-root path (Shizuku/proot/AVF) is **probe-only** in code. No bootable ROM exists: roadmap "First bootable Android 7.1.2 ROM image" unchecked, ROM store returns **HTTP 403 from two networks**. → **BLOCKED** (device *and* project).
- **RootlessAndroid15VM** — a ptrace/proot-style userspace Android-rootfs container. The *technique* is one of the few that can work unprivileged on Android (proot works in Termux daily). But the implementation is a **single-commit proof-of-concept (2★)** whose tracer handles only ~`openat/ioctl/faccessat/statx/newfstatat/readlinkat/execve/chdir/getcwd/mount/umount2/get*/setuid(pr)/prctl/connect/…` (~15 syscalls of the several hundred a real Android boot hits); mount ⇒ faked success, `getuid` ⇒ faked root. Android init/zygote cannot boot through this.
  **On-device smoke test (bonus evidence):** compiled its CLI engine in Termux (needed chims: `android/log` no-op, aaudio stub, memfd fallback) and ran it against a stub rootfs executing `/bin/true`: the engine **hangs forever in `do_wait` with no children and zero output** — it does not even complete supervision of a 1-syscall guest. → **BLOCKED in practice** (project non-functional; no demonstration exists anywhere).
- **Podroid** — mature and popular (3585★, 29 releases, v1.2.9), but it boots **Alpine Linux**, not Android. It is a genuine, reliable rootless VM shell: QEMU-TCG backend everywhere, AVF/pKVM **Pixel-only** (this SD680 phone has no pKVM: `ro.boot.hypervisor.*` empty, `virtualizationservice: stopped`). Its guest kernel ships **zero binder/ashmem support** (`podroid_kernel.config`: no `CONFIG_ANDROID_BINDER*`/ashmem lines — verified), so no Android runtime (ART/redroid/Anbox-style) can run inside. Also TCG is CPU-bound 5–20x slower (its own docs), choking any ART attempt even if binder existed. It cannot install/run an Android APK. → **Incomplete for the goal**, though it is the strongest *long-running VM + automation* substrate of the three.

**What actually meets the goal today (no guest):** run the ONE target app on the **host** and control it via an MCP bridge over **on-device wireless-debugging ADB** (`input tap/text`, `screencap`, `uiautomator dump`) driven by a small Python MCP server in Termux. This achieves single-app isolation-by-judgment rather than by VM, works for 10–12 h sessions, and requires zero root. Rated below as Alternative 0.

---

## B. Comparison table

| Criterion | VineOS | RootlessAndroid15VM | Podroid |
|---|---|---|---|
| What it actually is | Android-in-container engine + in-app ROM downloader (Anbox-style) | ptrace sys-call-filter toy engine + pre-patched Android-15 GSI rootfs release | Rootless **Alpine Linux** VM (QEMU-TCG; AVF on Pixels) + Podman/Docker/LXC + X11/VNC viewer |
| Boots complete **Android** guest | Designed yes; **no ROM exists** (store 403) | **No** (prototype; unproven) | **No** (Alpine only; kernel has no binder/ashmem) |
| Runs one Android APK in guest | No (no ROM, no writable `/data` phase yet, no app-install path) | No (cannot boot to zygote even on the project's own design) | No (no Android runtime in guest) |
| Maturity evidence | 118★, 6 mo active, CI + docs + honest roadmap; v0.1.0-alpha, 2 APKs; issue #64 "not booting on release-signed APK" | 1 commit, 2★, 0 issues, no CI, no screenshots; released day-1 708 MB rootfs | 3585★, 29 signed release APKs, real docs incl. honest "limitations" page |
| Root requirement | Root currently **required** (unshare/mount = CAP_SYS_ADMIN); no-root = unimplemented roadmap | No root by design (ptrace of own children) — technique valid, engine isn't | No root (ships own QEMU + prebuilt kernel; no `/dev/kvm` needed) |
| Virtualization/kernel need | unshare+mount+loop+binfmt_misc+cgroups (root-level) | ptrace only (works in untrusted_app domain) | QEMU-TCG (unprivileged) or AVF/pKVM (Pixel-only) |
| Android 15 + ColorOS host | minSdk 26 OK | minSdk 26 OK (app unstarted in practice) | minSdk 26 OK; **phantom-killer caveat on A12+ fixable via ADB setting** (documented by project) |
| ARM64 device compat | arm64-v8a OK (needs 32-bit path only for 32-bit-only ROMs) | arm64-only engine, 16K-page claims | arm64-v8a only — OK on this phone |
| Initial download | APK 7 MB (release) + **ROM blocked (store 403)** | APK (build-only; no published APK) or 708 MB tar.xz rootfs | **311 MB published APK v1.2.9** (bundles Alpine rootfs + QEMU) |
| Footprint (est.) | ROM ~0.7 GB + data 3–8 GB (planned) | ~0.7 GB compressed → **~3–5 GB extracted GSI** (est.) | APK 311 MB + storage.img 2–16 GB (device-capped) |
| Realistic guest RAM | 1.5–3 GB (planned) | ~unbounded (Android userspace, ~1.5–2.5 GB est.) | 512 MB–4 GB selectable; policy leaves ~3 GB for Android |
| This phone fits (15 GB free / ~2.2–2.4 GB RAM free)? | No (no ROM anyway) | Marginal-but-irrelevant (engine doesn't run) | **Yes, as a Linux VM** (512 MB–2 GB VM RAM) |
| Setup complexity | App + ROM (impossible today) | Android Studio + NDK (compile on phone unrealistic — requires NDK; Termux build needed 3 shims) | Install APK → setup wizard → done |
| MCP/AI control surface | **None in repo** (uinput + framebuffer only, no API/ADB-in-guest) | **None** | **Yes for the VM itself**: SSH :9922, SFTP, port-forwards, QMP, ContentProvider `ControlProvider` (start/stop/forwards), automation intents; + host-side adb never needed |
| 10–12 h reliability | Unverifiable (α, design-level fg-service) | Unknown (never runs) | **Likely to strongest**: fg-service + "unrestricted background" + boot-autostart; ColorOS kill risk remains |
| Verdict for this device | **BLOCKED** (root + missing ROM) | **BLOCKED in practice** (non-functional prototype) | **INCOMPLETE for Android app goal**; excellent Linux VM otherwise |

Labels: CONFIRMED = verified from code/docs or this device; estimates marked (est.).

---

## C. Evidence

- Device (this session, read-only): Android 15 / API 35 / kernel 5.15.180-android13-8 / arm64-v8a + armv7 abilist / 7.4 GiB RAM (~2.2–2.4 GiB free) / 15 GB free / CapEff 0 / no su (Termux stub) / `unshare(NEWNS|NEWPID|NEWUTS|NEWIPC)→EPERM`, `unshare(NEWUSER)→EINVAL` / no `/dev/kvm`, no `/dev/vhost-net`, hypervisor props empty, `init.svc.virtualizationservice=stopped` → **no AVF/pKVM**.
- VineOS:
  - Root requirement in README FAQ: https://github.com/Hexadecinull/VineOS#faq ("requires CAP_SYS_ADMIN for unshare() and mount()")
  - Engine root-gated calls: `app/src/main/cpp/container/namespace_manager.cpp` (loop mount, `unshare(CLONE_NEWPID|NEWNS|NEWUTS|NEWIPC)`, pivot_root, cgroups)
  - No-root status: `docs/NOROOT_RESEARCH.md` (probe-only; "needs empirical answer", "not implemented yet")
  - ROM missing: ROADMAP "First bootable Android 7.1.2 ROM image" unchecked (README.md) + store `https://vineos.hexadecinull.dpdns.org/roms/manifest.json` → HTTP 403 (verified twice this audit; also `docs/ROM_STORE_SETUP.md` says site would show "not live yet")
  - Release: v0.1.0-alpha, 2 APKs, no checksums published https://github.com/Hexadecinull/VineOS/releases
- RootlessAndroid15VM:
  - Repo state: created + single push 2026-08-30, 2★, 0 issues, Apache-2.0, 3.4 kLOC — https://github.com/Fiki-io/RootlessAndroid15VM
  - Tracer coverage: `engine/core/syscall_dispatcher.cpp` (handles ~15 syscalls; mount fake-success, identity fake-root), `engine/syscall_interceptor.cpp` (3 handlers, single-trace path)
  - Entry path: `engine/main_cli.cpp` (`spawn_guest` → `run_loop`); orchestrator `engine/guest/guest_init_orchestrator.cpp` is dead code (own `main()`, not linked by either build script)
  - Release rootfs (708 MB): https://github.com/Fiki-io/RootlessAndroid15VM/releases/download/v1.0.0-a15/android15_rootfs.tar.xz (not downloaded per safe-test policy)
  - **This-device smoke test (recorded this audit):** Termux g++ build of `main_cli.cpp + core + ipc + input + display(gralloc/fb)` with shim stubs (`__android_log_print` no-op, aaudio no-op, `memfd_create` fallback) ran engine + stub rootfs + `/bin/true` (toybox): process observed `wchan=do_wait`, `children=[]`, no output for 25 s (timeout) — **supervision loop never completes**. Confirms the ptrace *approach* runs in the app domain, but this implementation does not function.
- Podroid:
  - Identity/limit page (TCG 5–20x, "AVF Pixel-only", phantom-killer exit 137 + ADB workaround, downloads sharing, etc.): https://github.com/ExTV/Podroid/blob/main/docs/guide/limitations.html and `performance.html`
  - Guest kernel has no binder/ashmem: https://github.com/ExTV/Podroid/blob/main/podroid_kernel.config (0 matches for binder/ashmem — checked)
  - Control surfaces for automation/MCP: `engine/control/ControlProvider.kt` (ContentProvider: start/stop VM, backend select, port-forwards, navigate), automation intents in v1.2.8 release notes, SSH/SFTP :9922, `QmpClient.kt`
  - Resource policy (RAM options capped leaving ~3 GB for Android; storage ≤ ~25% of free, min 2 GB): `util/DeviceResourcePolicy.kt`
  - Releases: https://github.com/ExTV/Podroid/releases (29 signed APKs; latest v1.2.9, 311 MB)

---

## D. Blockers

**VineOS** (all CONFIRMED except where noted)
1. `unshare()`/`mount()` need CAP_SYS_ADMIN → EPERM on this device (measured); CapEff=0; no root; no Shizuku installed.
2. No writable guest `/data`, no app-install path → one-APP goal structurally not implemented yet.
3. No ROM published; store 403 → nothing to boot.
4. Even with root/emergency: no ROM → stop.
(Only unknown: whether root+future ROM would boot — moot.)

**RootlessAndroid15VM**
1. Engine cannot boot any guest beyond a trivial binary (measured: hangs on `/bin/true` supervision).
2. Syscall surface (~15 handlers) far below an Android 15 boot (needs mount/mknod-ish, setgroups, clone-costume, membarrier, futexability across tracer, binder-socket activations… — not present).
3. No demonstration anywhere (1 commit, 0 issues, no forks adoption), 2★.
4. App build requires NDK (Termux build needed hand shims → published APK doesn't exist).
5. MCP surface: none.
⇒ Not a candidate without major external development; rootfs existing ≠ runtime existing.

**Podroid**
1. Guest is Alpine Linux — **Android app runs impossible**: no binder/ashmem in guest kernel + no ART.
2. QEMU-TCG only (this phone has no pKVM/AVF): 2–5x..5–20x CPU slowdown (project's own docs) — disqualifying for ART even as a hypothetical.
3. Its remaining limits matter only for Linux use: 9p instability on some ARMv9.2 (not our SD680-class), Wayland-only desktop exposure, Raw VNC encoding.
4. A12+ phantom-process killer default can SIGKILL the VM (exit 137); fixable once via `adb shell` setting or Shizuku (no root).
⇒ Best-engineered of the three, but wrong guest OS for the stated goal.

---

## E. Ranked recommendations (feasibility for YOUR goal)

1. **Alternative 0 — no-guest host control (only option that works now):** target app on host; control via on-device ADB (Wireless debugging) MCP bridge. Lowest resources, zero root, 10–12 h friendly. Phone remains usable (foreground app switching just fine).
2. **Podroid — only if you accept a *Linux* guest instead of an Android one.** Realistic rootless VM for automation/AI purposes on this device; MCP-bridge substrate exists (SSH + ContentProvider + intents). Cannot host the one Android app. (311 MB download; 1–2 GB VM RAM; TCG speed tradeoff.)
3. **VineOS — watchlist.** Correct architecture for the real goal; blocked today by root-gate + zero ROMs. Re-evaluate when (a) ROM store returns HTTP 200 with ROM entries AND (b) Phase 4 no-root backend lands in code (proot/AVF), not just checkboxes.
4. **RootlessAndroid15VM — not usable.** Prototype whose core loop demonstrably halts even on a stub guest; would need proot-distro-grade engineering (months) + a display path + an MCP surface.

---

## F. Next actions — shortest practical sequence (Alternative 0)

Goal: MCP-driven control of ONE host app on this phone, no root, reversible.

Prereq approvals needed only for two on-device toggles you control: enable **Developer options → USB/Wireless debugging** once (path: Settings → About phone → tap Build number ×7).

```bash
# 1) tools in Termux (small, reversible)
pkg install -y android-tools  # adb, fastapi not needed
# 2) pair phone to itself: Settings → Developer options → Wireless debugging
#    → "Pair device with pairing code" (split-screen so you can type in Termux)
adb pair 127.0.0.1:PORT    # enter 6-digit code shown on screen
adb connect 127.0.0.1:5555 2>/dev/null || adb devices     # connect; keep Wireless debugging on
# 3) sanity: app inventory + the ONE target app package name
adb shell pm list packages | grep <target-app-substring>
# 4) control verbs the MCP bridge will expose (test manually first)
adb shell uiautomator dump /sdcard/window_dump.xml && adb shell cat /sdcard/window_dump.xml
adb shell input tap 500 900
adb shell screencap -p /sdcard/screen.png
# 5) small MCP server skeleton (Python), then plug into opencode/Claude
#    tools: tap(x,y) | swipe | text() | dump_ui() | screenshot() | app_state()
python -m pip install mcp  # or implement stdio JSON-RPC by hand, no pip needed
```

Expected results / stop conditions:
- After (3): package listed ⇒ ADB works. If `adb devices` shows unauthorized → re-pair.
- If ColorOS revokes wireless debugging on reboot → re-pair (2 min) or use Shizuku-onboot-up equivalent later.
- **Stop/skip**: any step that asks for root/bootloader/SELinux → abort; we never do that.
- Rollback: `adb disconnect; pkg uninstall android-tools; rm -rf ~/mcp-bridge` — zero residue.
Guest track (only when you explicitly want isolation later): re-run audit if VineOS ROM store returns HTTP 200; fully document Podroid Alpine path only if your app is Linux-capable.

---

## G. Confidence

| Conclusion | Level |
|---|---|
| No root/no caps/no unprivileged userns on this phone; no pKVM/AVF; no /dev/kvm | CONFIRMED |
| VineOS engine requires root today (README + code + EPERM test) | CONFIRMED |
| VineOS has no downloadable ROM (store 403 ×2, roadmap unchecked) | CONFIRMED |
| RootlessAndroid15VM engine fails even trivial supervision (live test) | CONFIRMED |
| RootlessAndroid15VM "Android 15 in userspace" achievable someday | UNKNOWN (technique plausible; project not there) |
| Podroid guest cannot run Android apps (no binder in kernel; docs TCG-only here) | CONFIRMED |
| Podroid as Linux VM works on this phone incl. RAM ~512 MB–2 GB | LIKELY (well-supported architecture; not runtime-tested here) |
| 10–12 h under 2.2-GiB-RAM pressure for any VM | LIKELY-BLOCKED (ColorOS kill pressure; ADB-change mitigation documented by Podroid) |
| Host-app MCP via wireless-debugging ADB on Android 15 | LIKELY→CONFIRMED pattern (input/screencap/uiautomator are standard adb verbs; run sequence F validates) |
| Any of the 3 meets "one Android app in guest + MCP" on THIS device | **NO — all three fail; goal requires Alternative 0 or a future no-root guest** |

*Notes: no_root_goal: never rooted/unlocked; tmp builds confined to `/tmp/opencode`; nothing irreversible performed; all repository clones kept under `…/tmp/opencode/guest-audit/`.*
