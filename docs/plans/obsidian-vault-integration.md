# Plan: Obsidian vault integration (read/write vault access, vault RAG, instruction notes)

Goal: the user's Obsidian vault becomes a mutual read/write space between the user and the on-device model. The model reads the vault as knowledge (RAG over notes) and as standing instructions for projects (designated instruction notes). The model writes back clean markdown (YAML frontmatter, wikilinks preserved) that renders correctly with the user's community plugins and themes.

Feasibility verdict from the 2026-07-06 study: feasible with constraints. The constraints shape this plan; read them before starting any phase.

Status:
- [ ] OBS-1 VaultAccessModule native contract (both platforms)
- [ ] OBS-2 VaultService + [VAULT-SM] + vaultStore + linking UI
- [ ] OBS-3 RAG schema migration + incremental lifecycle
- [ ] OBS-4 Markdown-aware ingestion
- [ ] OBS-5 Scan-and-diff vault sync + resumable indexing queue
- [ ] OBS-6 Instruction notes (standing project instructions from the vault)
- [ ] OBS-7 Core write path: "Save to Obsidian" + markdownWriter
- [ ] OBS-8 Retrieval scale hardening
- [ ] OBS-9 (pro/) Agentic vault tool extension
- [ ] OBS-10 (pro/, optional) Desktop MCP bridge recipe

## Hard constraints (verified 2026-07-06)

1. **Android "app storage" vaults are unreachable.** Obsidian 1.8.10+ offers an "app storage" vault location that no third-party app can access. The linking flow (OBS-2) must detect this failure and walk the user through Obsidian's built-in migration to "device storage" (see obsidian.md/help on Android).
2. **The Local REST API plugin is desktop-only.** Its manifest declares `isDesktopOnly: true`, so a localhost API on the phone is not an option. Direct file access is the path. The plugin's desktop MCP endpoint remains useful as a complement (OBS-10).
3. **No file watching.** Neither SAF trees (Android) nor security-scoped bookmarks (iOS) deliver reliable change notifications. Sync is scan-and-diff: rescan on app foreground plus a manual refresh intent. The model can act on a note that is seconds stale.
4. **Nothing in the repo does this today.** All four document-picker call sites are single-file copy-on-import (`src/utils/resolvePickedFileUri.ts:21-33`, `src/screens/KnowledgeBaseScreen.tsx:78-79`); there is zero SAF or bookmark code in `src/`, `android/`, or `ios/`.

## Architecture

Follows the repo's service -> store -> view pattern, mirroring the ModelDownloadService seam.

### VaultAccessModule (new native module, one TS contract, both platforms)

TS contract in `src/services/vaultAccess/types.ts` + `src/services/vaultAccess/index.ts` (classic bridge, `NativeModules.VaultAccessModule`, payload types like `src/services/backgroundDownloadTypes.ts`). Identical method names and semantics on both sides:

| Method | Purpose |
|---|---|
| `linkFolder()` | folder picker + persistable grant; returns a `linkId` |
| `verifyLink(linkId)` | re-validate the grant/bookmark at launch and on foreground |
| `listTree(linkId)` | full recursive listing: `{relativePath, name, mtimeMs, sizeBytes}[]` |
| `readFile(linkId, relPath)` | UTF-8 read |
| `writeFileAtomic(linkId, relPath, content)` | temp-then-rename write |
| `createFile(linkId, relPath, content)` | create without overwrite |
| `deleteFile(linkId, relPath)` | delete |
| `unlink(linkId)` | drop the grant and stored record |

Implementation notes:
- First try `@react-native-documents/picker` v12 (`pickDirectory` + `requestLongTermAccess`); it is already a dependency (`package.json:32`). Its long-term-access API is documented behind sponsor-only docs and node_modules is not installed in this clone, so verify the API surface immediately after `npm install`. Fallback: implement the grant/bookmark natively inside VaultAccessModule (adds ~1-2 days to OBS-1).
- Android (`android/app/src/main/java/ai/offgridmobile/vault/VaultAccessModule.kt`): `takePersistableUriPermission`; `listTree` MUST use bulk `DocumentsContract.buildChildDocumentsUriUsingTree` cursor enumeration, never per-file `DocumentFile` calls (measured ~14x slower on large trees, 48s vs 3.5s); reads/writes via `ContentResolver` streams. `DocumentsContract.renameDocument` cannot replace an existing target, so atomic write degrades to write-temp / delete-original / rename with a small non-atomic window; keep that window minimal and documented. Register in `MainApplication.kt`.
- iOS (`ios/VaultAccessModule.swift`): persist and resolve security-scoped bookmarks (recreate on staleness); bracket every access with `startAccessingSecurityScopedResource` / stop (a transient version of this pattern exists at `ios/PDFExtractorModule.swift:35`); reads through `NSFileCoordinator`; atomic writes via `FileManager.replaceItemAt`; hydrate iCloud-evicted `.icloud` placeholders with `startDownloadingUbiquitousItem`.
- Genuine OS gaps are capability-as-data, exactly like `DownloadCapabilities` (`src/services/modelDownloadService/types.ts:43-49`): `VaultCapabilities { persistentAccess, watch: false (both, today), atomicReplace, remoteFileHydration (true on iOS only) }`. UI renders from flags; no `Platform.OS` mechanism branches.
- Contract tests in `__tests__/contracts/` guard both platforms against the shared TS contract.

### VaultService (owning singleton, `src/services/vaultService/`)

Files: `index.ts`, `stateMachine.ts`, `scanner.ts`, `markdownWriter.ts`, `instructionNotes.ts`. Log tag `[VAULT-SM]`.

State machine: `unlinked -> linking -> validating -> idle -> scanning -> indexing(i/N) -> idle`, plus `revoked` and `error` (surface errors through the existing modelFailureHandler card pattern where appropriate).

Owns:
- Link records (persisted vault URI/bookmark plus project bindings) and launch/foreground revalidation.
- Scan-and-diff: compare `listTree` mtime/size against `rag_documents` rows; changed -> reindex, missing -> orphan-sweep delete, new -> index.
- The indexing queue: serial, resumable, progress events. This replaces nothing in the attachment flow; it exists because a 1,000+ note vault cannot be indexed in a foreground UI loop. The queue yields while `[GEN-SM]` is generating so indexing never competes with inference for thermals and bandwidth.
- ALL writes to the vault: atomic temp+rename, never touches `.obsidian/`, always through `markdownWriter.ts` (YAML frontmatter per the Obsidian Properties spec: quoted internal links, dates as YYYY-MM-DD; `[[wikilinks]]` preserved verbatim; no curly quotes).
- Rescan trigger on AppState `active` (mirrors Obsidian mobile's own rescan-on-resume).

### vaultStore (`src/stores/vaultStore.ts`)

Thin read-only zustand projection: `{linkStatus, vaultName, fileCount, phase, indexProgress, lastScanAt, error}`. Views (a Settings section, KnowledgeBase surfaces) observe it and dispatch intents (`linkVault()`, `rescan()`, `unlink()`) to VaultService. No logic in hooks.

### RAG subsystem extensions (all in `src/services/rag/`)

a. Schema migration in `database.ts`: add to `rag_documents`: `source ('attachment'|'vault')`, `source_uri`, `relative_path`, `mtime_ms`, `content_hash`; add to `rag_chunks`: `heading TEXT` (anchor path) so citations can render `[[Note#Heading]]`. Note: `database.ts` has NO migration mechanism today; tables are created only via `CREATE TABLE IF NOT EXISTS` (`database.ts:41-70`), so editing the CREATE statements alone is a silent no-op on every existing install. Add a `PRAGMA user_version` gate in `ensureReady()`: on version < N run `ALTER TABLE ... ADD COLUMN` for each new column (with defaults, e.g. `source TEXT NOT NULL DEFAULT 'attachment'`), then bump `user_version`. New installs get the columns in the CREATE statements; existing installs get them via the ALTERs. Test: open a legacy-schema db and assert the columns exist after `ensureReady()`.
b. Identity: vault docs dedupe on `(project_id, relative_path)`. Fixes the hard throw on duplicate document name at `rag/index.ts:39`, which breaks on any real vault folder.
c. `reindexDocument(docId, text)`: delete chunks+embeddings for the doc, reinsert (`insertEmbeddingsBatch` is insert-only today, so delete-first).
d. New `src/services/rag/markdown.ts`: YAML frontmatter parse (tags/aliases to doc metadata, YAML excluded from embedded text); heading-aware section splitting layered over `chunkDocument` (`chunking.ts:44-78` stays the base splitter within sections).
e. Retrieval scale: keep vectors as `Float32Array` end-to-end (drop the `Array.from` copy at `database.ts:119`), score in chunks, add a score threshold, and route vault-scale prompt injection through the already-written-but-unused `searchWithBudget` (`retrieval.ts`) instead of the bare top-5 call at `useChatGenerationActions.ts:240`. Coordination: FB-4 in feature-backlog.md rewrites the same scoring path (`retrieval.ts`); whichever of OBS-8/FB-4 lands second rebases onto the merged retrieval code.

### Instruction notes (the standing-instructions ask)

Extend `Project` (`src/types/index.ts`) with `instructionNotePaths?: string[]` plus a picker UI in ProjectEdit/ProjectDetail listing vault notes. `VaultService.instructionNotes.ts` resolves fresh note content (mtime-checked cache) and a service-level `buildProjectPrompt()` appends it after `project.systemPrompt` inside `resolveToolsAndPrompt` (`useChatGenerationActions.ts:257-273`). Note: `resolveToolsAndPrompt` is synchronous today; make it async and await `buildProjectPrompt()` there (its caller `startGenerationFn` at `useChatGenerationActions.ts:286` is already async and awaits `injectRagContext` on the next line), keeping the mtime-checked cache so unchanged notes cost a stat call, not a file read. Editing the note in Obsidian updates the model's standing instructions on the next turn.

Note the interaction with QW-2 (performance-quick-wins.md): instruction notes ARE part of the system prompt by design, and they change rarely (mtime-gated), so they do not defeat KV prefix reuse the way per-query RAG chunks do. Retrieved chunks go in the user message; instruction notes stay in the system prompt.

### Write path split

- Core, user-initiated: a "Save to Obsidian" message action calling `VaultService.writeNote()` with frontmatter (created, source, model, tags). Not a tool surface, so it is core (OBS-7).
- Pro, model-initiated: the agentic tool suite (OBS-9), gated by a user-confirmation setting.

## Core vs Pro

CORE: VaultAccessModule, VaultService + vaultStore, linking UI, RAG migration + markdown ingestion + sync + retrieval scaling, instruction notes, "Save to Obsidian". Rationale: the knowledge base, documentService, and the `search_knowledge_base` built-in (`src/services/tools/registry.ts:60-73`) are core today; vault ingestion is the same subsystem grown up, and `search_knowledge_base` covers vault docs with zero registry change.

PRO (separate branch + PR in the `@offgrid/pro` repo): `obsidian_list_notes` / `obsidian_read_note` / `obsidian_write_note` / `obsidian_append_note` as a ToolExtension via `src/services/tools/extensions.ts`, executing through core's VaultService (the `@offgrid/core` alias is the sanctioned import direction); model-initiated writes behind a confirm setting. Also PRO: the desktop MCP bridge recipe (OBS-10).

## Phases

| Phase | Scope | Effort (days) |
|---|---|---|
| OBS-1 | VaultAccessModule: TS contract + Kotlin (SAF grant, bulk cursor listTree, ContentResolver I/O, temp+rename) + Swift (bookmarks, scoped-access bracketing, NSFileCoordinator, replaceItemAt, iCloud hydration) + VaultCapabilities + contract tests + a manual Provit journey picking a real vault | 5 |
| OBS-2 | VaultService state machine + persisted link records + revalidation + vaultStore + Settings/KnowledgeBase "Link Obsidian vault" UI (design tokens, Feather icons) + Android app-storage detection with migration guidance | 3 |
| OBS-3 | RAG schema migration (source, source_uri, relative_path, mtime_ms, content_hash, chunk heading) + relative-path identity + reindexDocument + orphan sweep; fails-before/passes-after tests on the existing `__tests__/unit/services/rag/` anchors | 3 |
| OBS-4 | markdown.ts: frontmatter parse + heading-aware splitting + chunk heading anchors; vault docs only, attachment path byte-identical (behavior-neutral migration) | 2.5 |
| OBS-5 | Scanner: mtime/size diff, serial resumable indexing queue with progress, rescan on foreground + manual refresh, yields while [GEN-SM] active; vault docs appear in KnowledgeBase surfaces with a "vault" badge. Integration test: change/delete/add files under a fake tree, assert the index converges | 3.5 |
| OBS-6 | Project.instructionNotePaths + picker UI + mtime-cached resolution into buildProjectPrompt; regression test on prompt assembly order | 2 |
| OBS-7 | markdownWriter + atomic write + "Save to Obsidian" message action (append to chosen note / create note / daily-note path template). Provit: save a chat answer, open in Obsidian, verify rendering | 2.5 |
| OBS-8 | Float32Array end-to-end, chunked scoring, score threshold, searchWithBudget routing; benchmark against a synthetic 30k-chunk corpus; defer sqlite-vec until measurements demand it | 2.5 |
| OBS-9 | (pro/) ToolExtension with the four vault tools, confirm setting, surfaced through getToolDefinitions() | 3 |
| OBS-10 | (pro/, optional) Local REST API plugin /mcp/ endpoint over Tailscale wired into the existing Pro MCP servers UI; mostly config + docs + one Provit journey | 1 |

## Risks

- Android "app storage" vault location is a hard blocker; detection + migration guidance is part of OBS-2, not optional.
- No change notifications: the model can act on a stale note. Keep the staleness window visible (lastScanAt in the UI).
- SAF enumeration performance: per-file DocumentFile calls make multi-thousand-note vault scans take tens of seconds; bulk cursor enumeration in OBS-1 is load-bearing.
- Atomic-write semantics under SAF have a small non-atomic window; test against Obsidian Sync and Syncthing (both produce conflict artifacts on races; Obsidian Sync has a documented ~2-minute remote-wins race on newly created notes).
- Corrupting user notes is the reputational risk. Mitigations: the single markdownWriter path, never touching `.obsidian/`, defaulting model writes to new-note or append-only.
- Retrieval degrades on big vaults before OBS-8: today every embedding row is deserialized into a JS number[] and cosine-scored per query (`database.ts:141-155`); a 30k-chunk vault is ~12M floats per search. Sequential one-at-a-time embedding (`embedding.ts` embedBatch) makes first-index of a large vault a long job; the resumable queue is load-bearing, not cosmetic.
- The picker v12 long-term-access API needs verification after `npm install`; budget the native fallback.
- iOS specifics (bookmark staleness, Obsidian's "Require Face ID" lock blocking folder access, iCloud-evicted placeholders with no completion callback) do not block the primary user, but contract parity requires implementing and testing them.
- Prompt injection via vault content: instruction notes are injected verbatim by design (user-designated, acceptable, but document the trust decision); retrieved chunks already get bracket-stripping in `formatForPrompt`.
- Vault-scale prompt bloat: route injection through `searchWithBudget` (~25% context cap) and show token cost in the project UI.

## Device notes

Both reference devices are Android, so the SAF path serves the primary user from OBS-1; iOS ships in the same PRs for contract parity. Nothing Phone 3 (SM8735, 16 GB): the ~90 MB MiniLM embedder (registered with modelResidency by `src/services/rag/embedding.ts`) coexists with a 7-9B Q4 chat model. Galaxy S23 Ultra (SM8550, 12 GB): same, with less headroom; `[MEM-SM]` eviction already handles the embedder being dropped during large-model chat. Vault RAG makes prompts long and prefill is the mobile bottleneck, so time-to-first-token on big knowledge injections is the metric to watch; `searchWithBudget` is the cheap lever, and QW-1/QW-2 in performance-quick-wins.md are prerequisites for this feature feeling fast.
