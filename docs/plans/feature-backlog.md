# Plan: Feature backlog (search, export, project pinning, hybrid retrieval, memory, backup)

Six independent features from the 2026-07-06 improvement scan, each verified against the codebase by an adversarial critic pass. Ranked by impact-per-effort for a daily power user on 12-16 GB Android flagships. Each is one PR (FB-6 may need two).

Status:
- [ ] FB-1 Full-text search across conversations
- [ ] FB-2 Export conversation as Markdown via the system share sheet
- [ ] FB-3 Per-project model pinning and settings overrides
- [ ] FB-4 Hybrid RAG retrieval: FTS5 keyword search fused with vector cosine
- [ ] FB-5 Persistent cross-chat memory via remember/recall tools
- [ ] FB-6 Encrypted backup / restore between devices

---

## FB-1: Full-text search across conversations (~1-2 days)

ChatsListScreen has no search at all (verified by grep). A daily user accumulates hundreds of persisted conversations with no way to find "that chat where the model explained X". All conversations already live in memory via zustand persist (`src/stores/chatStore.ts:384-395`), so v1 is a pure filter over `conversations[].messages[].content` with match-snippet highlighting. Reuse the shared search styles the Models screen already has (reuse-before-building rule). No schema, no native code, no new state owner.

Tests: unit on the filter/snippet logic; rntl render test for the search states (empty query, hits, no hits).

## FB-2: Export conversation as Markdown via the share sheet (~1-2 days)

The only way to get a conversation out today is copying one message to the clipboard; no export path exists anywhere (verified: no exportConversation/toMarkdown in `src/`). Build a Markdown export (title, date, model name, role headers, fenced code blocks preserved, optional tok/s footer from GenerationMeta at `src/types/index.ts:185-187`) through the native share sheet.

Seams to reuse: `chatStore.getConversationMessages` (`src/stores/chatStore.ts:145`) for data; the existing `Share.share` pattern at `src/screens/GalleryScreen/useGalleryActions.ts:159` and `src/components/DebugLogsScreen/index.tsx:78`. Ship as a ChatScreen header action plus a long-press action in ChatsListScreen.

This is the best-fit stopgap for the Obsidian user before obsidian-vault-integration.md lands: transcripts can be filed straight into a vault.

Tests: unit on the Markdown serializer (code fences, multi-turn, metadata footer); the serializer is pure, so drive the real chatStore data shape.

## FB-3: Per-project model pinning and settings overrides (~2-3 days pinning; +2 for overrides)

`Project` carries only a systemPrompt as behavior config (verified: id/name/description/systemPrompt/icon/dates at `src/types/index.ts:365-373`). A power user runs different models for different jobs and re-selects manually every time. Add optional `preferredModelId` plus settings overrides (temperature, maxTokens, thinking) to `Project`, resolved once in the generation path so opening a project chat auto-loads its model through `activeModelService` - the ONLY load path, so residency/eviction just works.

Ship pinning first; overrides are an increment. Tests: integration test that opening a pinned project triggers the load intent through activeModelService and that an unpinned project keeps the current model.

## FB-4: Hybrid RAG retrieval: SQLite FTS5 + vector cosine (~2-3 days)

Retrieval is pure MiniLM cosine over every chunk (verified: `src/services/rag/retrieval.ts:24-66`), exactly the mode that misses exact-term queries: function names, error codes, config keys - the content a developer's knowledge base holds. Worse, when embeddings are missing or fail to load, the fallback is literally "return the first K chunks" (three separate fallback paths in retrieval.ts).

Change: op-sqlite supports FTS5 virtual tables. Index `rag_chunks.content` (schema at `src/services/rag/database.ts:52-60`) at ingestion; at query time run both searches and merge with reciprocal rank fusion inside `retrievalService.search()`, so `search_knowledge_base` and every other caller improve with zero interface change - and the embedding-failure cliff becomes a real keyword search instead of arbitrary chunks.

Coordination: OBS-8 in obsidian-vault-integration.md rewrites the same scoring path (`retrieval.ts`); whichever of FB-4/OBS-8 lands second rebases onto the merged retrieval code.

Tests: fails-before/passes-after on an exact-identifier query that cosine ranks poorly; the embedding-unavailable path returns keyword hits, not first-K.

## FB-5: Persistent cross-chat memory via remember/recall tools (~4-6 days)

Every conversation starts cold; a power user re-states preferences, stack details, and ongoing context in every chat. Add a memory scope to rag.db (a `memories` table beside rag_documents/rag_chunks/rag_embeddings), expose `remember` and `recall_memory` as built-in tools in AVAILABLE_TOOLS (verified: currently six tools, none memory-related, `src/services/tools/registry.ts:1-89`), and inject a compact memory digest the way the knowledge-base hint is already injected (`src/screens/ChatScreen/useChatGenerationActions.ts:228-250`).

Recall reuses the bundled MiniLM embedder (`src/services/rag/embedding.ts`) and cosine ranking: zero new models, zero new dependencies.

Ordering note: land QW-2 (performance-quick-wins.md) first, then inject the digest via the user message / stable-prefix rules established there, so memory does not reintroduce the KV-prefix-busting problem.

Tests: unit on the tools (drive the real rag database with an in-memory op-sqlite); integration on remember-in-chat-A, recall-in-chat-B.

## FB-6: Encrypted backup / restore between devices (~1-1.5 weeks)

The user owns two flagships and nothing moves between them: conversations, projects, and RAG documents are trapped per device. `docs/ARCHITECTURE.md:1136` already lists "Conversation export/import with encryption (planned)". Most pieces exist: `react-native-zip-archive` is a dependency (`package.json:76`), all state is AsyncStorage-persisted zustand with clean partialize boundaries (`src/stores/chatStore.ts:384-395`), rag.db is one file, attachments live in the documents dir.

One verified correction to scope honestly: `authService` has NO encryption machinery - it is a passphrase-hash lock only (`src/services/authService.ts:5-46`) and no crypto dependency exists in package.json. A real AES dependency (for example react-native-quick-crypto or a dedicated AES module) is part of the work. Import must rehydrate stores and re-base file paths. Models are excluded; they re-download through the existing download service.

Tests: round-trip integration (backup on a simulated device A layout, restore into a clean container, assert stores hydrate and paths re-base); wrong-passphrase and truncated-archive failure cases.
