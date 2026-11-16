# Pocket Antigravity: Architecture

Offline, on-device AI coding agent for low-end Android (target: 2 GB RAM, ARM64).
This document covers the pre-code deliverables: architecture, hardware limits, stack, structure, memory budget, risks.

## 1. Proposed architecture

```
┌──────────────────────── UI (Jetpack Compose, single Activity) ───────────────────────┐
│ Chat · Model Manager · Project Explorer · Code Viewer · Activity · Planner · Logs · Settings │
└───────────────┬──────────────────────────────────────────────────────┬────────────────┘
                │ StateFlow / events                                   │
┌───────────────▼───────────────┐                      ┌───────────────▼───────────────┐
│ Agent Runtime (Kotlin)        │                      │ Device layer                  │
│  Planner → Context Engine     │                      │  DeviceDiagnostics (Phase 1)  │
│  Tool Engine (validated)      │                      │  MemoryMonitor     (Phase 1)  │
│  Execution Engine             │                      │  ModelFitEstimator (Phase 1)  │
│  Verification Engine          │                      │  Storage/SAF access (Phase 4) │
│  Bounded loop + task state    │                      └───────────────────────────────┘
└───────────────┬───────────────┘
                │ Kotlin interface `InferenceEngine` (suspend/Flow<Token>, cancel())
┌───────────────▼───────────────┐
│ llama.cpp via JNI (Phase 2)   │  CPU only, GGUF, mmap weights, streaming, cancel flag
└───────────────────────────────┘
```

Key decisions (each is an ADR; add new ones to the log in section 8):

- **ADR-1 Native Kotlin + Compose, single Activity.** No WebView, no cross-platform runtime. Lowest RAM overhead and direct JNI access.
- **ADR-2 Engine behind an interface.** The agent never touches JNI directly; this keeps the agent loop unit-testable on the JVM with a scripted engine. Test engines are allowed only in tests; the shipped app has no simulated inference.
- **ADR-3 Diagnostics before model choice.** Model fit is computed from real device numbers; the app refuses or warns instead of attempting a load that will be killed.
- **ADR-4 Weights are mmap'd, but counted as fully resident.** Every token touches nearly all weight pages, so a mapped file under memory pressure thrashes. The estimator budgets the full file size.
- **ADR-5 Workspace access only through Storage Access Framework.** No MANAGE_EXTERNAL_STORAGE, no root.
- **ADR-6 Phase 1 has zero native code and almost no dependencies.** Compose + AndroidX only.

## 2. Hardware limitations (2 GB phone)

- Of 2 GB, Android and the system UI typically leave roughly 600–900 MB "available"; this fluctuates. The app must read it live, not assume it.
- The low-memory killer targets background and then foreground processes; a native allocation spike kills the app with no exception to catch. Pre-flight estimation is therefore the primary protection, not try/catch.
- CPU: usually 4–8 small ARM cores. Expect low single-digit tokens/s for ≤1B models at Q4. Prompt processing (prefill) is slower per token than generation on weak chips, so a large retrieved-code prompt can take tens of seconds.
- Storage: 16–32 GB eMMC, often mostly full. GGUF files are 0.3–1 GB each; the model manager must check free space before import.
- Thermal throttling during long generations will reduce speed further.
- Realistic model range: ≤0.5B at Q4/Q5 is the comfortable zone; 1B at Q4 is **borderline** and may only work with a small context and nothing else in memory. 1.5B+ should be expected to not fit. Final answers come from the estimator plus a real load test on the device, not from this table.

## 3. Android technology stack

| Concern | Choice |
|---|---|
| Language | Kotlin 2.0.x (C++17 for the llama.cpp bridge in Phase 2) |
| UI | Jetpack Compose + Material 3, dark theme default |
| Build | Gradle (Kotlin DSL), AGP 8.7, compileSdk 34, minSdk 26, ABI filter arm64-v8a |
| Native | NDK + CMake, llama.cpp vendored as a git submodule pinned to a commit (Phase 2) |
| Storage | SAF (`ACTION_OPEN_DOCUMENT_TREE`), app-private files for task state (JSON) |
| Persistence | Plain JSON/files; no Room, no vector DB (RAM) |
| Tests | JUnit4 on JVM for pure logic; instrumented tests on real hardware for native/IO |

minSdk 26 keeps Java NIO and modern ActivityManager APIs available while still covering essentially all ARM64 phones in use.

## 4. Project structure

```
PocketAntigravity/
├── ARCHITECTURE.md
├── settings.gradle.kts / build.gradle.kts / gradle.properties
└── app/
    ├── build.gradle.kts
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   └── java/com/pocketantigravity/
        │       ├── MainActivity.kt
        │       ├── device/        DeviceDiagnostics, MemoryMonitor, CpuFeatures   [Phase 1]
        │       ├── model/         ModelFitEstimator                               [Phase 1]
        │       ├── ui/            theme + DiagnosticsScreen                       [Phase 1]
        │       ├── llm/           InferenceEngine, JNI bridge                     [Phase 2]
        │       ├── agent/         planner, tools, executor, verifier              [Phase 5-6]
        │       └── workspace/     SAF access, indexer, search                     [Phase 4]
        └── test/java/com/pocketantigravity/   JVM unit tests
```

## 5. Expected memory consumption (estimates, to be validated on-device)

Formula used by `ModelFitEstimator`:

```
required = weights (GGUF file size)
         + KV cache  = 2 · n_layer · n_ctx · n_embd_kv · 2 bytes   (f16)
         + compute buffers   ≈ 96 MiB
         + runtime overhead  ≈ 180 MiB   (ART heap, Compose, JNI, allocator)
budget   = availMem − system lowMemory threshold
```

Worked examples at n_ctx = 1024 (KV numbers from published model configs):

| Model (Q4) | Weights | KV @1024 | + 276 MiB fixed | ≈ Required |
|---|---|---|---|---|
| ~0.5B (24 layers, kv-dim 128) | ~400 MB | 12 MiB | 276 MiB | ~690 MB |
| ~1B (16 layers, kv-dim 512) | ~800 MB | 32 MiB | 276 MiB | ~1.1 GB |
| ~1.5B (28 layers, kv-dim 256) | ~1.0 GB | 28 MiB | 276 MiB | ~1.3 GB |

KV = 2·layers·ctx·kv-dim·2 B, e.g. 0.5B: 2·24·1024·128·2 = 12 MiB. A 1B model at ~1.1 GB against a typical 600–900 MB budget is why the spec says never to assume 1B fits: the estimator returns WILL_NOT_FIT for it on most 2 GB devices, and the UI says so. These are estimates; the model configs are approximate published values.

## 6. Technical risks

1. **Native OOM kill** with no catchable exception. Mitigation: pre-flight estimate, conservative budget, `onTrimMemory` unload (Phase 7), small default context.
2. **Model quality at ≤1B.** Small models are unreliable at structured tool calling. Mitigation: constrained output (GBNF grammar for tool-call JSON), strict argument validation, bounded retries, and honest reporting when the model can't do the task. This may cap real autonomy on 2 GB; it will be measured, not assumed.
3. **Prefill latency** on weak CPUs with long prompts. Mitigation: small retrieved snippets, compact project summaries, prompt caching of the system prompt.
4. **Terminal/test execution on stock Android.** Without root there is no `git`, compiler, or test runner in a sandboxed app. `run_tests` and git checkpoints can only work for what is actually available (for example, a user-installed Termux toolchain is *not* reachable from our sandbox). Phase 5–6 will state this limitation in-app instead of faking results; app-side checkpoints will be file snapshots stored in app-private storage.
5. **SAF performance.** `DocumentFile` queries are slow on big trees. Mitigation: incremental, cancellable indexing with a bounded in-memory tree.
6. **Background limits.** Android may kill a background app mid-task. Mitigation: persist task state after each step; resume rather than assume continuity.
7. **Thermal throttling** inflating generation time; surfaced via the tokens/s display.
8. **llama.cpp API churn.** Mitigation: pin the commit, isolate usage in one C++ file.

## 7. Phase plan and exit criteria

Phase 1 exit: project builds, diagnostics screen shows real device values, estimator unit tests pass, on-device run recorded.
Later phases follow section 9 of the brief; none starts before the previous phase's tests are reported.

## 8. ADR log

| # | Decision | Date |
|---|---|---|
| 1-6 | See section 1 | 2026-11-17 |
