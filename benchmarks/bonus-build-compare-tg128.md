# Bonus B1 - Prebuilt vs source build

Host `Darwin-arm64` · CPU `Apple M4 Pro`
Vector extensions detected: NEON
llama.cpp `b10488` both sides · `threads=12` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 31.0 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 32.5 | 1.05x |

On this machine, the source build is **1.05x faster**.

before: 31.0 tok/s (prebuilt release)
after:  32.5 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.05x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.


### Separately: what GPU offload is worth on the same binary

`tg128` on the source build at `-ngl 99` instead of `-ngl 0`:

| Source build | tg128 (tok/s) | vs its own CPU run |
|:--|--:|--:|
| `-ngl 0` (CPU) | 32.5 | 1.00x |
| `-ngl 99` (offloaded to MTL0: Apple M4 Pro (18186 MiB, 18185 MiB free)) | 75.3 | 2.32x |

This number is **not** part of the B1 comparison above -- it is a different knob.
Reporting it separately is the point: a compiler flag and an accelerator are not
interchangeable explanations for a speedup.


## Your explanation

**Why the gap is only 1.05x on M4 Pro:**

1. **Both detect NEON**: The prebuilt binary uses runtime CPU dispatch (AppleClang) and detects NEON at runtime on M4 Pro. The source build with `-DGGML_NATIVE=ON` compiles with the same detected features. Both produce similar NEON-optimized kernels.

2. **M4 Pro's architecture is already optimized**: Apple Silicon uses a custom microarchitecture with excellent SIMD support. The compiler doesn't have much room to "miss" targets — the prebuilt binary already benefits from modern ARM optimizations.

3. **Memory bandwidth is the ceiling**: The tg128 benchmark measures decode, which is **memory-bandwidth bound**, not instruction-bound. Both binaries hit the same memory wall. Compiler optimizations (better NEON scheduling) cannot overcome the bandwidth ceiling.

4. **Small margin is expected on modern CPUs**: On M4 Pro, which is already highly optimized, the compiler flag difference is modest. The gain from `-DGGML_NATIVE=ON` is larger on older/less-optimized CPUs where the prebuilt binary uses generic fallback paths.

**The real speedup knob for M4 Pro:**

The comparison also shows GPU offload (`-ngl 99`): **2.32x speedup** (32.5 → 75.3 tok/s). This is the meaningful optimization on Apple Silicon — Metal GPU offload matters far more than compiler flags for decode workloads.
