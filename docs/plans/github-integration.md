# Plan: GitHub integration (read PRs, merge, fix review-bot comments, suggest improvements)

Goal: connect the user's GitHub repos so the app can read pull requests (diff, description, review comments), merge, fix comments left by review bots, and suggest improvements.

Feasibility verdict from the 2026-07-06 study: feasible with constraints. The controlling constraint is what each executor can actually do:

1. **Reading PRs and merging** are plain authenticated REST calls. Feasible now once a token flow and a confirmation gate exist.
2. **Fixing bot comments** splits three ways:
   - Applying GitHub suggestion blocks is 100% deterministic. No LLM involved; GitHub exposes no apply-suggestion API, so the app reimplements the splice (GH-6). This is the highest-frequency fix action and costs zero model tokens.
   - Small single-file rewrites: on-device with a 4-9B model, behind a mandatory human diff preview.
   - Free-form multi-file fixes: not realistic on-device (measured: reliable diff APPLICATION emerges around 7B; diff GENERATION is unreliable at any open-model size). Escalate to the user's remote server through the existing provider seam, or delegate to the bots themselves and monitor.
3. **Improvement suggestions**: per-file chunked review on-device; full-quality via a remote server model.

Status:
- [ ] GH-0 (pro/) GitHub MCP preset
- [ ] GH-1 (core, optional) Built-in dispatch cleanup
- [ ] GH-2 (core) Destructive-tool approval gate
- [ ] GH-3 (pro/) githubService + client + connect UX
- [ ] GH-4 (pro/) Read tools
- [ ] GH-5 (pro/) Write tools behind the gate
- [ ] GH-6 (pro/) Deterministic suggestion applier
- [ ] GH-7 (pro/) LLM fix flow + escalation

## Core precondition (build this first)

Registration for tool FAMILIES already exists, and this plan uses it: `registerToolExtension` (`src/services/tools/extensions.ts:19-21`) is checked first by `executeToolCallSafely` (`src/services/generationToolLoop.ts:243-254`; `exts.find(e => e.canHandle(tc.name))` at line 245) before the built-in switch, so the GitHub tools need zero dispatch changes in core. The one genuine precondition is the approval gate (GH-2).

### GH-1 (optional): Built-in dispatch cleanup (~1-2 days, core)
Not a precondition for any GitHub phase. Optional behavior-neutral cleanup of the six BUILT-IN tools' internal dispatch: the hardcoded switch spanning `src/services/tools/handlers.ts:26-51` (`dispatchTool`) becomes a `Map<name, handler>`. Do NOT build a second registration API parallel to `registerToolExtension`; extensions are the seam for new tool families (reuse-before-building). If done, pin the fails-before test to the layer where it genuinely fails today: calling the handlers.ts dispatch directly with a name outside the switch throws "Unknown tool" on main. All six existing tools keep byte-identical results.

### GH-2: Destructive-tool approval gate (~3-4 days, core)
Verified gap: `executeToolCalls` (`src/services/generationToolLoop.ts:256-279`) runs every model-issued call immediately; callbacks are notify-only. Shipping any merge/commit tool without a gate would let a hallucinated tool call merge a PR.

Design:
- `destructive?: boolean` as capability-as-data on `ToolDefinition` (`src/services/tools/types.ts`) and on extension schema metadata.
- A `toolApprovalService` singleton, log tag `[APPR-SM]`: `idle -> pending(call) -> approved | denied`, with timeout-deny as the default outcome.
- A read-only `approvalStore` projection rendering a confirm sheet.
- The gate hooks `executeToolCalls` before execution AND the LiteRT native `onToolCall` path (`generationToolLoop.ts:426-455`), so both engines are covered by one seam.
- Denial returns a typed ToolResult error so the model is told the action was refused.
- Integration test: a destructive call pauses the loop; deny produces an error result; approve executes; non-destructive tools are unaffected (backward-compatible migration per repo rules).

This gate is generic core infrastructure: MCP write tools and any future email/calendar write tools need it too. Open product question: whether the gate should apply retroactively to existing Pro MCP write tools (recommended yes, with a migration note).

## Architecture (pro/ except where noted)

### githubService (`pro/services/github/githubService.ts`)
Owning singleton, log tag `[GH-SM]`: `disconnected -> validating (token pasted; GET /user + GET /rate_limit) -> connected -> rate-limited (Retry-After honored) -> expired/error (401/403 detected, re-auth prompt)`. Owns the token lifecycle, the REST client, pagination, and the deterministic suggestion applier.

### Token storage (the one security-critical decision)
- Keychain only: `Keychain.setGenericPassword` under service `ai.offgridmobile.github`, `ACCESSIBLE.WHEN_UNLOCKED`, mirroring the best-in-repo pattern at `src/services/remoteServerManagerUtils.ts:18-54`.
- The token must NEVER appear in any zustand-persisted object. The remote-server add path is the live anti-pattern to avoid: `src/services/remoteServerManager.ts:45-47` -> `src/stores/remoteServerStore.ts:90-102` persists plaintext `apiKey` to AsyncStorage (servers array partialize-persisted at `:314-321`). Do not copy it. (Fixing that existing leak is worth its own PR, independent of this track.)
- Non-secret metadata (login, token permissions, verifiedAt, rate-limit snapshot) follows the proLicenseService blob pattern (`src/services/proLicenseService.ts:61-88`).
- Logging: `[GH-SM]` traces log `hasToken` booleans only. The dev debug file sink (`src/utils/debugLogFile.ts`) mirrors every logger line with no redaction layer.

### githubClient (`pro/services/github/githubClient.ts`)
Modeled on `src/services/keygenClient.ts:52-60` (one `request()` wrapper, typed NetworkError so offline is distinguishable from 4xx), built on `fetchWithTimeout` (`src/services/httpClient.ts:110-164`). Adds what the codebase lacks:
- `X-RateLimit-*` / `Retry-After`-aware backoff. GitHub secondary limits: 80 content-generating requests/min, 500/hr, shared with the web UI.
- RFC 5988 Link-header pagination.
- `Accept: application/vnd.github+json`, plus `application/vnd.github.diff` for diff fetches.
- A small GraphQL POST for `resolveReviewThread` (REST cannot resolve review threads).

### Auth UX
Fine-grained PAT pasted once (permissions: Pull requests R/W + Contents R/W on selected repos), matching the paste-a-key precedent of `activateProByKey` (`src/services/proLicenseService.ts:184-225`). Settings section registered through `src/components/settings/sectionRegistry.ts` with a deep link to the pre-filled fine-grained-token creation URL and a validity check via GET /user and /rate_limit. OAuth device flow is deferred: it requires registering a GitHub App identity, and GitHub's MCP endpoint has no dynamic client registration (github/github-mcp-server issue #1404). PAT sharp edges the `[GH-SM]` expired state must handle: no refresh mechanism; org-owned resources default to a 366-day max lifetime; one PAT per resource owner.

### githubStore
Read-only projection: login, connection status, token permissions, rate-limit remaining. Never the token.

### Tools (GitHubToolExtension via `src/services/tools/extensions.ts`)
Tight flat schemas, one concern per tool, never-throw ToolResult contract per `src/services/tools/toolResult.ts`:

| Tool | Notes |
|---|---|
| `github_list_prs` | repo + state filters, paginated |
| `github_read_pr` | `mode` enum: `description \| files \| diff \| comments \| reviews`; per-file and page params so a single call never exceeds a ~4,000-char result budget (consistent with `read_url`'s cap at `handlers.ts:346`); truncation text says "truncated, request next page" so the model paginates |
| `github_reply_review_comment` | POST /pulls/comments/{id}/replies |
| `github_update_branch` | update PR branch |
| `github_merge_pr` | PUT /pulls/{n}/merge; `destructive: true` |
| `github_commit_file` | PUT /repos/{o}/{r}/contents/{path}, one file per call; `destructive: true` |
| `github_apply_suggestions` | deterministic applier, see GH-6; `destructive: true` |

Bot comments surface with `user.type == 'Bot'` and thread-resolution state so the model (and the UI) can enumerate what needs fixing.

### Model routing
- On-device runs use llama.rn, not LiteRT: the LiteRT tool loop is capped near 4096 context (`generationToolLoop.ts:28-31`), which cannot hold PR content.
- One-tool-call-per-turn workflow design. Measured: single-call dispatch is 95%+ for good 4B models, but multi-turn agent benchmarks collapse to ~35-50% at 4B (BFCL v3/v4), and 95% per call compounds to ~66% over eight steps. The 3-iteration/5-call loop budget (`generationToolLoop.ts:19-20`) is then never the limiting factor.
- Defaults: a Qwen3-4B-class instruct model for tool dispatch; a 9B on the 16 GB device for summarize/review. Q4_K_M is the quantization floor for tool-call reliability; Q4_0 additionally gains Armv9 prefill for long diff chunks.
- Multi-step fix flows escalate through the existing `remoteServerManager` / provider registry seam, or delegate: comment `/gemini review`, reply invoking CodeRabbit, or assign the Copilot coding agent, then monitor resulting commits from the phone.
- llama.cpp/llama.rn tool-parsing churn is real (parser regressions as recent as 2026-03, llama.cpp issues #20198/#20260); the repo already carries salvage parsers and a Gemma token filter (`src/services/llmToolGeneration.ts:16-24`, `generationToolLoop.ts:33-121`). Pin llama.rn versions deliberately and keep the salvage path.

## Core vs Pro

Core gets only the generic seams (GH-1 dispatch refactor, GH-2 approval gate, optionally a shared keychain-secret helper generalizing the remoteServerManagerUtils trio). The GitHub feature itself is a Pro surface, matching the connected-external-account precedent (email/calendar tools, MCP). Product decision recorded honestly: under this split, free users get no GitHub tools; if the maintainer wants GitHub-read in free core (like `web_search`), the read tools + service could land in core's registry with zero architectural change.

## Phases

| Phase | Scope | Effort (days) |
|---|---|---|
| GH-0 | (pro/) Add a `github` preset to the Pro MCP presets: URL `https://api.githubcopilot.com/mcp/x/pulls` (toolset-scoped to keep schema count sane), authMode `header` (PAT as Bearer). Zero new client code; immediate PR read/merge for users driving a remote server model. Tests split: the preset data lands in `pro/ui/mcpPresets` (pro-repo PR); the shape test extends core `__tests__/unit/tools/mcpPresets.test.ts` in a small companion core PR (its generic data invariants cover any new preset automatically; add github-specific assertions mirroring the notion ones). Provit: connect and list PRs on a real repo | 1 |
| GH-1 | (core, optional) Built-in dispatch cleanup: handlers.ts:26-51 switch -> Map, behavior-neutral; not a precondition for any GitHub phase (see above) | 1-2 |
| GH-2 | (core) destructive flag + toolApprovalService `[APPR-SM]` + approvalStore + confirm sheet + gates in both engine paths | 3-4 |
| GH-3 | (pro/) githubService `[GH-SM]` + Keychain token storage + githubClient (backoff, pagination, accepts, GraphQL) + githubStore + connect settings section. Unit tests drive the real service against recorded REST fixtures; contract test for the 401 -> expired transition | 4-5 |
| GH-4 | (pro/) `github_list_prs` + `github_read_pr` with the pagination/truncation contract; bot-comment surfacing. Integration test through the real tool loop against fixtures. Provit: read a live PR's diff and bot comments on the S23 Ultra | 3-4 |
| GH-5 | (pro/) merge / update-branch / reply / GraphQL resolve, all `destructive: true` behind GH-2's gate; secondary-rate-limit backoff exercised in tests (403/429 with retry-after). Provit: merge a test-repo PR from the phone with the confirm sheet | 2-3 |
| GH-6 | (pro/) `github_apply_suggestions`: enumerate unresolved bot threads; parse suggestion fenced blocks with path/start_line/line/side/original_line/diff_hunk from the review-comment payload; staleness-check against the current head sha (refuse or re-fetch on drift); splice lines; commit via PUT contents (single-file) or the Git Data API (blobs/trees/commits/refs) for multi-file batches; reply per comment; GraphQL-resolve the thread. Heavy unit coverage on the splice/anchor math: it is the one place silent corruption is possible | 4-5 |
| GH-7 | (pro/) LLM fix flow: single-file whole-file rewrite with a mandatory diff-preview sheet before `github_commit_file`; explicit escalation paths (remote server via provider registry; delegate to bots and monitor); improvement-suggestions prompt preset (per-file chunked review). Provit: fix one real bot comment end to end on the Nothing Phone 3 | 4-6 |

## Risks

- Small-model multi-step reliability is the hard ceiling; the one-call-per-turn design is a requirement, not a style choice.
- GH-2 is a hard precondition for GH-5 to GH-7. Do not ship write tools without the gate.
- PR diffs are tens-to-hundreds of KB; without the per-file/pagination contract, one `github_read_pr` call destroys the turn.
- Token hygiene: Keychain-only, `hasToken` booleans in logs, never in persisted zustand. The existing remote-server AsyncStorage plaintext `apiKey` is the anti-pattern (and worth fixing separately).
- Suggestion anchors go stale when the branch moves; the applier must compare `original_line`/`position` against the current head commit and refuse on drift.
- No apply-suggestion or resolve-thread REST API exists; application is reimplemented (GH-6) and resolution is GraphQL-only, adding a second protocol surface.
- GitHub secondary rate limits can throttle a commit+reply+resolve loop; the client needs write batching and Retry-After honoring.
- MCP-preset caveats (GH-0): the hosted endpoint routes through Copilot infrastructure (brand tension with "nothing is sent anywhere" beyond github.com); org "MCP servers in Copilot" policies can block it; OAuth to it needs a registered GitHub App. PAT header auth avoids all of this.
- The pro/ submodule is not checked out in this clone: GH-0 (preset data) and GH-3 to GH-7 are separate branch+PRs in the `@offgrid/pro` repo; only GH-1, GH-2, and GH-0's companion core test can be developed and tested here.

## Device notes

Nothing Phone 3 (SM8735, 16 GB): the better on-device tier; a 9B Q4 (~5-6 GB weights) with an 8-16K context fits, making it the preferred device for PR summarization and per-file review. LiteRT NPU availability on the 8s-series SM8735 is unverified, and irrelevant here since LiteRT's context cap disqualifies it for PR content; use llama.rn (CPU decode, Armv9). Galaxy S23 Ultra (SM8550, 12 GB): 7-8B Q4 is the practical ceiling with a useful context; expect roughly 10 tok/s decode for 7B Q4, so a 500-token review answer takes about a minute; the 4B tier is the better default for tool dispatch there. Sustained-decode thermal derating of 15-41% from burst applies to both phones; the deterministic applier and all REST reads are unaffected (network-bound, zero model tokens). For multi-step fix flows, both devices should escalate to the user's LAN remote server; the phone then acts as orchestrator and approval surface.
