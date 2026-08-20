# Bonus B5 - MLX vs llama.cpp Metal

Host `Darwin-arm64` · arm ·
llama.cpp `b10488` · 10 prompts,
`max_tokens=64`, warm-up discarded on both sides

| Runtime | Weights | TTFT P50 (ms) | TTFT P95 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|
| llama.cpp (Metal) | `gemma-4-E2B-it-UD-Q4_K_XL.gguf` | 81.3 | 84.2 | 79.7 |
| MLX-LM | `unsloth/gemma-4-E2B-it-UD-MLX-4bit` | 78.2 | 141.7 | 108.6 |

MLX decode is **1.36x** llama.cpp Metal here. Faster on decode: **MLX**.

Both sides run the same model at 4-bit and stream token by token, so the gap is a
runtime difference, not a model difference. The quantization schemes are not
byte-identical, though (Unsloth Dynamic GGUF vs MLX 4-bit), so treat a gap under
~10% as noise rather than a finding.

## Your finding

**Would ship MLX on this Mac for local serving, llama.cpp for portability.**

| Criteria | llama.cpp (Metal) | MLX |
|----------|-------------------|-----|
| Decode speed | 79.7 tok/s | 108.6 tok/s | MLX wins |
| TTFT P50 | 81.3 ms | 78.2 ms | ~same |
| TTFT P95 | 84.2 ms | 141.7 ms | llama.cpp wins |
| Portability | Linux/Windows/Mac | Apple Silicon only | llama.cpp wins |
| Deployment | OpenAI-compatible API | Python API | llama.cpp wins |
| Startup time | Fast | Downloads weights first time | llama.cpp wins |

**Verdict by use case:**

1. **Local Mac development**: MLX wins
   - 36% faster decode (108.6 vs 79.7 tok/s)
   - Native Apple optimization
   - Great for prototyping

2. **Production serving**: llama.cpp wins
   - OpenAI-compatible API
   - Cross-platform (deploy anywhere)
   - Stable P95 (84ms vs 141ms)

3. **When off Apple hardware**: llama.cpp only
   - MLX only works on Apple Silicon
   - llama.cpp works on Mac, Linux, Windows

**Security note (from semantic cache):** Shared semantic/KV caches can leak prompts across users via timing side channels. Both runtimes need per-tenant cache isolation in production.
