## 4. Block Type Specifications

The research extract defines 22 block specification rows: 17 API-addressable block types plus 5 app-only extras. Fifteen API block types are itemised below across three functional groups; the two remaining API block types were truncated in the retrieved extract and are tracked as a Phase 1 verification item in Section 11. All generative blocks share the pending / running / completed / failed state machine.

### 4.1 Generative Document & Data Blocks

**TABLE 4 — GENERATIVE DOCUMENT AND DATA BLOCK SPECIFICATIONS**

| Block Type | Purpose | Input Schema | Output Schema | Generation Library / API |
| :--- | :--- | :--- | :--- | :--- |
| prompt-block | AI prompt / analysis | { "prompt": "string", "context_ids": ["string"] } | { "content": "string" } | OpenAI SDK / Vercel AI SDK |
| list-block | Expandable list | { "prompt": "string", "items": ["string"] } | { "list_items": ["string"] } | Vercel AI SDK structured JSON output |
| memo-block | Memo document (.docx) | { "prompt": "string", "sections": ["string"] } | { "file_url": "string", "content": "string" } | docx (npm) or python-docx |
| document-block | Report document (.docx) | { "prompt": "string", "formatting_rules": "string" } | { "file_url": "string", "content": "string" } | docx (npm) or python-docx |
| excel-block | Spreadsheet (.xlsx) | { "prompt": "string", "data": "array" } | { "file_url": "string", "sheets": ["string"] } | exceljs or pandas / openpyxl |
| presentation-block | Slide deck (.pptx) | { "prompt": "string", "slide_count": "number" } | { "file_url": "string", "slides": ["object"] } | pptxgenjs or python-pptx |
| table-block | Structured data table | { "prompt": "string", "columns": ["string"] } | { "rows": ["object"] } | Vercel AI SDK + AG Grid / TanStack Table |
| image-block | AI-generated image (.png) | { "prompt": "string", "style": "string" } | { "image_url": "string" } | Replicate API (Flux) or OpenAI DALL-E 3 |
| app-block | Web application (.html) | { "prompt": "string", "framework": "string" } | { "html_content": "string", "file_url": "string" } | Sandboxed iframe + LLM code-gen |

* UI behaviour: prompt-block renders a markdown text block; list-block renders an expandable list; memo/document-block render a document preview with download; excel-block renders a table preview with download; presentation-block renders a slide carousel preview; table-block renders an interactive data grid; image-block renders a thumbnail with download/expand; app-block renders a sandboxed iframe preview.
* Artifact pipeline: every block emitting file_url writes bytes to the private artifacts Supabase Storage bucket, inserts a files row, and — inside a run — a run_artifacts row.
* Gap: the exact per-block config schema and the artifact schema for memo-block/document-block are undocumented; validators are application-owned and versioned.

### 4.2 Research & Browsing Blocks

**TABLE 5 — AGENTIC RESEARCH AND BROWSING BLOCK SPECIFICATIONS**

| Block Type | Purpose | Input Schema | Output Schema | Generation Library / API |
| :--- | :--- | :--- | :--- | :--- |
| deep-research-block | Deep research report | { "query": "string", "depth": "number" } | { "report_content": "string", "sources": ["string"] } | LangChain / LlamaIndex with Tavily or SerpApi |
| web-research-block | Browser navigation / scraping ("BrowserUse Block") | { "task": "string", "start_url": "string" } | { "extracted_data": "string", "screenshots": ["string"] } | Browserbase, Browserless, or Playwright + AI agent |

* deep-research-block renders a multi-section report with citations; web-research-block renders a live browser view or step-by-step action log.
* Both are long-running and must execute as background functions with heartbeat writes to blocks.status.

### 4.3 Context & Import Blocks

**TABLE 6 — CONTEXT INGESTION BLOCK SPECIFICATIONS**

| Block Type | Purpose | Input Schema | Output Schema | Generation Library / API |
| :--- | :--- | :--- | :--- | :--- |
| text-block | Static markdown/text note ("Notes Block") | { "text": "string" } | { "content": "string" } | React Markdown / standard text area |
| web-block | Web URL reference | { "url": "string" } | { "page_content": "string", "metadata": "object" } | Cheerio / Puppeteer |
| yt-block | YouTube video reference | { "video_url": "string" } | { "transcript": "string", "metadata": "object" } | youtube-transcript (npm) or yt-dlp |
| file-block | File upload reference ("Files Block") | { "file_data": "binary", "filename": "string" } | { "file_url": "string", "extracted_text": "string" } | Supabase Storage + PDF.js / text extractors |

### 4.4 App-Only / UI-Layer Blocks

Five surfaces appear in the application but are not API block types. They must still be modelled in the blocks table (with a ui_only flag).

**TABLE 7 — APP-ONLY BLOCK SURFACES**

| Surface | Purpose | Input Schema | Output Schema | Library / Implementation |
| :--- | :--- | :--- | :--- | :--- |
| Chat | Conversational interface (L1 Orchestrator) | { "messages": ["object"] } | { "response": "string" } | Vercel AI SDK (useChat) |
| Frames | Organise workspace into sections/groups | { "title": "string", "bounds": "object", "child_node_ids": ["string"] } | { "layout_state": "object" } | React Flow / tldraw grouping |
| Inputs | Define variables flowing through blocks | { "variable_name": "string", "value": "any" } | { "injected_context": "object" } | React Hook Form |
| Folder | Group files | { "folder_name": "string", "file_ids": ["string"] } | { "aggregated_files": ["object"] } | React state |
| Folder Synthesis | Map-reduce prompt across folder files | { "prompt": "string", "folder_id": "string" } | { "synthesized_output": "string" } | LangChain MapReduceDocumentsChain |

* Chat is not an API block type; the L1 Orchestrator does not operate on the canvas or create blocks directly.
* Frames, Inputs, Folder are static, pure layout/config schemas with no AI generation.
* Folder Synthesis is the one app-only surface with a real generation lifecycle (pending/running/completed/failed).
