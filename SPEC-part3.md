## 5. Multi-Model Orchestration Layer

Model selection is a first-class routing concern rather than a hardcoded constant. A single provider registry built on the Vercel AI SDK exposes four providers — OpenAI, Anthropic, Google, and OpenRouter — behind one generate(kind, tier, payload) interface.

### 5.1 Routing Matrix

**TABLE 8 — PROVIDER ROUTING MATRIX BY WORKLOAD CLASS**

| Workload | Primary Provider | Fallback | Rationale |
| :--- | :--- | :--- | :--- |
| prompt-block, text synthesis, memo/document prose | OpenAI | Anthropic → OpenRouter | Documented library for prompt-block |
| Structured JSON (list-block, table-block, excel data) | OpenAI structured outputs | Google → OpenRouter | Documented approach for list/table blocks |
| app-block HTML/code generation | Anthropic | OpenAI → OpenRouter | tlbrowse reference uses ANTHROPIC_API_KEY |
| Long-context folder synthesis and file digestion | Google (long-context) | Anthropic → OpenRouter | Map-reduce over many files favours large context |
| Deep research planning and synthesis | Anthropic | OpenAI | Multi-step agentic synthesis over browsed sources |
| Image generation (image-block) | Replicate API (Flux) | OpenAI DALL-E 3 | Documented options for image-block |
| Exotic / experimental models, cost arbitrage | OpenRouter | n/a | Single key, breadth of models |

### 5.2 Router Contract and Accounting
1. Resolve — dispatcher passes (block.type, task.agent_tier, canvas overrides) to the registry; returns a concrete model handle plus ordered fallback chain.
2. Guard — enforce a per-run token ceiling derived from runs.estimated_duration_ms and the block-type cost model.
3. Execute — stream tokens where the UI benefits; buffer where a file must be assembled atomically.
4. Fail over — on 429/5xx or schema-validation failure, advance one step down the fallback chain, capped at two hops, then mark the block failed.
5. Account — write prompt/completion tokens and provider cost into the task row; aggregate into runs.credits_consumed on completion.

Provider keys are server-only. No NEXT_PUBLIC_ prefix is ever applied to a model provider key; all generation traverses app/api route handlers.

---

## 6. Swarm Mode Orchestrator

Swarm Mode is the hierarchical agent system that turns one prompt into a populated canvas. It is persisted entirely in the tasks table.

### 6.1 Tier Model

**TABLE 9 — SWARM TIER MODEL MAPPED ONTO THE TASKS TABLE**

| Tier | Role | Canvas Authority | Persisted As |
| :--- | :--- | :--- | :--- |
| L1 Orchestrator | Conversational planner (the Chat surface) | None — does not create blocks directly | tasks row, tier = 1, no block_id |
| L2 Persona Agents | Decompose the goal into block-shaped subtasks and wire edges | Creates blocks and edges within runs.allowed_block_types | tasks rows, tier = 2, parent_task_id → L1 |
| L3 Workers | Execute a single block generation; bulk-expand list items | Writes content/url for exactly one block_id | tasks rows, tier = 3, block_id set |

### 6.2 Scheduling Algorithm
1. Plan — L1 receives prompt + agent_instructions + template and emits an ordered list of L2 persona tasks.
2. Decompose — each L2 persona emits L3 tasks, one per block to be created, declaring prospective edges.
3. Materialise — the dispatcher inserts blocks (pending) and edges, validating acyclicity.
4. Topologically sort — compute execution order over the edges table.
5. Fan out — execute the ready set concurrently with a bounded worker pool (default 5).
6. Bulk expand — a completed list-block may spawn child L3 tasks, one per item.
7. Settle — write runs.final_output, credits_consumed, and completed_at when all tasks resolve.

### 6.3 Cascading Re-Run on Upstream Change
Cascading re-run is a topological sort over the edges table combined with a Supabase Realtime postgres_changes subscription on blocks, so downstream nodes auto-invalidate and re-queue whenever an upstream block's content changes.

* Staleness is a derived UI state (dashed border + re-run affordance), not a new enum value.
* Debounce: coalesce rapid successive edits within a 2-second window before initiating a cascade.
* Cycle safety: edges are constrained to a DAG; an insert guard rejects cycle-forming edges.
* Evidence caveat: the cited Supabase Realtime reference is a minimal chat demo, not a DAG example — cited only as proof the pattern is real.

---

## 7. Deep Research Block

deep-research-block performs agentic research that browses the web and synthesises sources into a cited report. Its contract is minimal — { query, depth } in, { report_content, sources[] } out.

**TABLE 10 — DEEP RESEARCH PIPELINE STAGES**

| Stage | Operation | Implementation | Writes |
| :--- | :--- | :--- | :--- |
| 1 — Decompose | Expand query into depth × N sub-questions | LangChain / LlamaIndex planner (Anthropic primary) | tasks child rows (tier 3) |
| 2 — Retrieve | Search and fetch candidate sources | Tavily or SerpApi | In-memory candidate set |
| 3 — Extract | Convert pages to clean text | @mozilla/readability / cheerio; Playwright for JS-heavy pages | Cached extracts |
| 4 — Deduplicate | Canonicalise URLs, dedupe, rank | Embedding similarity + domain heuristics | Ranked source list |
| 5 — Synthesise | Compose multi-section report with citations | LLM synthesis over ranked extracts | blocks.content = report_content |
| 6 — Attribute | Emit full source list bound to citation markers | Structured JSON output validation | sources[] in payload |
| 7 — Publish | Optionally render report to .docx | docx (npm), reusing document-block generator | files + run_artifacts rows |

* Depth budget: depth 1/2/3 → 3/8/18 sub-questions and 8/25/60 fetched sources.
* UI: multi-section report with citations; stream stage transitions (retrieving → extracting → synthesising).
* Resumability: persist per-sub-question child tasks so a failed synthesis retries without repaying retrieval cost.
* Sibling relationship: interactive navigation (logins, click-throughs) delegates to web-research-block.
