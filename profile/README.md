# VoxRT

**Voice AI that runs on device. Runtime and models built from scratch, for the CPU.**

We build two halves of the same product:

1. A custom **inference runtime** in Rust, written for streaming audio on commodity CPUs: no GPU, no NPU, no vendor accelerator in the critical path. One code base runs as NEON on ARMv8 (phones, Raspberry Pi, Jetson, Graviton), AVX2 on x86-64 and SIMD128 in WebAssembly.
2. **Audio models** that run on it: text-to-speech, streaming speech recognition, wake word, keyword spotting and voice activity detection.

Real deployments do not have a spare GPU. They have one ARM core, a battery indicator the user is watching, and a privacy policy that says audio never leaves the device. On a $15 Raspberry Pi Zero 2 W our wake word takes 5 % of one A53 core, sustained; on a phone our text-to-speech starts speaking about 40 ms after it gets the text.

## What we ship

| Product | Android | iOS | Linux | Browser | Models |
|---|---|---|---|---|---|
| **LinnetTTS**, text-to-speech (41 M, English, streaming every 80 ms, voice prompts) | [LinnetTTS](https://github.com/VoxRT/LinnetTTS) (AAR) | same repo (Swift Package) | same repo (C, Python, Rust) | [demo on voxrt.com](https://voxrt.com/text-to-speech#live-demo) | [linnet-en-35m](https://huggingface.co/VoxRT/linnet-en-35m), [demo Space](https://huggingface.co/spaces/VoxRT/linnet-tts) |
| **Wake word** ("Hey Assistant", ~100 KB model) | [voxrt-wake-word-android](https://github.com/VoxRT/voxrt-wake-word-android) | [voxrt-wake-word-ios](https://github.com/VoxRT/voxrt-wake-word-ios) | [voxrt-wake-word-linux](https://github.com/VoxRT/voxrt-wake-word-linux) (aarch64) | [voxrt-wake-word-browser](https://github.com/VoxRT/voxrt-wake-word-browser) | [voxrt-wake-word-models](https://github.com/VoxRT/voxrt-wake-word-models) |
| **Keyword spotting** (14 commands, 636 K-parameter streaming Conformer) | [voxrt-kws-android](https://github.com/VoxRT/voxrt-kws-android) | [voxrt-kws-ios](https://github.com/VoxRT/voxrt-kws-ios) | [voxrt-kws-linux](https://github.com/VoxRT/voxrt-kws-linux) (x86-64, aarch64) | [voxrt-kws-browser](https://github.com/VoxRT/voxrt-kws-browser) | [voxrt-kws-models](https://github.com/VoxRT/voxrt-kws-models) |
| **Streaming ASR** (NeMo FastConformer 32 M, CTC + RNN-T, cache-aware streaming) | [voxrt-asr-android](https://github.com/VoxRT/voxrt-asr-android) | [voxrt-asr-ios](https://github.com/VoxRT/voxrt-asr-ios) | [voxrt-asr-linux](https://github.com/VoxRT/voxrt-asr-linux) (x86-64, aarch64) | demo on voxrt.com | [voxrt-asr-models](https://github.com/VoxRT/voxrt-asr-models) |
| **Voice activity detection** (Silero v5) | [voxrt-silero-android](https://github.com/VoxRT/voxrt-silero-android) | [voxrt-silero-ios](https://github.com/VoxRT/voxrt-silero-ios) | on request | | [voxrt-silero-models](https://github.com/VoxRT/voxrt-silero-models) |

Channels are the ones developers already use: JitPack and Gradle on Android, Swift Package Manager on iOS, PyPI, npm, `go get`, C tarballs with CMake and pkg-config, Rust as a git dependency, npm for the browser packages. The Linux SDKs are one `.so` per architecture on a glibc 2.17 baseline, so the same binary runs on Raspberry Pi 3, 4, 5 and Zero 2, NVIDIA Jetson, AWS Graviton and any x86-64 server from Haswell on. The models are mirrored on [Hugging Face](https://huggingface.co/VoxRT).

All four engines run side by side in a browser tab at [voxrt.com/try](https://voxrt.com/try).

## Numbers

Measured on the shipped builds, one thread unless stated. RTF = compute time / audio time, lower is better.

| Product | Device | Result |
|---|---|---|
| LinnetTTS | Apple M5 | 3.3 ms per 80 ms of speech (RTF 4 %) |
| LinnetTTS | iPhone 13 Pro Max (A15) | 5.5 ms per step (RTF 7 %), first audio 38 ms after the text |
| LinnetTTS | Snapdragon 662, 4 threads | 14 ms per step (RTF 18 %) |
| LinnetTTS | Raspberry Pi Zero 2 W, 4 threads | 59 ms per step (RTF 74 %), still real time |
| LinnetTTS | Core i9-13900K, one core | RTF 10.6 % including the prime pass, first audio 22 ms, 187 MB peak |
| Wake word | Raspberry Pi Zero 2 W (A53 @ 1 GHz) | RTF 5.3 % |
| Wake word | Snapdragon 662 (A73) / iPhone 13 Pro Max | RTF 2.1 % / 1.5 % |
| Wake word, WebAssembly | iPhone 13 Pro Max, Safari | RTF 0.23 % |
| Keyword spotting | Snapdragon 662 (A73) / iPhone 13 Pro Max | RTF 16 % / 8.9 % |
| Keyword spotting | x86-64 AVX2 / Raspberry Pi Zero 2 W | RTF 1 % / 32 % |
| Streaming ASR | x86-64 AVX2 | RTF 5.9 % (CTC), 8.0 % (RNN-T); WER 3.27 % (RNN-T) on a 500-utterance LibriSpeech test-clean subset |
| Streaming ASR | iPhone 13 Pro Max / Snapdragon 662, live microphone | RTF 8–10 % / 35 % |
| Voice activity detection | Snapdragon 662 / iPhone 13 Pro Max | RTF 3.05 % / 1.85 % |

Every repository has the method behind its numbers (device, clocks, chunk sizes, warm-up) and the scripts to reproduce them. LinnetTTS also has a [side-by-side benchmark](https://github.com/VoxRT/LinnetTTS/blob/main/docs/benchmarks.md) against Piper, Kokoro and pocket-tts on the same CPU, with audio.

## The runtime

| | Typical mobile / edge runtimes | VoxRT runtime |
|---|---|---|
| Binary size | 5–20 MB | about 600 KB per engine |
| Streaming audio | retrofitted | the only mode |
| ARMv8 kernels | partial | universal, one binary from Cortex-A53 to Apple M-series |
| x86-64 / browser | separate stacks | the same kernels as AVX2 + FMA and WebAssembly SIMD128 |
| Hot-path allocations | many | none (pre-allocated scratch) |
| Numerics | "looks about right" | bit-exact across scalar, NEON, AVX2 and WebAssembly |
| Model weights at rest | plain files | encrypted `.vxrt` for the native voice SDKs; plain safetensors for LinnetTTS |
| Scalar vs NEON | undocumented | 8.7× on Cortex-A73, [method published](https://github.com/VoxRT/voxrt-wake-word-linux#neon-vs-scalar) |

Adding a model is wiring its weights into the existing operator set, not a new deployment story. The same team wrote the models: the wake word, keyword spotting and LinnetTTS models are trained in house; the ASR is NVIDIA's FastConformer (CC-BY-4.0) and the VAD is Silero v5 (MIT), both ported to the runtime bit-exactly.

## Licensing

- **Voice SDKs** (wake word, keyword spotting, ASR, VAD): the Kotlin, Swift, Python, Node.js, Go and C wrappers are Apache-2.0 in the open; the compiled runtime is a proprietary binary that is free to use and redistribute inside your product. Wake-word and keyword-spotting weights are ours and free to use with the SDK; the ASR weights are CC-BY-4.0, the VAD weights MIT.
- **LinnetTTS**: the whole runtime, bindings and tools are Apache-2.0 source. The weights are under the Linnet Weights License: free in your own products, commercial included, with the built-in voices; custom voices, fine-tuning, training on the outputs and general-purpose TTS services need a separate licence.

## What we sell

Models tuned to your product on the same runtime: wake phrases with your brand name and your languages, keyword sets, speech-to-intent, domain ASR, custom TTS voices. The open libraries above are the proof that the runtime ships; the custom work is what funds it.

Licensing, OEM integration, custom model packaging: **help@voxrt.com** · [voxrt.com](https://voxrt.com)

## How we work

- **CPU first.** A single ARMv8 core is the target. Accelerator paths are a return-on-investment question, not the foundation.
- **One binary across the CPU tier.** Universal NEON kernels, same code on a cheap A53 and a flagship X-series core; runtime feature detection is opt-in, not load-bearing.
- **Battery-aware by construction.** No allocations on the hot path, no `f64` where `f32` works, every kernel profiled against its audio budget before it ships.
- **Bit-exact or it does not land.** NEON, AVX2 and WebAssembly must match the scalar reference within the ULP budget; the TTS reproduces the same audio for the same seed on a given platform at any thread count.
- **Numbers with a method.** Every published figure names the device, the clock, the chunk size and the warm-up, and most come with a script.

## Targets

- **iOS** (iPhone, iPad, arm64), **macOS** (Apple silicon, for LinnetTTS)
- **Android** (arm64-v8a, x86_64 emulator)
- **Linux** x86-64 (Haswell and later) and aarch64 (glibc 2.17+): Raspberry Pi 3 / 4 / 5 / Zero 2, NVIDIA Jetson, AWS Graviton, Rock Pi / Orange Pi / Khadas boards
- **Browser** (WebAssembly SIMD128, no server)
- Microcontrollers and other targets on request

## Stack

Rust · ARMv8 NEON and x86-64 AVX2 intrinsics · WebAssembly SIMD128 · cbindgen · pyo3 / maturin · napi-rs · cgo · wasm-bindgen · cargo-ndk · Swift Package Manager · JitPack · PyPI · npm · Go modules · Xcode xcframework · GitHub Actions

---

*If you are putting voice into a product and the CPU budget, the battery or the privacy policy is the constraint, we want to talk.*
