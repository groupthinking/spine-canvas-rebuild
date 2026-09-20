## 11. 4-Phase Build Roadmap

**TABLE 16 — FOUR-PHASE BUILD ROADMAP, 15 WEEKS TOTAL**

| Phase | Weeks | Scope | Exit Criteria |
| :--- | :--- | :--- | :--- |
| 1 — Foundation & Graph | 1–3 | Next.js 14 scaffold; Supabase 12 tables with RLS/Realtime; React Flow canvas with drag/connect and position persistence; text-block, web-block, file-block, prompt-block, table-block; verify inferred schema against live API payloads | A block graph can be created, persisted, reloaded, and re-rendered pixel-identically; one AI block generates end-to-end; the two truncated API block types are identified |
| 2 — Generative Library | 4–7 | Model registry with all four providers and fallback chains; artifact pipeline; memo-block, document-block, excel-block, presentation-block, image-block, list-block, app-block, yt-block; run lifecycle with estimated_duration_ms and credits_consumed | Every documented artifact-producing block type emits a real, openable file with a working download button; runs reconcile token spend into credits_consumed |
| 3 — Swarm & Research | 8–11 | Swarm planner writing the L1/L2/L3 tasks tree; topological dispatcher with bounded concurrency; cascading re-run via edges topological sort + Realtime; deep-research-block; web-research-block; Folder and Folder Synthesis map-reduce | A single prompt produces a multi-block canvas; editing an upstream block reliably invalidates and re-runs all descendants; a depth-2 research report renders with citations and a source list |
| 4 — Integrations & Hardening | 12–15 | MCP server with StreamableHTTPServerTransport; GitHub via github-mcp-server; Google OAuth2Client connectors with encrypted token storage; api_keys issuance; webhook_endpoints + signed webhook_deliveries with retry; observability, backup/restore | External MCP clients can drive the canvas; connectors survive token refresh; every run emits signed webhook deliveries with a logged response_status; restore-from-backup rehearsed |

### 11.1 Sequencing Rationale
1. Schema before surface. All schema rows are Inferred; validating in Phase 1 costs days, discovering a mismatch in Phase 3 costs weeks.
2. Artifacts before agents. Phase 2 proves the docx/xlsx/pptx pipeline before orchestration complexity is layered on.
3. Cascade with Swarm. Cascading re-run and the Swarm dispatcher share the same topological sort over edges, so they are built together.
4. Integrations last. Connectors are additive and independently testable; deferring them keeps Phases 1–3 free of OAuth and quota variability.

---

## 12. Known Gaps and Confidence Ledger

**TABLE 17 — KNOWN GAPS, IMPACT ASSESSMENT, AND MITIGATIONS**

| # | Gap | Impact | Mitigation |
| :--- | :--- | :--- | :--- |
| 1 | Entire Postgres schema is inferred, not production DDL | High | Phase 1 verification task: diff inferred columns against live API payloads before irreversible migrations |
| 2 | Two of the 17 API block types were truncated from the retrieved extract | Medium | Re-query the block spec source at Phase 1 kickoff; reserve enum slots and keep blocks.type extensible (resolved by supplement note: prototype-block, landing-page-block) |
| 3 | Exact per-block config schema is undocumented | Medium | Version the config jsonb with schema_version; per-type Zod validators that evolve without migration |
| 4 | Artifact schema for memo-block and document-block undocumented | Medium | Emit both content (markdown) and file_url for every document block; treat content as canonical |
| 5 | Cited Supabase Realtime repo is a minimal chat demo, not a DAG example | Low | Cite only as proof Realtime is a real pattern; build the DAG cascade from first principles |
| 6 | YouTube Data API v3 free quota is a hard constraint; live calls returned 403/429 | Medium | Use yt-dlp or youtube-transcript for transcripts; restrict official API to cached lightweight metadata |
| 7 | github-mcp-server is a Go binary/Docker image, not npm | Low | Install as a sibling Docker service; complement with @octokit/rest later |
| 8 | tldraw requires more custom work than React Flow to enforce DAG semantics | Low | Commit to @xyflow/react for v1 |
| 9 | Chat (L1 Orchestrator) is not an API block type and does not create blocks directly | Low | Persist L1 as tier=1 task with no block_id; all canvas mutation flows through L2/L3 |
| 10 | Credit columns have no billing counterpart in a single-user deployment | Low | Use as local spend telemetry only; reconcile against runs.credits_consumed |
| 11 | No documented API parameter forces a specific model per block | Low | Model choice lives entirely in the router + agent tiers |
| 12 | "Computer Use" is not a documented Spine block type | Low | Treat web-research-block (BrowserUse) as the browser/computer-control primitive |

Acceptance definition for the rebuild: a single natural-language prompt, submitted through the Chat surface, produces a multi-block canvas whose document, spreadsheet and presentation blocks yield openable files; editing any upstream block reliably invalidates and re-runs its descendants; and an external MCP client can perform the same operations through the published tool surface. Anything short of all three is an incomplete Phase 3.

### Appendix A — Naming Reconciliation

**TABLE 18 — API IDENTIFIERS VERSUS APPLICATION-VISIBLE LABELS**

| API Block Type | App-Visible Name |
| :--- | :--- |
| text-block | Notes Block |
| web-research-block | BrowserUse Block |
| file-block | Files Block |
| (none — app-only surface) | Chat, Frames, Inputs, Folder, Folder Synthesis |

Maintaining this mapping in a single constants module prevents divergence between the Postgres enum, the canvas node label, and the MCP create_block type string.

### Appendix B — Ready-to-Paste Coding Agent Prompt

> Build a single-user, privately-deployed rebuild of Spine Canvas/Swarm on Next.js 14 (App Router) + TypeScript + Supabase (Auth, Postgres, Storage, Realtime) + Tailwind CSS, deployed on Vercel. Implement the 12-table Postgres schema (users, api_keys, integrations, canvases, blocks, edges, files, runs, run_artifacts, tasks, webhook_endpoints, webhook_deliveries) with single-owner RLS and a supabase_realtime publication on blocks/runs/tasks. Build an infinite canvas with @xyflow/react bound 1:1 to the blocks/edges tables, with Zustand client state kept fresh via Realtime. Implement all block types across four groups — generative document/data blocks (prompt, list, memo, document, excel, presentation, table, image, app), research/browsing blocks (deep-research, web-research), context/import blocks (text, web, yt, file), and app-only surfaces (Chat, Frames, Inputs, Folder, Folder Synthesis) — sharing a four-state (pending/running/completed/failed) lifecycle for all generative types. Implement a multi-provider model router over OpenAI, Anthropic, Google, and OpenRouter behind a single generate(kind, tier, payload) interface with fallback chains capped at two hops. Implement the Swarm Mode orchestrator as an L1/L2/L3 task tree persisted in tasks, with a topological dispatcher (bounded concurrency = 5) and a cascading re-run engine (forward BFS over edges + Realtime, 2s debounce). Implement a first-party MCP server on modelcontextprotocol/typescript-sdk v2 with StreamableHTTPServerTransport exposing create_canvas, create_run, get_run, create_block, connect_blocks, list_artifacts, and folder_synthesis tools. Follow the 4-phase / 15-week roadmap: (1) Foundation & Graph, (2) Generative Library, (3) Swarm & Research, (4) Integrations & Hardening. Treat every inferred Postgres schema column as unverified until checked against live API payloads in Phase 1.

### References
[1] Platform Architecture & Data Model — Consolidated Findings for Rebuild Spec
[2] Per-Block-Type Rebuild Spec (22 Rows: 17 Block Types + 5 App-Only Extras)
[3] Integrations, MCP, Canvas UI & Tech Stack — Connector-Grounded Findings
[5] Platform Architecture & Data Model (supplementary schema findings)
