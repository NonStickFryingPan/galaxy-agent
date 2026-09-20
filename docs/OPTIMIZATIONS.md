# ⚡ Galaxy-Agent Architectural Optimizations: The Unified Blueprint
## Cross-System Synthesis of Modern MCP Tooling, Progressive Disclosure & Code Execution

---

## Executive Summary

Connecting LLMs to real-world software catalogs reveals a fundamental architectural tension: **the Tool Scaling Wall**. Modern platforms expose thousands of capabilities (Galaxy exposes 3,890+ tools on `usegalaxy.eu`; GitHub/Stripe/Jira expose 1,600+ APIs). Naively injecting all tool schemas into the model context upfront consumes **100k to 500k+ tokens per request**, destroys prompt caching, degrades tool-selection accuracy by 25–40%, and costs hundreds of dollars per evaluation run.

Independent teams—Anthropic, Maxim AI (Bifrost), UsefulSoftwareCo (Executor.sh), Context Mode (`mksglu`), and Cloudflare—converged on an identical paradigm shift between late 2025 and 2026:

> **Do not stream tool definitions or raw intermediate data into LLM context. Treat the model as a software engineer writing sandboxed orchestration code against lazily discovered, typed APIs with storage handles.**

This document synthesizes findings across these systems, maps their overlapping breakthroughs, selects high-confidence architectural patterns, and provides an implementation blueprint tailored to **`galaxy-agent`**—a terminal-native orchestrator for bioinformatics workflows.

---

## 1. Cross-System Architectural Comparison Matrix

| Dimension | Anthropic Tool Search Tool | Bifrost "Code Mode" (Maxim AI) | Executor.sh (`UsefulSoftwareCo`) | Context Mode (`mksglu`) | Cloudflare Code Mode | Traditional Static MCP |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Primary Paper / Origin** | Official API Beta (`advanced-tool-use-2025-11-20`) | `maximhq/bifrost` & `bifrost-benchmarking` | `UsefulSoftwareCo/executor` & Blog Analysis | `mksglu/context-mode` (OSS ELv2 + Insight) | Kenton Varda (Cloudflare Dynamic Workers) | MCP Specification Baseline (2024–2025) |
| **Primary Problem Solved** | Upfront schema bloat & tool hallucination | Multi-turn round trips & schema bloat | Fragile CLI scripts, auth leaks, context bloat | Tool output bloat (50KB+ terminal logs) | Exposing 2,594 REST endpoints to workers | Simple 1-to-1 tool dispatch |
| **Token Reduction** | **>85%** (77k → 8.7k tokens) | **92%** (75.1M → 5.4M tokens) | **99.6%** (278k → 1,044 tokens) | **Variable** (saves ~80% of stdout tokens) | **99.9%** (1.17M → 1.1k tokens) | 0% (Baseline) |
| **Cost Reduction** | Proportional to prompt size | **$377 → $29** (-92.3%) | Negligible prompt input cost | Reduced re-prompt costs | Reduced Cloudflare AI Gateway spend | Baseline |
| **Discovery Mechanism** | Server-side BM25 / Regex (`defer_loading: true`) | Virtual filesystem stubs (`listToolFiles`, `getToolDocs`) | Lazy JS Proxy (`tools.search`, `tools.describe`) | SQLite FTS5 BM25 search (`ctx_search`) | Dynamic 3-tool API (`search`, `docs`, `execute`) | Full upfront catalog loading (`tools/list`) |
| **Execution Runtime** | Standard client execution | Sandboxed **Google Starlark** (in Go) | Sandboxed **QuickJS WASM** or Cloudflare Worker | Host subprocess (Node/Python/etc.) | Ephemeral **V8 Isolates** (WorkerLoader) | Host-side direct JSON-RPC call |
| **Data Plane / Intermediate Results**| Returned into conversation history | Processed in Starlark isolate memory | Processed in QuickJS WASM; emitted via `emit()` | Intercepted & written to local SQLite FTS5 | Processed inside V8 Isolate memory | Inlined into LLM context window |
| **Data Handles** | Not standardized | Virtual filesystem paths | `handle://...` KV / blob pointers | SQLite document IDs | Cloudflare R2 / KV bindings | Inlined raw payloads |
| **Prompt Cache Impact** | **Preserved** (inline history injection) | **Preserved** (4 static meta-tools) | **Preserved** (1-3 static meta-tools) | **Preserved** (tool output replaced by stubs) | **Preserved** (static meta-tools) | **Broken** on dynamic server changes |
| **Safety / HITL** | Client-level confirmation | Hermetic sandbox (no IO/net) | Durable `Run` pause/resume; auth isolation | Local file permissions | Isolate resource limits & timeouts | Manual approval prompts |
| **Model Portability** | Anthropic Claude only (server feature) | Universal (any model) | Universal (any MCP-capable model) | Universal (any model) | Universal (any model) | Universal |

---

## 2. Core Overlapping Breakthroughs

Across all five implementations, five universal architectural laws emerge:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          UNIFIED OPTIMIZATION STACK                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│ 1. Schema Lazy Loading:  Catalog -> Index -> 4 Meta-Tools -> Typed Stubs (.pyi) │
│ 2. Pipeline Execution:   Replace 15 JSON-RPC turns with 1 sandboxed script      │
│ 3. Memory Isolation:     Filtering/aggregation happens inside isolate memory    │
│ 4. Storage Handles:      Genomic datasets & large tables remain out-of-band     │
│ 5. Cache Preservation:   Never alter system prompt; append discoveries inline   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Breakthrough 1: Dynamic Lazy Discovery (Schema Deflation)
* **The Antipattern:** Sending 100+ JSON Schema objects in the API request headers or system prompt.
* **The Proven Fix:** Present the model with a fixed, small surface (1 to 4 meta-tools). The catalog is stored out-of-band in an in-memory index or local SQLite database. The model queries tools only when intent requires it.
* **Representational Efficiency:** JSON Schema is 5–10x more verbose than equivalent Python type stubs (`.pyi`) or TypeScript definitions (`.d.ts`). Exposing stubs instead of JSON schemas saves 80%+ tokens even when definitions are inspected.

### Breakthrough 2: Code Mode over Multi-Turn JSON-RPC
* **The Antipattern:** For a 5-step pipeline (Extract → QC → Filter → Align → Summarize), making 5 round-trip LLM generations where each step's output is serialized into JSON-RPC, fed back to the LLM, and used to generate the next call.
* **The Proven Fix:** The model writes a short orchestration script in Python, TypeScript, or Starlark. The script executes locally in a lightweight sandbox. Intermediate variables reside in isolate memory. The LLM only sees the script and the final emitted summary.

### Breakthrough 3: Context Shielding via Storage Handles
* **The Antipattern:** Returning 50KB–500MB tool results (such as raw SAM/BAM alignments, VCF records, or tabular matrices) directly into the message context.
* **The Proven Fix:** All heavy payloads are retained in external storage (Galaxy History or local SQLite cache). The tool returns a lightweight opaque pointer (e.g., `galaxy://histories/h123/contents/d456` or `handle://tmp_fastqc_789`). Sandboxed code inspects metadata or slices via utility functions (`head()`, `summary()`, `grep()`) without bloating the prompt.

### Breakthrough 4: Immutable System Prompt & Prompt Cache Preservation
* **The Antipattern:** Mutating the system prompt dynamically as tools are enabled or discovered, which busts KV-cache prefixes across Claude, OpenAI, and local engines.
* **The Proven Fix:** Keep the system prompt completely static (holding only resident meta-tools). Hydrate newly discovered tools as inline conversation blocks (`tool_reference` or message attachments), preserving 100% of KV-cache hits.

### Breakthrough 5: Hermetic Sandboxing & Secret Decoupling
* **The Antipattern:** Running code directly in the host shell or passing raw API keys into the execution environment.
* **The Proven Fix:** Use deterministic, fast-booting interpreters (Google Starlark or QuickJS WASM). Prohibit imports, network access, and file I/O within the sandbox. The host environment injects authentication tokens at the dispatch boundary when the sandbox invokes an MCP bridge tool.

---

## 3. High-Confidence Architectural Blueprint for `galaxy-agent`

Galaxy 26.1 presents the extreme test case for these patterns:
* **Tool Shed Scale:** 8,000+ tool repositories; 3,890+ tools on `usegalaxy.eu`.
* **Schema Verbosity:** Galaxy tool definitions are XML-based; converted naively to JSON Schema, a single tool (e.g., `hisat2` or `trimmomatic`) can reach 15,000+ tokens.
* **Dataset Volume:** FASTQ, BAM, and CRAM files are 5GB to 100GB+ each.

The unified optimization blueprint for `galaxy-agent` combines the best of all five systems:

```
                               ┌────────────────────────────────┐
                               │   User Terminal REPL (TUI)     │
                               │   Textual + Rich + Plotext     │
                               └───────────────┬────────────────┘
                                               │
                                               ▼
                               ┌────────────────────────────────┐
                               │       galaxy-agent Core        │
                               │   Static Prompt: 4 Meta-Tools  │
                               └───────┬────────────────┬───────┘
                                       │                │
            Tool Discovery (Tier 1 & 2)│                │ Orchestration Code (Tier 3)
                                       ▼                ▼
┌──────────────────────────────────────────────┐ ┌──────────────────────────────────────────┐
│             Local Index Engine               │ │        Hermetic Python Sandbox           │
│ - In-memory BM25Okapi (Galaxy tools index)   │ │ - Sub-millisecond startup (Starlark/WASM)│
│ - EDAM Bio-Ontology & GTN workflow tags      │ │ - BioBlend / Galaxy-MCP Lazy Proxy       │
│ - Regex tool ID matching                     │ │ - Zero OS / Net / File I/O access        │
└──────────────────────┬───────────────────────┘ └────────────────────┬─────────────────────┘
                       │                                              │ Dispatch
                       ▼                                              ▼
┌──────────────────────────────────────────────┐ ┌──────────────────────────────────────────┐
│          JIT Schema / Stub Engine            │ │           Human-In-The-Loop (HITL)       │
│ - Formats Galaxy XML into lean .pyi stubs    │ │ - Gated execution for Slurm/HPC jobs     │
│ - Injects curated bio-parameter examples     │ │ - Parameter diff modal & quota checks    │
└──────────────────────────────────────────────┘ └────────────────────┬─────────────────────┘
                                                                      │ Approved Jobs
                                                                      ▼
                                                 ┌──────────────────────────────────────────┐
                                                 │       Galaxy 26.1 Compute Cluster        │
                                                 │       Cloud / HPC (usegalaxy.org/.eu)    │
                                                 │  Data stays on cluster (galaxy://hda_id) │
                                                 └──────────────────────────────────────────┘
```

### Module A: The 4 Resident Meta-Tools (Fixed Footprint: ~1,100 tokens)
Instead of streaming tools, `galaxy-agent` initializes LLM sessions with only four resident tools:

1. `galaxy_search_tools(query: str, category: Optional[str] = None) -> List[ToolSummary]`
   * Searches the in-memory BM25 index of tools, returning concise tuples: `(tool_id, name, version, one_line_summary)`.
2. `galaxy_describe_tool(tool_id: str) -> str`
   * Compiles verbose Galaxy XML into a concise Python type stub (`.pyi`) with `input_examples` for tricky genomic parameters (e.g., paired-end collections, strandedness).
3. `galaxy_execute_script(code: str) -> ExecutionResult`
   * Executes a sandboxed Python script against a lazy Galaxy proxy (`galaxy.tools.*`). Runs multi-step pipelines without round-trip LLM latency.
4. `galaxy_inspect_dataset(handle: str, view_type: str = "head") -> DatasetSummary`
   * Inspects datasets retained in Galaxy history. Returns formatted tabular heads, VCF summary statistics, or ASCII plot representations (using `Plotext`) without loading multi-gigabyte files into context.

### Module B: In-Memory Hybrid Tool Indexing with Bio-Ontology Enrichment
* **Engine:** Fast in-memory `BM25Okapi` paired with token regex.
* **Corpus Augmentation:** Index tool names and descriptions alongside:
  * **EDAM Ontology:** Map biological operations (e.g., *operation_3198: Read pre-processing*) and data formats (*format_1930: FASTQ*).
  * **Galaxy Training Network (GTN) Tags:** Index workflow step sequences (e.g., `["qc", "trim", "align", "count", "dge"]`).
* **Result:** Resolves vocabulary mismatches (e.g., user asks for "expression analysis", index identifies `deseq2` and `featureCounts` even if tool summaries only say "differential analysis of count data").

### Module C: Sandboxed Pipeline Execution ("Bioinformatics Code Mode")
* **Multi-Step Efficiency:** Instead of asking the model to invoke `fastqc`, wait, inspect results, then invoke `hisat2`, the model emits a single Python orchestration script:
  ```python
  # Sandboxed execution block emitted by LLM
  history = galaxy.get_history(name="Patient_402_RNASeq")
  raw_reads = history.get_dataset(name="reads.fastq.gz")
  
  # Run FastQC and Trimmomatic in pipeline
  qc = galaxy.tools.fastqc(input=raw_reads)
  trimmed = galaxy.tools.trimmomatic(
      input=raw_reads, 
      slidingwindow="4:20", 
      illuminaclip="TruSeq3:2:30:10"
  )
  
  # Return handle references, not gigabytes of data
  emit({"qc_report": qc.handle, "trimmed_reads": trimmed.handle})
  ```
* **Performance Gain:** Reduces 8–12 round-trip API calls to a single generation, saving minutes of latency and tens of thousands of intermediate tokens.

### Module D: Context Shielding & Data Handles
* **Data Abstraction:** Datasets are represented as `galaxy://histories/{history_id}/contents/{dataset_id}`.
* **Local Caching:** Textual logs (stdout/stderr) and tabular tables (e.g., DESeq2 results tables) are cached into a local SQLite database with `FTS5` full-text indexing (`~/.galaxy-agent/cache.db`).
* **Targeted Querying:** When the model needs to investigate an error or check a p-value, it queries the SQLite cache via BM25 instead of loading the entire log or table.

### Module E: Human-In-The-Loop (HITL) Gating & Quota Protection
* Borrowing Executor.sh's `Run` state abstraction and policy model:
  * **Read-Only / Metadata Operations:** Auto-approved (`galaxy_search_tools`, `galaxy_describe_tool`, `galaxy_inspect_dataset`).
  * **Compute / HPC Operations:** Intercepted by `galaxy-agent`'s Textual TUI before cluster dispatch:
    * Displays parameter diff modal.
    * Checks server storage quota (warns if quota > 80% of 250 GB).
    * Requires user keypress `[y]` to submit Slurm cluster jobs.

---

## 4. Implementation Priority & Roadmap

| Priority | Component | Objective | Target Metric |
| :--- | :--- | :--- | :--- |
| **P0** | **4 Meta-Tools Engine** | Replace full schema loading with `search`, `describe`, `execute`, `inspect` | Context footprint fixed at <1,500 tokens |
| **P0** | **In-Memory BM25 Index** | Index all 3,890+ tools on target Galaxy server with EDAM ontology tags | <50ms query time, >90% tool retrieval accuracy |
| **P1** | **JIT `.pyi` Stub Generator** | Convert Galaxy XML parameter trees into minimal, typed Python stubs with examples | <400 tokens per tool definition vs 10k XML |
| **P1** | **Hermetic Code Runner** | Sandboxed execution runtime (Starlark or QuickJS WASM) for multi-tool scripts | Zero token leakage of intermediate datasets |
| **P2** | **SQLite FTS5 Output Cache** | Local full-text indexing for QC summaries and job error logs | Context-saving tool inspection via targeted search |
| **P2** | **HITL & Quota Guard** | Interactive confirmation modal before cluster dispatch; auto-purge on quota | Prevent aborted runs due to 250GB disk caps |

---

## 5. Primary References & Citations

1. **Anthropic Engineering:**
   * *Code Execution with MCP: Building More Efficient Agents* (Nov 4, 2025): `https://www.anthropic.com/engineering/code-execution-with-mcp`
   * *Advanced Tool Use: Tool Search Tool & Deferred Loading* (Nov 20, 2025): `https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool`
   * *Model Context Protocol Specification & Client Best Practices*: `https://modelcontextprotocol.io/docs/develop/clients/client-best-practices`
2. **Maxim AI (Bifrost):**
   * *Bifrost Gateway & Code Mode Architecture*: `https://github.com/maximhq/bifrost`
   * *Benchmarking MCP Token Reduction (508 Tools Suite)*: `https://github.com/maximhq/bifrost-benchmarking`
   * *Google Starlark Execution Runtime*: `https://github.com/google/starlark-go`
3. **UsefulSoftwareCo (Executor.sh):**
   * *Why MCP Had Growing Pains (Rhys Sullivan)*: `https://executor.sh/blog/why-mcp-had-growing-pains/`
   * *Executor Monorepo (QuickJS WASM & Dynamic Worker Runtimes)*: `https://github.com/UsefulSoftwareCo/executor`
   * *QuickJS Emscripten WASM Sandbox*: `https://github.com/justjake/quickjs-emscripten`
4. **Context Mode (`mksglu`):**
   * *Context Mode Platform & SQLite FTS5 Output Sandboxing*: `https://context-mode.com/`
   * *Context Mode Source Repository*: `https://github.com/mksglu/context-mode`
5. **Cloudflare:**
   * *Kenton Varda on Code Mode & Dynamic Worker Loaders*: `https://blog.cloudflare.com/`
   * *Cloudflare MCP Server & Dynamic Workspaces*: `https://github.com/cloudflare/mcp`
6. **Galaxy Project:**
   * *Galaxy 26.1 Release Notes & Native MCP Integration*: `https://docs.galaxyproject.org/`
   * *Galaxy MCP Server Reference Implementation*: `https://github.com/galaxyproject/galaxy-mcp`
   * *BioBlend Python SDK for Galaxy*: `https://github.com/galaxyproject/bioblend`
