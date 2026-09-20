## 8. Infinite Canvas UI

React Flow (@xyflow/react) is the primary canvas library because it maps 1:1 onto the DAG data model; tldraw is the alternative if freeform/sketch-style canvases become a requirement, but it demands more custom work to enforce DAG semantics.

**TABLE 11 — REACT FLOW TO POSTGRES BINDING**

| React Flow Concept | Database Binding | Notes |
| :--- | :--- | :--- |
| Node.id | blocks.id | Direct identity mapping |
| Node.type | blocks.type | One custom React Flow node component per block type |
| Node.position / measured | blocks.position (jsonb) | Persisted on drag-end, debounced |
| Node.data | blocks.name, status, content, url, config | Status drives the badge and border treatment |
| Edge.source / Edge.target | edges.source_id / edges.target_id | Insert guarded against cycles |
| Viewport / zoom / pan | canvases.layout (jsonb) | Restored on canvas open |
| Group / parent node | Frames block (ui_only) with bounds + child_node_ids | Pure layout schema, no AI generation |

### 8.1 Reference Implementations
* react-logic-workflow-builder — @xyflow/react with Zustand and 10 unique node types; confirms React Flow + Tailwind + Zustand is current and well-supported.
* react-flow-pipeline-builder — React + React Flow with a FastAPI backend, useful as a shape reference for pipeline execution APIs.
* workflow-builder — compact reference for node palette and edge-editing ergonomics.
* tlbrowse — "Generate imagined websites on an infinite canvas." Clicking a link generates a new connected page, a direct analog to cascading block generation on edge connection; confirms ANTHROPIC_API_KEY usage, bun package manager, and model-call logic in app/api/html/route.ts.

### 8.2 Interaction Requirements
* Drag-to-connect creates an edges row and immediately offers "generate downstream block".
* Per-block status chrome: pending (muted), running (animated), completed (solid), failed (error state with retry).
* Realtime hydration: subscribe to postgres_changes on blocks, runs, tasks; reconcile into the Zustand store.
* Download affordances rendered directly on memo/document/excel/presentation/image nodes, resolving files.storage_path to a signed URL.
* app-block previews always rendered in a sandboxed iframe with a restrictive sandbox attribute and no same-origin privileges.

---

## 9. Integrations and MCP Layer

### 9.1 MCP Server
The first-party MCP server is built on the official modelcontextprotocol/typescript-sdk (v2, 2026-07-28 MCP spec), running on Node.js/Bun/Deno with optional Express/Fastify/Hono middleware. Swap StdioServerTransport for StreamableHTTPServerTransport, exposed from a Next.js route handler for the hosted, multi-client case.

**TABLE 12 — MCP TOOL SURFACE EXPOSED BY THE REBUILD**

| Exposed MCP Tool | Signature (Zod) | Backing Operation |
| :--- | :--- | :--- |
| create_canvas | { name: string } | INSERT canvases |
| create_run | { canvas_id, prompt, template?, allowed_block_types?, agent_instructions? } | INSERT runs, invoke Swarm planner |
| get_run | { run_id } | SELECT runs + tasks tree |
| create_block | { canvas_id, type, config } | INSERT blocks, enqueue L3 task |
| connect_blocks | { canvas_id, source_id, target_id } | INSERT edges with cycle guard |
| list_artifacts | { run_id } | SELECT run_artifacts, return signed URLs |
| folder_synthesis | { folder_id, prompt } | Map-reduce across folder files |

### 9.2 Connectors

**TABLE 13 — CONNECTOR RECOMMENDATIONS**

| Connector | Recommendation | Reference | Key Detail |
| :--- | :--- | :--- | :--- |
| GitHub | Use github/github-mcp-server as authoritative reference, alongside a hand-rolled @octokit/rest connector | github/github-mcp-server, 33k★ | Go binary / Docker image, not npm; invoked via MCP stdio or Docker |
| Gmail / Sheets / Workspace | googleapis + google-auth-library OAuth2Client pattern | quinnjr/google-mcp; karthikcsq/google-tools-mcp | Tokens persisted in the generic integrations table |
| Web / PDF import | Playwright / Readability / cheerio | — | Powers web-block content extraction |
| YouTube | Do NOT rely on YouTube Data API v3 for transcripts; use yt-dlp or youtube-transcript npm package | yt-dlp; youtube-transcript | Reserve official API for lightweight metadata only |

First-hand quota evidence: live connector calls to YouTube Data API v3 returned HTTP 403 (captions.list) and HTTP 429 (videos.list). Free quota is 10,000 units/day; search.list=100, captions.list=50 units per call — a hard practical constraint at volume.

### 9.3 OAuth Token Handling
* All connectors write to the integrations table: provider, oauth_access_token, oauth_refresh_token, scopes, status, connected_at.
* Access tokens encrypted at rest; refresh happens lazily on 401 with a single retry; status flips to needs_reauth on failure.
* Scope minimisation: request read-only Google scopes unless a write-capable block type is explicitly enabled.
* GitHub connector may be satisfied by installing github/github-mcp-server locally as a Docker image.

---

## 10. Tech Stack and Environment Variables

The confirmed stack is Next.js 14 App Router + TypeScript + Supabase (Auth, Postgres, Storage, Realtime) + Tailwind + Vercel.

**TABLE 14 — CONFIRMED TECHNOLOGY SELECTIONS**

| Concern | Choice |
| :--- | :--- |
| Framework | Next.js 14 App Router + TypeScript |
| Styling | Tailwind CSS |
| Canvas | @xyflow/react (React Flow); tldraw as alternative |
| Client state | Zustand |
| Backend / DB | Supabase — Auth, Postgres, Storage, Realtime |
| Hosting | Vercel (serverless + background functions) |
| Model access | Vercel AI SDK provider registry over OpenAI / Anthropic / Google / OpenRouter |
| Artifact generation | docx, exceljs, pptxgenjs (Node-first) |
| Research / agents | LangChain or LlamaIndex + Tavily or SerpApi |
| Browser automation | Browserbase, Browserless, or Playwright + AI agent |
| MCP | modelcontextprotocol/typescript-sdk v2 |
| Package manager | pnpm (bun acceptable) |

### 10.1 Environment Variables

**TABLE 15 — ENVIRONMENT VARIABLE CONTRACT**

| Variable | Scope | Purpose |
| :--- | :--- | :--- |
| NEXT_PUBLIC_SUPABASE_URL | Client + Server | Supabase project endpoint |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | Client + Server | RLS-scoped client access and Realtime subscriptions |
| SUPABASE_SERVICE_ROLE_KEY | Server only | Migrations, storage writes, dispatcher writes bypassing RLS |
| OPENAI_API_KEY | Server only | prompt-block text generation and structured JSON output |
| ANTHROPIC_API_KEY | Server only | app-block code generation and research synthesis |
| GOOGLE_GENERATIVE_AI_API_KEY | Server only | Long-context synthesis and folder synthesis map-reduce |
| OPENROUTER_API_KEY | Server only | Breadth/fallback provider |
| REPLICATE_API_TOKEN | Server only | image-block generation via Flux |
| TAVILY_API_KEY / SERPAPI_API_KEY | Server only | deep-research-block retrieval |
| BROWSERBASE_API_KEY / BROWSERBASE_PROJECT_ID | Server only | web-research-block hosted browser sessions |
| GOOGLE_OAUTH_CLIENT_ID / _SECRET / _REDIRECT_URI | Server only | googleapis OAuth2Client flow |
| YOUTUBE_API_KEY | Server only | Lightweight YouTube metadata only |
| YT_DLP_PATH | Server only | Binary path for yt-dlp transcript extraction |
| GITHUB_TOKEN / GITHUB_MCP_DOCKER_IMAGE | Server only | GitHub connector / github-mcp-server |
| MCP_TRANSPORT / MCP_PUBLIC_URL | Server only | Selects stdio vs StreamableHTTP transport |
| TOKEN_ENCRYPTION_KEY | Server only | AES key for encrypting integrations OAuth tokens |
| WEBHOOK_SIGNING_SECRET | Server only | HMAC signing for webhook_deliveries payloads |
| NEXT_PUBLIC_APP_URL | Client + Server | OAuth redirects, signed-URL bases, webhook callbacks |
| MAX_CONCURRENT_BLOCK_WORKERS | Server only | Dispatcher fan-out bound (default 5) |
