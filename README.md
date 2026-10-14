# FallVision

Standalone native port of **Titanfall 2 VR** (CircuitLord's VR mod on top of Northstar) to Apple Vision Pro.

The app runs entirely on the headset: it signs in to Steam, verifies ownership, downloads the Windows x64 depot, and runs the game inside a single-process Windows compatibility layer (Wine ARM64EC + FEX JIT) with Direct3D 11 translated to Metal and an in-app OpenXR runtime backed by Compositor Services.

## Status

**Planning / spike.** Nothing is built yet. The project is gated on one unknown: whether Wine can run as a single process inside a visionOS app. See [PLAN.md](PLAN.md) for the gates and the order of work.

## Architecture (target)

```
FallVision.app (visionOS, native ARM64)
├── Launcher            Steam sign-in, depot download, settings, logs
├── Wine ARM64EC        single-process build, no child processes
│   └── FEX (WoW64)     x86-64 → ARM64 JIT for the game, Northstar, TF2VR
├── DXVK → VPVulkan     D3D11 → Vulkan → Metal (from VPKEngine)
├── OpenXR runtime      in-app, over Compositor Services + PSVR2 Sense
└── Game folder         user-visible, survives app deletion
```

## Credits

Written by Claude, building on the great work of [Wine](https://www.winehq.org/), [FEX-Emu](https://fex-emu.com/), [DXVK](https://github.com/doitsujin/dxvk), [Northstar](https://northstar.tf/), CircuitLord's [Titanfall 2 VR](https://github.com/CircuitLord/CircuitLordVRModInstaller) and [VPKEngine](https://github.com/escojoncio/VPKEngine). The repository owner contributed ideas on how some things could be adapted.

Titanfall 2 is © Respawn Entertainment / Electronic Arts. This project ships no game data; a legitimately owned copy is required.
