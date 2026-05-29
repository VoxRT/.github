# VoxRT

**Audio AI on the CPU — runtime and models built from scratch.**

We design two halves of the same product:

1. A custom **inference runtime** written in Rust, tuned for streaming audio on commodity CPUs — no GPU, no NPU, no vendor accelerator in the critical path.
2. **Audio models** — voice activity detection, streaming speech recognition, and (soon) wake-word, keyword-spotting, and domain-specific ASR — packaged to run on that runtime.

The runtime is CPU-only by design. Real deployments don't have a free GPU sitting around — they have a constrained budget on a single ARM core and a battery indicator the user is watching.

## What CPU-first buys you

| | Off-the-shelf mobile / edge runtimes | VoxRT runtime |
|---|---|---|
| Binary size | 5–20 MB | ~600 KB |
| Streaming-audio fit | retrofitted | designed for it |
| Universal ARMv8 kernels | partial | yes — one binary, A53 to flagship |
| Hot-path allocations | many | none (pre-allocated scratch) |
| Encrypted weights at rest | rare | AES-256-GCM by default |
| Power profile | "best effort" | measured against a watt meter |

Concretely: zero allocations in the streaming inference loop, scalar/NEON kernels that match each other bit-exactly so we can ship one binary across the whole CPU tier, and a `.vxrt` model format that stays mmap-loadable end-to-end (the bytes never round-trip through a managed heap).

## What we ship today

Open, free, proof-of-runtime products. Same JitPack / SwiftPM channels real consumers use:

| Product | Android | iOS | Models |
|---|---|---|---|
| **Silero VAD** on the VoxRT runtime | [voxrt-silero-android](https://github.com/VoxRT/voxrt-silero-android) | [voxrt-silero-ios](https://github.com/VoxRT/voxrt-silero-ios) | [voxrt-silero-models](https://github.com/VoxRT/voxrt-silero-models) |
| **Streaming ASR** (NeMo FastConformer Medium, 32M) | [voxrt-asr-android](https://github.com/VoxRT/voxrt-asr-android) | [voxrt-asr-ios](https://github.com/VoxRT/voxrt-asr-ios) | [voxrt-asr-models](https://github.com/VoxRT/voxrt-asr-models) |

Reference performance on a single CPU core: VAD frame ~0.6 ms on Apple A15, streaming ASR realtime factor 0.08–0.10 on the same. On a midrange Snapdragon 662 (Cortex-A73) — live-mic ASR at RTF 0.35, leaving 65% of one core for the rest of the audio pipeline.

## What we sell

In-house models built on the same runtime — wake-word, keyword-spotting, voice-bio, domain ASR. The open libraries above are the proof-of-runtime; the commercial roster is what funds the runtime work.

Same runtime, same kernels, same toolchain — adding a new model is wiring weights into the existing op set, not rewriting the deploy story.

Licensing, OEM integration, custom model packaging: **help@voxrt.com** · [voxrt.com](https://voxrt.com)

## Engineering principles

- **CPU first.** Single-thread ARMv8 NEON is the target. GPU / NPU paths are a future ROI question, not the foundation.
- **One binary across the CPU tier.** Universal NEON kernels — same code on cheap-tier A53 and flagship X-series. Runtime feature detection is opt-in, not load-bearing.
- **Battery-aware by construction.** Zero allocations on the hot path. No `f64` accumulators where `f32` works. Every kernel is profiled against the encoder budget before it ships.
- **Bit-exact validation.** Every kernel matches a reference numerics baseline within float noise; no "looks about right." NEON has to equal scalar within ULP budget, or the patch doesn't land.
- **Closed where it matters, open where it ships.** Runtime is proprietary; the consumer-facing Kotlin / Swift wrapper layers are Apache-2.0 in the open.

## Targets

- iOS (iPhone, iPad — arm64)
- Android (arm64-v8a today; armeabi-v7a, x86_64 emulator on demand)
- Embedded Linux on ARM (Raspberry Pi 4/5, NXP i.MX, Rockchip — on request)
- macOS, desktop Linux — on demand

## Stack

Rust • ARMv8 NEON intrinsics • cbindgen • Swift Package Manager • JitPack • cargo-ndk • Xcode xcframework

---

*If you're integrating on-device audio and your CPU budget or battery is the bottleneck, we want to talk.*
