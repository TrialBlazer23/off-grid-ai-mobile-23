# Plan: Performance quick wins (KV cache, RAG prompt placement, context ceiling, quant defaults, threads)

Five small, independent changes. Each is one PR. Together they attack the same measured bottleneck: on-device prefill is the slow path (decode is memory-bandwidth-bound; accelerator evidence in arXiv 2410.03613 and 2605.08195), and the app currently wastes prefill work every turn.

Status:
- [ ] QW-1 Remove the pre-turn KV cache wipe
- [ ] QW-2 Move RAG chunks out of the system prompt
- [ ] QW-3 Raise the context ceiling for 12-16 GB devices
- [ ] QW-4 Recommend Q4_0/IQ4_NL on Armv9 Android
- [ ] QW-5 Cluster-aware auto thread count

---

## QW-1: Remove the pre-turn KV cache wipe at 70% context usage (~1 day)

### Problem
`prepareContext` calls `llmService.clearKVCache(false)` before EVERY turn once `contextUsagePercent > 70` or anything was truncated. Every turn of a long chat then re-prefills the full history (2,900+ tokens) on CPU: multi-second time-to-first-token exactly when conversations get long. The same block calls `getContextDebugInfo` (a full-prompt tokenize over the bridge) on every turn even when the debug UI is off.

### Evidence
- The wipe and the unconditional `getContextDebugInfo` call: `src/screens/ChatScreen/useChatGenerationActions.ts:195-203`.
- The code's own design says not to do this: `manageContextWindow` is a documented no-op ("lets llama.rn's native ctx_shift handle overflow for KV cache reuse") at `src/services/llm.ts:340-343`, and `buildCompletionParams` defaults `ctx_shift` to true at `src/services/llmHelpers.ts:416`. Exception: `llm.ts:245-246` (`shouldDisableCtxShift`) disables ctx_shift on Android when GPU layers are active (OpenCL SIGSEGV workaround); on that configuration the compaction retry is the only overflow net after this change.
- A context-full compaction retry already exists as the safety net (triggered by the completion's `context_full` flag at `llm.ts:312`): `src/screens/ChatScreen/useChatGenerationActions.ts:205-227`.

### Change
1. Delete the pre-turn `clearKVCache` call; trust `ctx_shift` plus the existing compaction retry.
2. Gate `getContextDebugInfo` behind the `showGenerationDetails` setting.

### Caution
Verify on a real device that the wipe was not papering over a stale-KV-after-truncation bug: run a long chat past 70% usage before and after the change and compare outputs for corruption. Test BOTH configurations: default (ctx_shift on) and Android with GPU layers active (ctx_shift disabled, compaction retry is the only net).

### Tests
- Fails-before/passes-after: a long conversation past 70% context must not call `clearKVCache` between turns.
- Provit journey: long-chat conversation on device; record time-to-first-token on the turns past 70%.

---

## QW-2: Keep RAG chunks out of the system prompt (~1-2 days)

### Problem
`injectRagContext` appends per-query retrieved chunks to the SYSTEM prompt, which becomes `messages[0]`. Chunks differ on every query, so the prompt prefix changes at token 0 on every project-chat turn:
- llama.cpp loses all KV prefix reuse and re-prefills the whole history.
- LiteRT hits its `sysChanged` check and performs a full native `resetConversation` plus history re-prefill.

This is the largest recurring time-to-first-token tax in the app's main RAG use case.

### Evidence
- Chunks appended to the prompt that becomes the system message: `src/screens/ChatScreen/useChatGenerationActions.ts:228-250`.
- LiteRT reset on system-prompt change: `src/services/litert.ts:219-226`.

### Change
Keep the system prompt stable across turns (the doc list + tool hint already built separately at `useChatGenerationActions.ts:237-239` stays there) and attach the retrieved chunks to the CURRENT USER message instead.

`injectRagContext` has TWO call sites: `startGenerationFn` (`useChatGenerationActions.ts:287`) and `regenerateResponseFn` (line 454). Both paths must attach the chunks to the current user message, and the chunks must not persist into the stored user message: attach at context-build time (for example via the messageText substitution in `buildMessagesForContext`). Restructuring only the send path either reintroduces the LiteRT reset on every regenerate or silently drops RAG context from regenerates.

### Tests
- Unit: prompt assembly produces an identical system message across two RAG turns with different retrievals; chunks appear in the user message.
- Integration: with the LiteRT path, a second RAG turn must not trigger `resetConversation` (drive the real service with a stubbed native module and assert the reset is not invoked). Cover the regenerate path too.
- Provit journey: two consecutive knowledge-base questions in a project chat; the second turn's time-to-first-token must not include a full-history re-prefill.

---

## QW-3: Raise the context ceiling for 12-16 GB devices (~1-2 days)

### Problem
`getMaxContextForDevice` returns 8192 for ANY device over 8 GB, and the default-settings path additionally caps at `min(modelMax, 4096, deviceCap)`. A 16 GB phone running a 2.5 GB 4-bit model advertised with 262K context chats at 4096 tokens.

### Evidence
- Flat cap: `src/services/llmHelpers.ts:356-362` (>8 GB -> 8192).
- Default path: `src/services/llm.ts:212` (`targetCtx = Math.min(modelMax, 4096, deviceMaxCtx)`) and `maxContextLength: 4096` at `src/constants/index.ts:112`.
- The safety net that makes this low-risk already exists: `resolveSafeContext` steps down from any requested size (`src/services/llm.ts:94-125`), `checkMemoryForModel` estimates KV cost, and the app defaults to q8_0 KV cache + flash attention (`src/stores/appStore.ts:189-190`).

### Change
Add tiers to `getMaxContextForDevice` (example: >8 GB -> 8192 stays; >=12 GB -> 16384; >=16 GB -> 32768) and raise the default-path cap to follow the device tier. Weights dominate RSS (a 7B Q4 model is ~3.8 GB); q8_0 KV for a 4B model at 16K context is well under 1 GB.

### Tests
- Unit: tier boundaries for 8/12/16 GB inputs (fails before: 16 GB returns 8192; passes after: 32768).
- Integration: default-settings path for a 16 GB device requests the tier value and `resolveSafeContext` still steps down when memory is constrained.
- Provit journey: load a long-context model on a 16 GB device and confirm the loaded context size matches the new tier in the generation details.

---

## QW-4: Recommend Q4_0/IQ4_NL on Armv9 Android in the quant picker (~2-3 days)

### Problem
`QUANTIZATION_INFO` marks Q4_0 "recommended: false" and steers users to Q4_K_M, yet the loader already special-cases repackable quants (`['q4_0','iq4_nl']`, mmap disabled on Android so Armv9-repacked weights can allocate). Measured evidence: Q4_0 gains up to 6x prefill from Armv9 i8mm/sdot runtime repacking, with accuracy on par with Q4_K_M (arXiv 2605.08195, 2410.03613). The loader knows; the catalog does not.

### Evidence
- Catalog: `src/constants/models.ts` around line 157 (Q4_0 `recommended: false`; Q4_K_M/Q4_K_S `recommended: true`).
- Loader: `REPACKABLE_QUANTS` handling at `src/services/llmHelpers.ts:19-24` (Android-only mmap handling).

### Change
Make the recommendation platform-aware through ONE seam: a `hardwareService` capability (for example `supportsInt8MatmulRepack`) consumed by the quant picker. Detection mechanism: parse the `Features` line of `/proc/cpuinfo` (match `i8mm`, plus `asimddp` for sdot) inside `hardwareService`, mirroring the existing `/proc/cpuinfo` read in `getCpuCoreCount` (`src/services/hardware.ts:425-431`); default to false on read failure and on iOS. On the capability, Q4_0 becomes the recommended chip with proof-first copy (for example "up to 6x faster prefill on this phone's CPU", per `docs/brand_tone_voice.md`); Q4_K_M stays recommended elsewhere. No `Platform.OS` branch in any component; the capability is data.

### Tests
- Unit: the capability flag derivation from mocked `/proc/cpuinfo` contents (i8mm present, absent, unreadable).
- Integration: the flag flows from `hardwareService` through to the picker rendering (both states, driving the real picker logic).
- Provit journey: open the quant picker on an Armv9 Android device and confirm Q4_0 carries the recommendation chip.

---

## QW-5: Cluster-aware auto thread count (~2-3 days)

### Problem
The auto sentinel (`nThreads = 0`) resolves to `floor(cores * 0.8)` = 6 on the 8-core SoCs both reference devices use, spilling decode threads onto Cortex-A510/A520 efficiency cores. The codebase documents that this is wrong in its own comment ("targets performance cores only; over-threading onto efficiency cores (A520) hurts"). Decode is memory-bandwidth-bound, so little-core threads add synchronization overhead without adding bandwidth.

### Evidence
- Formula: `src/services/hardware.ts:432-435`.
- Contradicting comment: `src/services/llmHelpers.ts:14`.
- Measurement seam for before/after: `lastDecodeTokensPerSecond` stats at `src/services/llmHelpers.ts:419-437`.

### Change
On Android, read `/sys/devices/system/cpu/cpufreq/policy*/cpuinfo_max_freq`, count cores in the top frequency clusters (S23 Ultra: 1x X3 + 4x A7xx = 5), and return that count. Keep the current formula as the fallback when sysfs is unreadable. Keep iOS behavior unchanged. The logic lives inside `hardwareService` (the existing seam); callers keep calling `getRecommendedThreadCount()`.

### Tests
- Unit: mocked sysfs values for S23 Ultra-like and Nothing Phone 3-like topologies produce the expected counts; unreadable sysfs falls back to the current formula.
- Provit journey (doubles as the fails-before/passes-after evidence): the same model and prompt on device, auto thread count before vs after, compared via `lastDecodeTokensPerSecond`.
