# VoxRT

**Audio AI on the CPU — runtime and models built from scratch.**

We design two halves of the same product:

1. A custom **inference runtime** written in Rust, tuned for streaming audio on commodity CPUs — no GPU, no NPU, no vendor accelerator in the critical path.
2. **Audio models** — voice activity detection, streaming speech recognition, wake-word, and (soon) keyword-spotting and domain-specific ASR — packaged to run on that runtime.

The runtime is CPU-only by design. Real deployments don't have a free GPU sitting around — they have a constrained budget on a single ARM core and a battery indicator the user is watching. On a **$15 Raspberry Pi Zero 2 W**, our wake-word burns **5 % of one A53 core**, sustained. That's the runtime story in one number.

## What CPU-first buys you

| | Off-the-shelf mobile / edge runtimes | VoxRT runtime |
|---|---|---|
| Binary size | 5–20 MB | ~600 KB |
| Streaming-audio fit | retrofitted | designed for it |
| Universal ARMv8 kernels | partial | yes — one binary, A53 to flagship |
| Hot-path allocations | many | none (pre-allocated scratch) |
| Encrypted weights at rest | rare | AES-256-GCM by default |
| Scalar vs NEON speedup | undocumented per-kernel | **8.7× on Cortex-A73** — 0.182 → 0.021 RTF, [full methodology](https://github.com/VoxRT/voxrt-wake-word-linux#neon-vs-scalar) |

Concretely: zero allocations in the streaming inference loop, scalar/NEON kernels that match each other bit-exactly so we can ship one binary across the whole CPU tier, and a `.vxrt` model format that stays mmap-loadable end-to-end (the bytes never round-trip through a managed heap).

## What we ship today

Open, free, proof-of-runtime products. Same JitPack / Swift Package Manager / PyPI / npm / Go / crates channels real consumers use:

| Product | Android | iOS | Linux aarch64 | Models |
|---|---|---|---|---|
| **Silero VAD** on the VoxRT runtime | [voxrt-silero-android](https://github.com/VoxRT/voxrt-silero-android) | [voxrt-silero-ios](https://github.com/VoxRT/voxrt-silero-ios) | on request | [voxrt-silero-models](https://github.com/VoxRT/voxrt-silero-models) |
| **Streaming ASR** (NeMo FastConformer Medium, 32M) | [voxrt-asr-android](https://github.com/VoxRT/voxrt-asr-android) | [voxrt-asr-ios](https://github.com/VoxRT/voxrt-asr-ios) | on request | [voxrt-asr-models](https://github.com/VoxRT/voxrt-asr-models) |
| **Wake-word** ("Hey Assistant") | [voxrt-wake-word-android](https://github.com/VoxRT/voxrt-wake-word-android) | [voxrt-wake-word-ios](https://github.com/VoxRT/voxrt-wake-word-ios) | [voxrt-wake-word-linux](https://github.com/VoxRT/voxrt-wake-word-linux) | [voxrt-wake-word-models](https://github.com/VoxRT/voxrt-wake-word-models) |

The Linux SDK ships as one hardened `.so` behind five language wrappers — C / C++ (tarball + CMake + pkg-config), Python (PyPI wheel, `abi3` covers 3.9-3.13), Node.js (npm), Go (`go get`), Rust (git). One binary across Raspberry Pi 3 / 4 / 5 / Zero 2, NVIDIA Jetson, AWS Graviton, and every other aarch64 Linux SBC on a glibc 2.17+ baseline.

Reference performance on a single CPU core:

| Product | Device | RTF | CPU budget |
|---|---|---|---|
| Wake-word | Raspberry Pi Zero 2 W (Cortex-A53 @ 1.0 GHz) | **0.053** | 5.3 % |
| Wake-word | Snapdragon 662 (Cortex-A73 @ 2.0 GHz + NEON) | **0.021** | 2.1 % |
| Wake-word | iPhone 13 Pro Max (Apple A15) | **0.015** | 1.5 % |
| Streaming ASR | Snapdragon 662 (Cortex-A73) | 0.30 | 30 % |
| Streaming ASR | iPhone 13 Pro Max (A15) | 0.08–0.10 | ~9 % |
| Silero VAD | Apple A15 | ~0.6 ms / frame | negligible |

## What we sell

In-house models built on the same runtime — custom wake phrases (your own brand name, your own languages), keyword-spotting, voice-bio, domain ASR. The open libraries above are the proof-of-runtime; the commercial roster is what funds the runtime work.

Same runtime, same kernels, same toolchain — adding a new model is wiring weights into the existing op set, not rewriting the deploy story.

Licensing, OEM integration, custom model packaging: **help@voxrt.com** · [voxrt.com](https://voxrt.com)

## Engineering principles

- **CPU first.** Single-thread ARMv8 NEON is the target. GPU / NPU paths are a future ROI question, not the foundation.
- **One binary across the CPU tier.** Universal NEON kernels — same code on cheap-tier A53 and flagship X-series. Runtime feature detection is opt-in, not load-bearing.
- **Battery-aware by construction.** Zero allocations on the hot path. No `f64` accumulators where `f32` works. Every kernel is profiled against the encoder budget before it ships.
- **Bit-exact validation.** Every kernel matches a reference numerics baseline within float noise; no "looks about right." NEON has to equal scalar within ULP budget, or the patch doesn't land.
- **Closed where it matters, open where it ships.** Runtime is proprietary; the consumer-facing Kotlin / Swift / Python / JS / Go wrapper layers are Apache-2.0 in the open.

## Targets

- **iOS** (iPhone, iPad — arm64)
- **Android** (arm64-v8a, x86_64 emulator)
- **Embedded Linux aarch64** — Raspberry Pi 3 / 4 / 5 / Zero 2, NVIDIA Jetson, AWS Graviton, Rock Pi / Orange Pi / Khadas SBCs (glibc 2.17+; wake-word SDK shipped, VAD + ASR on request)
- macOS, desktop Linux — on demand

## Stack

Rust • ARMv8 NEON intrinsics • cbindgen • pyo3 • napi-rs • cgo • cargo-ndk • cargo-zigbuild • Swift Package Manager • JitPack • PyPI • npm • Go modules • Xcode xcframework

---

*If you're integrating on-device audio and your CPU budget or battery is the bottleneck, we want to talk.*
