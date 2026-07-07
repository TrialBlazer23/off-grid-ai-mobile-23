# Implementation Plans

Work orders for coding agents. The four tracked plans below come from a feasibility study run on 2026-07-06: twelve agents mapped this codebase and researched integration options, and every load-bearing claim was verified against the actual files. Each plan is written so an agent can pick up one phase, read the referenced files, and ship it as one PR without needing the study.

Line references were verified on 2026-07-06. Line numbers drift; if a reference does not land, re-locate by the named symbol before assuming the claim is stale.

## Tracks

| Track | Plan | Phases | Effort | Depends on |
|---|---|---|---|---|
| Performance quick wins | [performance-quick-wins.md](performance-quick-wins.md) | QW-1 to QW-5 | ~7-11 days | nothing; each phase independent |
| Obsidian vault integration | [obsidian-vault-integration.md](obsidian-vault-integration.md) | OBS-1 to OBS-10 | ~28 days | OBS-1 blocks OBS-2/5/6/7/9/10; OBS-3/4/8 are RAG-side and independent |
| GitHub integration | [github-integration.md](github-integration.md) | GH-0 to GH-7 | ~22-30 days | GH-2 (core approval gate) blocks GH-5 to GH-7 |
| Feature backlog | [feature-backlog.md](feature-backlog.md) | FB-1 to FB-6 | varies | FB-5 should land after QW-2 |

## Suggested order

1. QW-1, QW-2, QW-3 and FB-2 (markdown export): small PRs, immediate daily-use payoff, and QW-1/QW-2 make every long chat and RAG turn faster before the bigger tracks add load.
2. GH-0 (MCP preset, one day of work in `pro/`) for immediate PR read/merge capability via remote-server sessions.
3. GH-2: the core approval gate that the GitHub write tools and any future MCP/write tools need.
4. OBS-1 to OBS-3 in parallel with GH-3 and GH-4.
5. Everything else by the dependency columns above.

## Rules every agent must follow

These restate the non-negotiables from the repo root `CLAUDE.md`; read that file first. The short version:

- Never push to `main`. Branch (`feat/`, `fix/`, `docs/`, `chore/`) and open a PR.
- One phase = one PR. Do not collapse phases to save round trips; the plans are sized to the repo's one-concern-per-PR policy.
- Every behavior change ships with a fails-before/passes-after jest test plus an integration test. Mocks only at genuine boundaries (native modules, network, clock); never mock the thing under assertion.
- Every PR carries a Provit on-device journey (or a self-audit note on why one is not applicable) and the self-audit comment from the CLAUDE.md template.
- Work under `pro/` is a separate branch and PR in the `@offgrid/pro` repo. The submodule is not always checked out; check for `pro/package.json` before assuming it is present.
- Service -> store -> view: a singleton service owns state machines, resources, and side effects; zustand stores are read-only projections; views dispatch intents. New subsystems get a `[X-SM]` logger tag.
- Platform gaps are capability flags on data (see `DownloadCapabilities` in `src/services/modelDownloadService/types.ts`), never `Platform.OS` mechanism branches in callers.
- UI work uses design tokens only (`TYPOGRAPHY`, `COLORS`, `SPACING`, weights <= 400, vector icons, no emoji). Copy follows `docs/brand_tone_voice.md`.
- Reuse before building: grep for an existing component, hook, or service before writing a new one.

## Other plans in this directory

[audio-mode-progress-captions.md](audio-mode-progress-captions.md) and [whisper-download-sync.md](whisper-download-sync.md) predate the 2026-07-06 study and are not phase-tracked by this index.

## Status

Mark a phase done in the checklist at the top of its plan when the PR merges.
