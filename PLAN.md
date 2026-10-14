# FallVision — Plan

## Why this is harder than AVP

AVP started from a native ARM64 build of the game. Titanfall 2 only exists as an x86-64 Windows binary (modified Source, D3D11), and the VR mod is a closed-source x64 DLL loaded through Northstar. Nothing in that stack can be recompiled; all of it has to run as-is under emulation. The Mac precedent (CrossOver/GPTK) proves the game fits an Apple GPU in flat mode, but GPTK relies on Rosetta 2, D3DMetal and macOS process semantics, none of which exist on visionOS.

## Known-good pieces (reuse, no research)

| Piece | Source | State |
|---|---|---|
| JIT on device | StikDebug, proven in AstroVisionPro | working on visionOS 27.2 |
| Expanded memory (~8 GB per app) | AstroVisionPro | working |
| x86-64 → ARM64 JIT | FEX-Emu | mature on Linux; used by AstroVisionPro path |
| D3D11 → Vulkan | DXVK | mature |
| Vulkan → Metal, foveation, VPKTAA | VPKEngine (VPVulkan) | in progress, shared with AVP |
| Steam sign-in + depot download + launcher | AVP | working |
| Compositor Services + PSVR2 Sense input | AVP / VisionQuestPro | working |

## The unknown: Wine as a single process

Wine assumes a multi-process world: `wineserver`, `services.exe`, `winedevice.exe`, `explorer.exe`, `plugplay`, and the game itself are separate OS processes talking over sockets and shared memory. visionOS apps cannot `fork`, `posix_spawn` or launch helpers. The whole Wine runtime has to live in one process as threads, with `wineserver` turned into an in-process server and the service processes either stubbed or run as threads.

This is the only piece nobody has done on visionOS. Everything else is integration. **No other work starts until this gate is passed.**

## Gates (strict order)

### Gate 0 — JIT ✅
StikDebug JIT confirmed in AstroVisionPro. Reuse the same enable flow in FallVision's launcher.

### Gate 1 — Wine single-process, hello world (the spike)
Deliverable: a visionOS IPA that boots Wine ARM64EC in-process, with FEX as the WoW64 layer, and runs an x64 `hello.exe` that prints to the log.
- Build Wine for ARM64 Darwin against the visionOS SDK (static libs into the app).
- In-process `wineserver`: replace the socket/`fork` transport with a thread + pipes; no process creation anywhere.
- Stub or thread-ify: `services.exe`, `winedevice.exe`, `plugplay`, `explorer.exe`, `rpcss`.
- FEX built as the `wow64` emulator interface (`libwow64fex`), JIT enabled via StikDebug.
- Filesystem: Wine prefix inside the app container; game folder outside it (user-visible, survives deletion).
- Exit criteria: `hello.exe` output appears in the in-app log. Nothing graphical yet.

### Gate 2 — D3D11 triangle
Deliverable: an x64 D3D11 sample (`triangle.exe`) rendered flat in a window on the headset.
- DXVK as `d3d11.dll`/`dxgi.dll` over VPVulkan.
- Present path into a visionOS window (flat), no VR yet.
- Exit criteria: triangle visible, ≥60 fps, no leaks over 10 min.

### Gate 3 — Titanfall 2 flat
Deliverable: `Titanfall2.exe` (vanilla, no Northstar) reaches the main menu in flat mode.
- Steam depot download via the AVP launcher code.
- Launch requirements of the retail build must be resolved here. **Decision pending (owner): how the EA app requirement is handled.** Emulating the EA client inside Wine is the highest-risk, lowest-value piece of the whole project and is not on the critical path until this decision is made.
- Exit criteria: menu reachable, campaign loads, playable with a Sense controller mapped as gamepad.

### Gate 4 — Northstar + TF2VR + OpenXR
Deliverable: the VR mod boots and renders stereo.

What the PC installer does (from `CircuitLord/CircuitLordVRModInstaller`, MIT, and the 1.0.17 package); FallVision reproduces it in-app:
- Northstar pinned to **v1.31.13** (`R2Northstar/Northstar/releases/download/v1.31.13/Northstar.release.v1.31.13.zip`, sha256 `e622b96e…`). `NorthstarLauncher.exe` → `<game>/Titanfall2VRLauncher.exe`; `R2Northstar/*` → `<game>/TF2VR/`.
- Mod package from the installer repo's `mod-packages` release (`Titanfall2VR-<ver>.zip`, URL + sha256 in `manifest-v3.json`). Contents: `Titanfall2VR.dll` → `TF2VR/plugins/`, `mods/` → `TF2VR/mods/`, `launch.json`, `release.json`, plus `patches/` (bsdiff against the installed game's VPK files) and `asset_patcher.exe` which builds the game-derived mod files at install time. `xr_probe.exe` and `crash_monitor.exe` are the PC launcher's helpers; FallVision's launcher replaces them.
- **The mod is closed source and may not be redistributed.** FallVision never bundles it: the app downloads it from the upstream URL at install time, verifies the sha256, and keeps it in the user's game folder.
- `asset_patcher.exe` is x64; either run it under FEX at install time or reimplement the bsdiff apply step natively in the app (simpler, no emulation needed before the game runs).
- Launch: `Titanfall2VRLauncher.exe -profile=TF2VR -novid -windowed -w 1280 -h 720 +ns_has_agreed_to_send_token 2 +ns_masterserver_hostname http://127.0.0.1:9 +ns_report_server_to_masterserver 0 +sp_not_focus_pause 0 +not_focus_sleep 0 +sound_without_focus 1` plus the `vrArguments` from `launch.json` (DoF off, vsync off, `fps_max 0`, `mat_viewportscale 1`), env `TF2VR_OPENXR=1`. Masterserver is pointed at localhost, so Northstar runs offline.
- The mod must find real `dxgi.dll`/`d3d11.dll` (it hooks `Present`); it refuses to run if a mod dropped replacements beside the game. DXVK as Wine builtins is fine; nothing goes in the game folder.

OpenXR runtime:
- The mod embeds the OpenXR loader and honours `XR_RUNTIME_JSON`. It requests `XR_KHR_D3D11_enable`, `XR_EXT_hand_tracking`, `XR_EXT_hand_joints_motion_range`, `XR_EXT_user_presence`, `XR_KHR_visibility_mask`, `XR_KHR_win32_convert_performance_counter_time`, `XR_EXT_debug_utils`; interaction profiles `oculus/touch_controller`, `valve/index_controller`, `khr/simple_controller`.
- Template: Proton's `wineopenxr` — a Wine builtin DLL that is the OpenXR runtime inside the Windows process, with D3D11 swapchains bridged through DXVK's Vulkan images. FallVision's runtime does the same but ends in Compositor Services (VPVulkan images → Metal textures), headset poses, Sense controllers as `oculus/touch_controller`, and Vision Pro hand tracking exposed through `XR_EXT_hand_tracking` (the mod already animates fingers from it).
- Exit criteria: campaign intro in stereo with working hands.

### Gate 5 — Performance
Only measurable once Gate 4 passes. Levers, in order: render resolution, round foveation + VPKTAA on the periphery (as in PTVisionPro/AVP), DXVK/FEX tuning. Target: stable 45 fps on M2. If unreachable, stop and document.

## What is explicitly out of scope

- Multiplayer (the mod itself is campaign-only).
- Any Mac-side tooling: everything runs on the headset or in CI.
- MetalFX upscaling.
- Shipping any game data, Northstar or TF2VR binaries in this repo or in CI.

## Repository layout (planned)

```
FallVision/          visionOS app (SwiftUI launcher, log, settings)
wine/                Wine ARM64EC patches + build scripts (single-process)
fex/                 FEX build scripts for visionOS
runtime/             OpenXR runtime (C++, Compositor Services bridge)
.github/workflows/   build on manual dispatch or [build] in the commit message
bitacora.md          working log for the next session
```

## Effort (Claude, not human)

- Gate 1: the one that decides the project. Expect many small build/test cycles; the repo is public, so CI Mac minutes are not a constraint — on-device test time is.
- Gates 2, 4: days of sessions each; mostly integration of known pieces.
- Gate 3: unknown until the EA decision is made.
- Gate 5: unknown.

**Do not start Gate 1 while AVP, AstroVisionPro or VPKEngine are mid-build.** Gate 1 needs sustained attention and many on-device test rounds; splitting it with other ports slows all of them.
