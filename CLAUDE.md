# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Off Grid AI: a React Native 0.83 app (iOS + Android) that runs AI fully on-device — GGUF text/vision chat via llama.rn (plus Google AI Edge LiteRT on Android), Stable Diffusion image generation, Whisper STT, RAG, tool calling/MCP, and remote OpenAI-compatible servers. The free core is MIT; paid Pro features live in the `pro/` submodule.

## Commands

```sh
npm start                                   # Metro
npm run ios                                 # iOS simulator
npm run ios:device                          # physical iPhone via manual signing (env: IOS_DEVICE_ID, IOS_PROFILE, IOS_TEAM)
npm run android                             # Android debug build (appId ai.offgridmobile.dev)

npx tsc --noEmit                            # typecheck
npx eslint .                                # JS/TS lint (npm run lint adds gradle lintDebug + swiftlint)

npx jest                                    # all JS tests (npm test adds --coverage AND the native android+ios suites)
npx jest __tests__/unit/utils/foo.test.ts   # single test file
npx jest path/to/file.test.tsx -t "name"    # single test by name
npm run test:android                        # gradle :app:testDebugUnitTest
npm run test:ios                            # xcodebuild test, iPhone 16e simulator
npm run test:e2e                            # Maestro flows in .maestro/flows/p0 (needs booted sim/emulator with the app installed)
maestro test .maestro/flows/p0/02-text-generation.yaml    # single e2e flow
```

- Jest coverage threshold is 80% global (also Codecov's project/patch target), enforced only when `--coverage` is passed — single-file runs won't trip it. `DEBUG_JEST_CONSOLE=1` un-suppresses console output in tests.
- `npm install` runs `patch-package`; `patches/` carries required native fixes (whisper.rn crash guards, background-downloader threading, zip-archive, etc.).
- Dead-code check: `npx knip`. Release pipeline: `scripts/release.sh`.

## Repository Layout

**All Pro feature code lives in the `pro/` submodule (its own git repo, `@offgrid/pro`) — not in core.** When changing or adding a Pro feature (e.g. TTS/audio, MCP/tools, and other paid surfaces), edit files under `pro/` and commit/PR them in that repo. Core only wires Pro in through the slot/hook registries; it never imports Pro code directly. Pro changes are a separate branch + PR from core (see `pro/CLAUDE.md`).

## Architecture Overview

Depth lives in `docs/ARCHITECTURE.md`; subsystem designs in `docs/design/` (`MODEL_DOWNLOAD_SERVICE.md`, `MODEL_ROUTING.md`, `AUDIO_PLAYBACK_SERVICE.md`) and `docs/standards/CODEBASE_GUIDE.md`.

### Service → store → view (the one pattern everything follows)

A plain-TS singleton service owns each subsystem's state machine, resources, and side effects; a zustand store (`src/stores/`) is a read-only projection for rendering; views/hooks dispatch intents to the service. Each machine logs `[*-SM]` transition traces (see Device Logs below):

| Tag | Owner | Owns |
|---|---|---|
| `[MODEL-SM]` | `src/services/activeModelService/` | the ONLY place models load/unload (text + image), engine-aware |
| `[MEM-SM]` | `src/services/modelResidency/` | RAM budget, eviction, the global FIFO `runExclusive` load lock, memory-warning response |
| `[DL-SM]` | `src/services/modelDownloadService/` | ALL downloads (text/image/stt/tts) via per-domain `DownloadProvider`s; UI is gated by `DownloadCapabilities` flags (`modelDownloadService/types.ts`) |
| `[GEN-SM]` | `src/services/generationSession.ts` | which conversation is generating (the streaming engine itself is `generationService.ts` + `generationToolLoop.ts`) |
| `[IMG-SM]` | `src/services/imageGenerationService.ts` | image-gen phase machine (`idle→enhancing→loading→generating→saving→…`); `phase` is the only stored truth |
| `[ROUTE-SM]` | `src/services/intentClassifier.ts` | image-vs-text intent (cache → regex → optional LLM via the auto-provisioned SmolLM2 classifier, `classifierProvisioning.ts`) |
| `[FAIL-SM]` | `src/services/modelFailureHandler.ts` | turns any model failure into the one dismissible card (`modelFailureStore`) |
| `[TTS-SM]` | `src/services/audioSessionManager.ts` | iOS AVAudioSession category/activation (TTS engines themselves live in Pro) |

### Inference engines
- Text/vision: llama.rn (GGUF) wrapped by `src/services/llm.ts`; Android-only LiteRT in `src/services/litert.ts`. `src/services/engines.ts` (`getActiveEngineService()`) dispatches the operations both engines support.
- Image: `src/services/localDreamGenerator.ts` bridges via `Platform.select` to `CoreMLDiffusionModule` (iOS, Core ML/ANE) or `LocalDreamModule` (Android, MNN CPU / QNN NPU); orchestrated by `imageGenerationService.ts`.
- STT: whisper.rn (`src/services/whisperService.ts`). Remote OpenAI-compatible servers: `remoteServerManager.ts` + `src/services/providers/`. RAG: `src/services/rag/` — the only op-sqlite consumer (`rag.db`).
- Model discovery: HuggingFace API + curated catalog — `src/services/modelManager/`, `src/constants/models.ts`.

### Native modules (classic bridge, not codegen TurboModules)
TS contracts are declared inline in the consuming service (`NativeModules.X` + interfaces; shared payload types in e.g. `src/services/backgroundDownloadTypes.ts`). Cross-platform modules keep identical JS names/methods on both sides: `DownloadManagerModule` (iOS URLSession — `ios/DownloadManagerModule.swift`; Android WorkManager + Room — `android/app/src/main/java/ai/offgridmobile/download/`), `DeviceMemoryModule`, `PDFExtractorModule`. Platform-specific backends (`CoreMLDiffusionModule` / `LocalDreamModule`) are reconciled inside ONE JS service; `LiteRTModule` is Android-only. Android modules register in `MainApplication.kt`.

### Pro wiring (core never imports Pro)
`metro.config.js` aliases `@offgrid/pro` to the `pro/` submodule only if `pro/package.json` exists, else to the null stub `src/bootstrap/proStub.js`; `jest.config.js` mirrors this and skips pro-dependent suites (audio/engine/TTS/MCP) when the submodule is absent. `src/bootstrap/loadProFeatures.ts` requires the package, checks the entitlement, and calls `pro.activate({...registration callbacks})`. The seams: `src/bootstrap/slotRegistry.ts` (UI slots), `src/bootstrap/hookRegistry.ts` (behavior hooks, e.g. `audio.speak`), `src/navigation/screenRegistry.ts` (Pro screens — the `McpServers` screen's presence is the app-wide "Pro is active" signal, see `useIsProActive`), `src/components/settings/sectionRegistry.ts`, and `src/services/tools/extensions.ts` (tool extensions over the built-ins in `tools/registry.ts`). Licensing is Keygen license keys cached in the Keychain (`proLicenseService.ts`, `keygenClient.ts`); the RevenueCat SDK is never called in core — purchase happens on the web.

### Persistence
Chats/conversations use zustand `persist` → AsyncStorage (`chatStore.ts`), as do the app/project/remote-server/whisper/auth stores. op-sqlite is ONLY for RAG. In-flight downloads live natively (Room DB on Android, URLSession state on iOS) and are rehydrated into the non-persisted `downloadStore` at launch.

### App init order (`App.tsx` `initializeApp`) — order matters
appStore hydration → download-store hydration from native → reattach download recovery → register core download providers → model manager init + list refresh → remote-server store hydration before provider init → auth/RAG → `checkProStatus()` → `loadProFeatures()`. Models are intentionally NOT loaded at boot; they load lazily on first use.

### UI
Root native stack wrapping a 5-tab bottom-tab navigator, all in `src/navigation/AppNavigator.tsx`. Design tokens are split across two places: COLORS/SHADOWS come from `useTheme()`/`useThemedStyles()` in `src/theme/` (palettes in `src/theme/palettes.ts`); TYPOGRAPHY/SPACING/FONTS come from `src/constants/index.ts`.

### Tests
Centralized under `__tests__/` (not colocated with src): `unit/` and `integration/` mirror `src/`, plus `rntl/` (render tests), `contracts/` (native-module contract tests), `hardening/` (regression batches). Module aliases `@/*` and `@offgrid/core/*` → `src/*`.

## Device Logs (how to see what's actually happening on the device)

**RN 0.83 moved JS `console.log` off the Metro terminal into React Native DevTools, and RN's console never reaches the iOS device syslog.** So `metro` stdout, `idevicesyslog`, and `npx react-native log-ios` (simulator-only) all capture NOTHING from a physical device. Do not waste time tailing Metro for app logs.

Instead, a **dev-only persistent file sink** (`src/utils/debugLogFile.ts`, wired in `App.tsx` behind `__DEV__`) mirrors every `logger.*` line — which is where ALL the state-machine traces go (`[TTS-SM]`, `[GEN-SM]`, `[MODEL-SM]`, `[DL-SM]`, `[ROUTE-SM]`, `[IMG-SM]`, `[MEM-SM]`, `[FAIL-SM]`) — into a file in the app container. Pull it over the cable to read the real trace:

```sh
xcrun devicectl device copy from \
  --device 00008150-000225103CD8C01C \
  --domain-type appDataContainer --domain-identifier ai.offgridmobile \
  --source Documents/offgrid-debug.log --destination /tmp/offgrid-debug.log
```

Then `grep`/read `/tmp/offgrid-debug.log`. The file appends a `===== session start … =====` marker on each launch and is size-capped (rotates, keeping the tail). The in-app **Debug Logs** screen (Settings → Debug Logs) shows the same lines live for quick visual checks. **When diagnosing a device issue, pull this file rather than guessing.**

## Branch Policy

**Never push directly to `main`.** All changes must go through a pull request:

0. Always create a branch specific to the change before committing: `feat/`, `fix/`, `docs/`, `chore/`, `test/`, etc.
1. Push the branch and open a PR — never `git push origin main`.
2. If you find yourself on `main`, create a branch first: `git checkout -b <branch-name>`.

## Copy & Content Standards

**Any change to website copy, essays, docs text, UI strings, or marketing content must follow the brand voice guide:**

- Read `docs/brand_tone_voice.md` before writing or editing any copy.
- The full quality checklist is at the bottom of that file — run every item before committing content changes.

Key rules that are easy to miss:

| Rule | Wrong | Right |
|---|---|---|
| Proof-first | "fast" | "15-30 tok/s on flagship devices" |
| Privacy as mechanism | "we value your privacy" | "the model runs in your phone's RAM, nothing is sent anywhere" |
| No exclamation marks | "It works!" | "It works." |
| No em dashes | "private — always" | "private - always" |
| No forbidden words | revolutionary, seamlessly, empower, leverage, robust, comprehensive, crucial, pivotal, delve, tapestry, testament, underscore, foster, cultivate, showcase, enhance | use specific, plain words instead |
| No AI slop phrases | "serves as", "stands as", "represents a", "marks a turning point", "it is worth noting" | just say "is" |
| No structural clichés | "Not just X, but Y" / "It's not X, it's Y" | state the thing directly |
| No curly quotes | "private" | "private" |

The emotional arc for all content: **Recognition -> Return -> Freedom**. Name what's been happening, show what's being given back, hand over the capability without condition.

---

## Design Standards

**Any change that touches UI (screens, components, styles) must comply with the design system:**

- Read `docs/design/VISUAL_HIERARCHY_STANDARD.md` before writing or modifying any UI code.
- Check `docs/design/` for any other relevant design documents.
- Use `TYPOGRAPHY` tokens — never hardcode font sizes or weights.
- Use `COLORS` tokens — never hardcode color values.
- Use `SPACING` tokens — never hardcode margin/padding values.
- Weights must stay ≤ 400 (no bold).
- Never use emojis or emoticons in UI text — always use `react-native-vector-icons` instead. Feather is the default; MaterialIcons is allowed only when Feather lacks a suitable icon (e.g. `whatshot` for trending).
- Never use `lucide-react` or any other icon library — only `react-native-vector-icons`.
- Follow the 5-category text hierarchy: TITLE → BODY → SUBTITLE/DESCRIPTION → META.

## Reuse Before Building

**Before writing any new component, style, hook, or service, search for an existing one and reuse it.** Building a parallel version of something that already exists creates visual and behavioural drift (e.g. a search box that looks different from every other search box).

- For UI: grep `src/components/` and the relevant screen folder for an existing component or shared style (e.g. `ModelCard`, `Card`, `Button`, shared `searchContainer`/`searchInput` styles) before creating your own. Two screens that show the same kind of thing must use the same component.
- For logic: check for an existing hook/service/store action (`grep -rn`) before adding a new one.
- If an existing component is close but not exact, extend it with a prop rather than forking a copy.
- Only build new when nothing fits — and say so in the PR description.

## Architecture & Abstractions (SOLID)

**Design to abstractions, not concrete implementations.** When there are multiple interchangeable implementations of a thing (TTS engines, model backends, providers, storage), the rest of the app must depend on a single interface/service layer — never branch on a concrete type.

**Before every code edit, stop and ask three questions — out loud, in the response:**

1. **Is there enough here to abstract?** Two or more concrete cases handled by the same caller (text vs vision vs image models, Slack vs Mail surfaces, kokoro vs piper TTS) means there's a seam. One case, used once, is not — don't abstract speculatively (YAGNI).
2. **Can we apply SOLID here?** Mainly: does one thing own one responsibility (SRP), and do callers depend on an interface rather than the concretes (DSP)? A `kind === 'x'` / `instanceof` / per-type `switch` in a caller — *especially in the renderer* — is the tell that the decision belongs behind a service.
3. **Are we actually using it?** A mapping or rule must be defined ONCE and reused. If the same kind→modality map, the same routing `if`, or the same capability check appears in two layers (e.g. main process AND renderer), that's duplication, not abstraction — collapse it to a single source of truth and have both sides call it.

If the answer to 1 is "no", say so and write the simple version. If "yes", build the seam before piling on the second concrete branch — retrofitting after drift is the expensive path.

- **No leaking implementation details upward.** UI and stores must not do `instanceof SpecificEngine`, check `engineId === 'kokoro'`, or branch on capabilities to decide *how* to do something. Push that decision behind the abstraction (the engine/provider implements it; or a service layer dispatches once). If you find yourself writing `if (engine X) … else …` in a component, the abstraction is wrong.
- **Single uniform entry point.** Prefer one polymorphic method (e.g. `engine.play(text, opts)`) that every implementation satisfies over several mechanism-specific methods (`speak` vs `playFromFile`) that callers must choose between.
- **Service layer between UI and implementations.** Implementations (engines/adapters) are swappable; a service abstracts them and exposes a normalized API + state. Adding a new implementation must require zero changes to UI/store.
- **Dependency Inversion / Liskov:** any implementation must be substitutable through the interface without callers knowing which one is active. Normalize gaps (e.g. an engine that can't report playback position) inside the service, not in the UI.
- Apply the rest of SOLID: single responsibility per module, open for extension (add an implementation) / closed for modification (don't touch callers), segregated interfaces (don't force implementations to stub methods they can't support — model that with the abstraction).
- **Think from first principles and keep a reference architecture in mind.** Before changing a subsystem, know its intended shape: what owns which state and resources, and how the pieces compose. Make changes consistent with that architecture.
- **Fix the seam — never patch around a missing abstraction.** When a subsystem has shared state or resources spread across multiple implementations (e.g. audio playback: the iOS AVAudioSession + AudioContext lifecycle + playback state across the streaming-TTS / file-player / PCM-replay paths), build/extend the *single owning service* and route everything through it. Do NOT add gates, guards, or flags in callers/UI/stores to compensate for the missing owner. Point-patches layered on shared mutable state cause cascading regressions — one fix silently breaks another path — and the subsystem becomes chaotic and flaky. If the owning abstraction doesn't exist yet, that's the work: create it, then migrate every path onto it with no bypass.
- **Migrations to an owning abstraction MUST be backward-compatible / behavior-neutral for existing paths.** When you route existing code through a new service, preserve its exact prior behavior — the refactor should be *additive* (it may fix a missing case), never change a behavior callers depended on. Example: the old TTS/recorder paths re-activated the iOS AVAudioSession on *every* call; making the new session owner "idempotent" silently dropped that re-activation and broke TTS. Verify each migrated path behaves exactly as before, then layer the fix on top.
- **Reactive stores are for UI projection — NOT for coordinating side-effects or owning resources.** Zustand/reactive state is the right tool for rendering; it is the wrong source of truth for imperative coordination (audio session/context, model loads, playback control, any hardware/resource). Most of the audio flakiness came from making imperative decisions (play vs block, which session category) by branching on a reactive store snapshot that several code paths write and desync. Follow a clear presentation separation (MVVM/MVP): the **Service/Model** owns the authoritative state machine + resources + side-effects; the reactive store is a **thin read-only projection** of that service; the **View** observes the projection and dispatches *intents* to the service. Never make an imperative decision (or fire a side-effect) by reading a reactive snapshot that multiple writers can mutate — that is the recipe for the desync/race bugs.
- **State and data MUST NOT live in the presentation layer.** A screen/component/hook (the View) holds NO authoritative state, NO business logic, and NO side-effecting data operations — it observes a service's projection and dispatches intents. Concretely: no retry/cancel/delete/finalize logic, no platform-branched mechanism, no store-mutation orchestration, no "compute the real value from several sources" in a screen or a `useXxxScreen`/`useXxxManager` hook. That logic belongs in the owning **service** (which carries the state machine + permanent logs). If a UI hook is doing the work instead of calling a service, that is the bug — move the work into the service and have the hook delegate. (This is why download retry/remove moved out of `useDownloadManager`/`retryHandlers` into `ModelDownloadService` + its providers.)

## Platform Abstraction (no iOS-only / Android-only bugs)

**A platform-specific bug is the symptom of a leaked platform detail.** With the right abstraction every bug is catchable on both platforms at once — that is the goal. We are writing ONE common layer, not two parallel apps.

- **One typed TS contract per native capability; both Swift and Kotlin must satisfy it.** Downloads, audio session, model load, image gen, STT — each has a single interface the JS calls. A method that exists on one platform but not the other is a contract violation, not an acceptable difference. Make the missing method a *compile error* (the TS interface requires it), never a runtime `"only available on Android"` throw.
- **Never branch on `Platform.OS` to decide HOW to do something.** Branching to choose a *mechanism* (which download path, which retry strategy, which audio setup) is the missing-abstraction smell — push that decision into the native module / a service that dispatches once. Branching for a genuine presentation value (a keyboard event name, a style inset) is fine.
- **Genuine OS capability gaps are declared DATA, not silent divergence.** When one platform truly can't do something (iOS URLSession dies on app-kill while Android WorkManager survives; an engine can't cancel), model it as a capability flag on the object (like `DownloadCapabilities`), normalize the gap ONCE inside the service, and let the UI render from the flag. The gap is then testable — never an `if (ios)` scattered through callers.
- **Contract tests run against the abstraction, so they catch both platforms.** Test the common interface + the capability flags; a single test then guards iOS and Android together. If a test can only be written per-platform, the abstraction is wrong.
- **Native module contract parity is mandatory.** The Swift and Kotlin implementations of a module must expose the SAME method names, the SAME events (names + payloads), and the SAME semantics (persistence, cleanup, error cascading). Contract drift between Swift and Kotlin is the root cause of platform-only bugs — when you touch a native module on one platform, verify/mirror the other side against the shared TS contract.

## Pre-Push Quality Gates

Quality gates run automatically via Husky on `git push` (`.husky/pre-push` — there is no pre-commit hook), scoped to the file types changed in the push range:

| Changed file type | Checks that run automatically |
|---|---|
| `.ts` / `.tsx` / `.js` / `.jsx` | eslint (changed files only), `tsc --noEmit`, `npx jest --findRelatedTests` |
| `.swift` | swiftlint (changed files only), `npm run test:ios` |
| `.kt` / `.kts` | `compileDebugKotlin` (type check), `lintDebug`, `npm run test:android` |

If anything changed at all, `npm run sonar` also runs (it skips itself when `SONAR_TOKEN` is unset).

**Requirements:**
- SwiftLint: `brew install swiftlint` (skipped with a warning if not installed)
- Android checks require the Gradle wrapper in `android/`

Before writing new code, ensure tests exist for your changes. If the hook fails, fix the issue and push again — never skip with `--no-verify`.

## Testing Requirements

Always write **both** unit tests and integration tests for new features and significant changes:

- **Unit tests** (`__tests__/unit/`): Test individual functions, hooks, and store actions in isolation with mocked dependencies.
- **Integration tests** (`__tests__/integration/`): Test how multiple modules work together end-to-end (e.g., service A calls service B which writes to database C). Use mocked native modules but real logic across layers.

Do not consider a feature complete with only unit tests. Integration tests catch wiring bugs, incorrect data flow between layers, and lifecycle issues that unit tests miss.

**Use mocks very sparingly — a green suite must mean the real thing works, not that a mock returned what it was told.** Mock only what you genuinely cannot run in the test environment (native modules, the network, the device clock). Everything else — the service under test, the stores it writes, the logic across layers — runs for real. A test that mocks the very thing it is asserting (so it would pass even if the implementation were deleted) is worse than no test: it hides the broken behaviour behind a false green. Prefer driving the real class/store/reducer and asserting the observable outcome. When you must stub a boundary, keep the stub dumb (return plain data) and let the real logic on top of it do the work. If a behaviour can only be proven by mocking out the behaviour, that is the signal to test it at a higher layer (integration) or on-device (Provit) instead.

**Design to SOLID with real abstraction layers (not incidental ones).** These are the same rules as the Architecture section above, restated as a standing expectation for every change: one responsibility per module (SRP); callers depend on an interface/service, never on a concrete implementation or a `kind===`/`instanceof`/`Platform.OS`-mechanism branch (DIP); a new implementation (engine, provider, backend) drops in behind the existing seam with zero caller changes (OCP); any implementation is substitutable through the interface (LSP); interfaces are segregated so an implementation never stubs methods it can't support. The abstraction layer must be a genuine owning seam — a service that owns the state machine, resources, and side-effects — not a thin pass-through that leaks the concretes upward. If a fix would add a second concrete branch in a caller, build/extend the seam instead.

**Test every approved behavior change in the same pass.** When iterating (a request, a fix, a tweak you just confirmed), add a test that captures that specific behavior as part of the same change — a regression test that would fail before the change and pass after. This applies to bug fixes (test the exact broken case), new branches/conditions (cover each one), and copy/contract changes that other code or tests depend on. Do not defer tests to "later" or to a separate commit. Then run `npx tsc --noEmit && npm test` and fix any failures before reporting the change done.

## Push = Create PR + Address Review

When the user says "push" (or any equivalent like "ship it", "send it", "push this"), follow this full workflow:

### Before pushing
0. Write tests for any new or changed logic if they don't already exist.
1. Run `npm run lint && npx tsc --noEmit && npm test` — fix any failures before continuing.
2. Commit all staged changes with a descriptive message.
3. Ensure you are NOT on `main`. If you are, create an appropriately named branch first: `git checkout -b feat/...` or `fix/...` or `chore/...` etc.

### Pushing & PR
4. Push the branch: `git push -u origin <branch>`
5. If no PR exists for this branch, create one with `gh pr create`. **Do NOT include "Generated with Codex" or any AI attribution in PR descriptions.**
6. If a PR already exists, update its description to reflect **all commits in the PR** (not just the latest push). Read the full commit history with `git log main..HEAD` and write a coherent description that summarises the entire change set — what it does, why, and how.

### Review loop
7. Wait for Gemini to review the PR (poll with `gh pr checks` and `gh api repos/{owner}/{repo}/pulls/{number}/reviews` until a review appears).
8. Pull down review comments: `gh api repos/{owner}/{repo}/pulls/{number}/comments` and `gh api repos/{owner}/{repo}/pulls/{number}/reviews`.
9. Address every review comment — fix the code, re-run quality gates (lint, tsc, test).
10. Reply to **each** review comment individually using `gh api` (`/pulls/comments/{id}/replies`). Every comment gets its own reply — do not post a single summary comment.
11. Push fixes, update the PR description again to stay coherent across all commits.
12. Report what was changed in response to the review.

## CI Review Loop

The repo has three automated reviewers on every PR. After pushing, loop until all are green:

| Reviewer | What it checks | How to address |
|---|---|---|
| **Gemini Bot** | Code quality, style, logic issues | Read comments via `gh api`, fix code or reply explaining why it's fine, then comment `/gemini review` to trigger a fresh pass |
| **Codecov** | Test coverage thresholds | Add missing tests, ensure new code is covered. Check the Codecov report for uncovered lines |
| **SonarCloud** | Security hotspots, code smells, duplications, bugs | Fix flagged issues — especially security hotspots and duplications. Resolve quality gate failures before merging |

**Workflow:**
1. Push code → wait for all three reviewers to report
2. Pull down Gemini comments, Codecov report, and SonarCloud findings
3. Fix issues: code changes for Gemini/SonarCloud, add tests for Codecov
4. Re-run local quality gates (`npm run lint && npm test && npx tsc --noEmit`)
5. Push fixes, comment `/gemini review` on the PR to re-trigger Gemini
6. Repeat until all three reviewers pass with no blocking issues

## Every PR: small, Provit-proven, self-audited (MANDATORY)

This is the standing bar for **every** PR — no exceptions. A PR that is missing the Provit journey or the self-audit comment is not ready to merge.

1. **One concern, small diff.** Extends the small-meaningful-commits rule to the PR level: one subsystem/behaviour per PR, minimal surface. If a change spans two concerns, split it into two PRs.
2. **A Provit E2E journey.** Every PR ships (or updates) a [Provit](../ (its own repo)) journey that (a) exercises the exact user flow the change affects on a **real device** and (b) doubles as the **regression guard** — re-running it proves no regression. Reference the journey name + the run result (pass/fail + device) in the PR. If the change can't be proven on-device by a journey, say why in the self-audit.
3. **A fails-before / passes-after jest test.** At least one unit/integration test that **fails without the change and passes with it** — the exact regression case. Mocks only at genuine boundaries (native/network/clock); never mock the thing under assertion (a green suite must mean the real thing works — deleting the impl must fail the test).
4. **A self-audit comment on the PR** (template below), posted **as a comment alongside the Provit result**. It records the SOLID/abstraction verdict, the mock-honesty check, platform parity, and standards for that specific change — so the audit travels with the PR and the reviewer sees the reasoning, not just the diff.

### Self-audit comment template (paste and fill on every PR)

```markdown
## Self-audit

### SOLID / abstraction
- Enough to abstract? [is there a real owning seam, or is a caller branching on a concrete type / `Platform.OS` mechanism?]
- SRP / DIP: [one responsibility; callers depend on an interface, not a concrete — no `kind===` / `instanceof` / `Platform.OS`-mechanism branch in a View or store]
- Single source of truth: [the rule/map/capability is defined ONCE, not duplicated across layers]
- Verdict: [clean · justified exception (why) · follow-up filed]

### Tests — no false green
- Unit: [what it drives — the REAL class/store/reducer, not a mock of the thing asserted]
- Integration: [the cross-layer path exercised end to end]
- Mocks: [only boundaries (native module / network / clock). Deleting the implementation under test MUST fail these tests.]
- Fails-before / passes-after: [the exact case that fails on `main` and passes here]

### Provit (on-device E2E)
- Journey: `<name>` — proves `<flow>` works on device AND guards regression
- Run: [pass/fail · device] (or: why an on-device journey isn't applicable)

### Platform parity
- iOS + Android: [both covered — genuine gaps modelled as capability-as-data, NOT a leaked `if (ios)` branch. One contract test guards both.]

### Standards (only if UI / copy touched)
- Design tokens (no hardcoded colors/sizes, weights ≤400, no emoji — vector icons only); brand voice (no em dashes, no exclamation marks, no forbidden words, no curly quotes).
```
