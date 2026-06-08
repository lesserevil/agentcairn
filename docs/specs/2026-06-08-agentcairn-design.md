# agentcairn — Design Spec

**Status:** Draft for review · **Date:** 2026-06-08
**Product/org/package:** `agentcairn` · **CLI command:** `cairn`

> A local-first agent-memory system where a plain-markdown Obsidian vault is the
> **source of truth**, and a **rebuildable embedded DuckDB index** gives the agent
> harness fast hybrid retrieval. Daemonless: a `cairn` CLI + an on-demand MCP server.
> You read, edit, and add memory by hand in Obsidian; the agent reads it all back.

---

## 1. Goals & non-goals

**Goals**
- **Local-first & transparent:** memory is human-readable markdown the user owns, browses, and edits — no opaque DB as the source of truth.
- **Competitive retrieval:** hybrid vector + BM25 + graph + recency/importance, on par with agentmemory/Mem0/Letta/Zep.
- **Robust by construction:** eliminate the failure classes seen in hook/daemon systems (silent capture loss, store-location footguns, provider-quota stalls, version drift).
- **Knowledge fusion:** the agent draws on the user's hand-written notes as well as its own memories — one linked graph.

**Non-goals (v1)**
- Not a managed cloud service; not multi-user/team memory.
- Not an Obsidian plugin in v1 (the in-browser surface is a v2 fast-follow; see §11).
- No always-on daemon, no external database, no required cloud provider.

## 2. Competitive context (and the honest wedge)

The closest competitor is the same-named `rohitg00/agentmemory`, which already has hybrid+graph retrieval, local embeddings, Ebbinghaus decay, Obsidian *export*, a viewer, and secret redaction. **Our differentiation is narrow and must stay crisp and demonstrable**, or agentcairn reads as a reskin. The wedge — which no competitor occupies (Mem0/Zep are cloud DBs; Letta/agentmemory are DB-as-truth with files as export-only):

1. **Vault is the source of truth** (not a one-way export). Hand-edit a fact in Obsidian → the index honors it.
2. **Index is disposable/rebuildable** → survives model upgrades, corruption, schema changes, and tool uninstall with zero data loss.
3. **Non-lossy by construction** → the full note is always on disk; distillation only *adds* derived notes that link back to the source, never replaces it (structural immunity to Mem0's acknowledged lossy-extraction failure).
4. **Free deterministic graph** from Obsidian `[[wikilinks]]` + frontmatter (no LLM, no hallucinated entities).
5. **Daemonless, zero external DB** — one embedded DuckDB file does vector + BM25 + SQL-graph (simpler than Neo4j/pgvector/Qdrant/Chroma).

**Positioning:** lead with transparency/ownership/robustness, **not** a single headline benchmark number (vendor LoCoMo/LongMemEval numbers are self-reported and actively disputed — see §13).

## 3. Architecture

```
Obsidian vault (markdown + frontmatter + [[links]])   ← SOURCE OF TRUTH
        │  parse (deterministic)            ▲ write (agent + human)
        ▼                                   │
  cairn.index  ──build──►  DuckDB index (DISPOSABLE cache, LOCAL disk)
                            • notes / chunks / embeddings FLOAT[N] / links / scores
                            • VSS HNSW (in-memory, rebuilt on spawn) + FTS BM25 + recursive-CTE graph
        ▲                                   │
        │ reconcile-on-spawn (mtime/hash)   ▼ READ_ONLY queries
  cairn.ingest ◄── transcripts (~/.claude/projects/**/*.jsonl, out-of-band, append-only)
        │
  cairn.mcp (on-demand, localhost) ──► harness tools: remember / recall / search / build_context / recent
```

Only two roles ever touch the DB: a **single-writer CLI** (ingest/index/reindex) and a **READ_ONLY MCP server** (multi-reader-safe). This sidesteps DuckDB's single-writer-XOR-multi-reader file lock. The `.duckdb` lives on **local disk, never inside the synced vault folder**.

## 4. Robustness model (the differentiator)

- **Index = disposable cache.** Any corruption / schema change / embedding-model swap → `cairn reindex` rebuilds from markdown. This single decision neutralizes DuckDB's *experimental* HNSW persistence risk **and** its non-auto-updating FTS.
- **In-memory HNSW rebuilt at MCP spawn** for small/medium vaults (avoids `hnsw_enable_experimental_persistence` entirely). Persisted HNSW is opt-in for very large vaults, always with rebuild-on-corruption fallback. *(Open: benchmark rebuild/cold-start latency vs. vault size to set the threshold — §14.)*
- **Reconcile-on-spawn** (the daemonless tax, the critical correctness path): cheap mtime/content-hash diff → re-parse/re-embed only changed notes → rebuild FTS + HNSW before serving the first query. Atomic.
- **Reader/writer policy:** MCP opens `access_mode=READ_ONLY`; all writes funnel through one short-lived CLI process. On "Could not set lock on file," surface a clear message + READ_ONLY fallback — never a silent dead retrieval.
- **Degradation ladder (never silently dead):** full hybrid → local-only embeddings if no cloud → brute-force `array_cosine_distance` if HNSW absent → **BM25-only floor** (proven ~86% R@5 alone).

## 5. Components (Python modules)

| Module | Responsibility | Key deps |
|---|---|---|
| `cairn.vault` | **Hardest component.** Parse/write markdown: frontmatter, body, `[[wikilinks]]`, observation/relation lines, inline fields. Preserve human edits, frontmatter order, unresolved forward-refs; rewrite links on move. | `python-frontmatter`, `markdown-it-py` |
| `cairn.index` | Vault → DuckDB tables; reconcile-on-spawn; build FTS + in-memory HNSW; `reindex`. | `duckdb` |
| `cairn.embed` | Pluggable embeddings; records model id+dim; asymmetric query/passage prefixes. | `fastembed` (default); opt `sentence-transformers`/Ollama/cloud |
| `cairn.search` | Hybrid query, RRF fusion, graph-boost, scoring, degradation ladder, optional rerank, progressive disclosure. | `duckdb` |
| `cairn.ingest` | Per-harness transcript locator → redact → dedup → importance-gate → distill → write derived notes. | stdlib |
| `cairn.mcp` | MCP server, READ_ONLY DuckDB, tool surface; launched via `uvx`. | `mcp` / FastMCP |
| `cairn` (CLI) | `remember / recall / search / ingest / sweep / reindex / reflect / defrag / serve / doctor`. | `typer` |

**Build order:** `vault` → `index` → `embed` → `search` → `ingest` → `mcp`/CLI. Invest most in `cairn.vault` first.

## 6. Markdown contract (human-browsable AND machine-parseable)

Adopt **basic-memory's conventions verbatim** (drop-in familiarity) + **Dataview/Datacore-compatible** inline fields (so existing Obsidian tooling indexes the same vault).

- **One note per memory/entity.** Frontmatter:
  - Core: `title`, `type`, `permalink` (slug backing a `memory://` URL scheme), `tags`, `created`.
  - cairn fields: `source` (wikilink / `memory://` back to origin), `importance` (0–1), `last_accessed`, `access_count`, decay metadata (e.g. `half_life`).
  - **Reserved (v1.1):** `valid_from`, `valid_until`, `superseded_by` (bi-temporal validity).
- **Observations:** `- [category] content #tag (optional context)`
- **Relations:** `- relation_type [[Target]]`; bare `[[Target]]` ⇒ implicit `links_to`. These are the **free, deterministic graph edges**.
- **Inline fields** (Dataview): `key:: value`, `[key:: value]`, `(key:: value)`.
- **Forward references** to not-yet-created notes are allowed and resolved later — capture never blocks on link integrity.
- **Chunking:** header-aware split → ~512-token recursive sub-chunks (~15% overlap; aligns with bge-small's 512-token cap), each prefixed `Title: … | Section: … |` as a semantic anchor, with provenance back to note+heading for jump-to-source.
- **Non-lossy law:** distillation only **adds** a derived note linking back to its source; it never rewrites/deletes the source.

## 7. DuckDB index schema (rebuildable cache)

Reference schema (after `INSTALL/LOAD vss; INSTALL/LOAD fts;`):

- `notes(note_id PK, permalink, path, title, type, tags[], created, importance, last_accessed, access_count, content_hash, mtime)`
- `chunks(chunk_id PK, note_id FK, heading_path, ordinal, text, anchor_prefix)`
- `chunk_embeddings(chunk_id FK, model_id, dim, emb_local FLOAT[384], emb_cloud FLOAT[N] NULL)` — per-model columns so a cloud upgrade is an **additive re-index**, not a migration.
- `links(src_note_id, dst_note_id NULL, dst_permalink, edge_type)` — wikilink/relation graph; unresolved forward-refs keep `dst_note_id NULL` until target exists.
- `meta(key, value)` — records embedding `model_id+version+dim`; index **auto-rebuilds on mismatch** (guards even same-dim semantic mismatch, e.g. MiniLM↔bge both 384).
- HNSW: `CREATE INDEX ON chunk_embeddings USING HNSW (emb_local) WITH (metric='cosine')`; FTS: `PRAGMA create_fts_index('chunks','chunk_id','text', overwrite=1)` wired into every ingest. `PRAGMA hnsw_compact_index` in the maintenance/rebuild cycle to clear delete tombstones.

## 8. Retrieval

Hybrid in (ideally) one SQL statement:
- BM25 CTE (`match_bm25(chunk_id, :q, k:=1.2, b:=0.75)`) + vector CTE (`array_cosine_distance(emb_local, :qvec)`, HNSW) → **RRF fuse** (`1/(k+rank)`, k=60) → **graph-boost** (×~1.2 for notes connected via `links`) → recency/importance scaling columns.
- **Progressive disclosure:** `recall` returns a compact `id + snippet` index first (~50–100 tokens); the agent hydrates full notes on a second call (token efficiency).
- **Optional cross-encoder rerank** (quality profile, bounded top-20) — **off by default** until validated on code/task transcripts (domain-shift risk).
- Degradation ladder per §4.

## 9. Capture / ingest (robustness backbone)

- **Backbone:** out-of-band reading of harness transcripts (`~/.claude/projects/**/*.jsonl`, append-only ⇒ safe to read with no locking) via a **per-harness locator** with graceful "no transcripts found" (paths differ across Codex/Cursor/Gemini and break when project dirs move). **Not** live hooks.
- **Plus:** agent-driven `remember` MCP tool for curated, high-value memories.
- **Pipeline (mandatory order):** locate → **redact secrets** (non-negotiable; we write plaintext) → SHA-256 dedup → **importance gate** (every prior project flooded the vault without one) → **distill** → write derived markdown (non-lossy).
- **Distill runtime:** default = agent-loop (`cairn reflect`, executed by the live agent — zero extra keys, agent-grade quality); optional = background local/cloud LLM for hands-off operation.
- **Triggers:** on-demand, optional session-end hook (accelerator only), or `cairn sweep` (cron).

## 10. Embedding tiers

One rebuildable index; upgrades are additive re-indexes.
- **Tier 1 (default, zero-config, offline):** `fastembed` (ONNX, no PyTorch) running **bge-small-en-v1.5** (384-d, ~32 MB). Asymmetric query/passage prefixes implemented consistently.
- **Tier 2 (opt-in, local-heavy):** Ollama (`nomic-embed-text` 768-d Matryoshka, or `mxbai-embed-large` 1024-d) — separate server, never default.
- **Tier 3 (opt-in, cloud):** OpenAI / Voyage / Cohere for max recall.
- **Reranker:** `sentence-transformers` CrossEncoder (`ms-marco-MiniLM-L-6-v2` or `bge-reranker-v2-m3`) — off by default.
- **First-run UX:** pre-warn on model download size; pre-cacheable for offline.

## 11. Security

- **Secret/credential redaction before every write** (regex + high-entropy heuristics) — the single biggest exposure of a file-first model; mandatory and explicitly tested.
- **MCP binds localhost-only**; **path-traversal guards** confine all note read/write/move to the vault root.
- **Index `.duckdb` on local disk**, never inside a synced/NAS vault folder.
- **No telemetry.** Document the trust boundary (vault is plaintext the user owns; encryption-at-rest is the user's choice).

## 12. Tech stack

Python 3.12+. `duckdb` (+`vss`,`fts`) · `fastembed` (default embeddings) / optional `sentence-transformers`, Ollama, cloud SDKs · `mcp`/FastMCP · `typer` CLI · `python-frontmatter` + `markdown-it-py`. Distributed via `uv`/`uvx`/`pipx`; MCP launched via `uvx agentcairn` (matches the ecosystem norm). PyPI/GitHub package `agentcairn`; CLI binary `cairn`. A future Obsidian plugin (v2) is a separate TypeScript package sharing the vault + `.duckdb` file (DuckDB's file format is cross-language).

## 13. Testing & validation

- **Unit (heaviest on `cairn.vault`):** round-trip parse↔write preserving frontmatter order, forward-refs, link-rewrite-on-move; reconcile mtime/hash diff; RRF math; degradation ladder; **redaction** (golden fake-secret corpus).
- **Integration:** end-to-end ingest→index→recall; **rebuildability invariant** (`reindex` from scratch ≡ incremental result); concurrency (READ_ONLY MCP + writer CLI; "Could not set lock" handling).
- **Retrieval quality:** reproducible **LongMemEval-S *and* LoCoMo** harnesses with committed scripts + commit hashes; validate the local baseline on a **QA-style metric (not just R@5)**; publish ranges with an explicit apples-to-oranges caveat — **no single headline number**.

## 14. Key risks & open questions

- **Large-vault HNSW rebuild / cold-start latency** (daemonless) — *unvalidated*; benchmark to set in-memory-vs-persisted threshold.
- **`cairn.vault` writer/parser is the hardest part** — edit/move/link-rewrite + frontmatter-order + forward-ref preservation; underestimating it yields corruption/broken graphs.
- **Local bge-small < frontier cloud** on hard multi-hop/temporal — frame "competitive" as "hybrid+rerank closes most of the gap," validate on a QA metric.
- **Reranker domain shift** on code/technical content — validate before default-on.
- **Redaction completeness** — incomplete redaction leaks secrets into a user-readable vault (worse than managed-DB competitors); treat as a first-class, continuously-tested concern.
- **DuckDB-WASM has no VSS** (issue #1931) — the deferred v2 Obsidian plugin gets BM25+graph only in-browser; do **not** promise in-browser semantic search.
- **Differentiation window is real but not permanent** — Mem0 (Apr-2026 algorithm) closed much of the lossy gap; Letta MemFS git-tracking encroaches on file-transparency. Ship the wedge crisply and soon.

## 15. v1 scope (YAGNI) + roadmap

**v1 — core loop + the wedge, demonstrably:**
1. `cairn.vault` parser/writer (basic-memory contract + Dataview inline fields).
2. `cairn.index` (DuckDB, reconcile-on-spawn, in-memory HNSW + FTS, disposable).
3. `cairn.embed` Tier-1 (FastEmbed bge-small) behind a pluggable interface.
4. `cairn.search` hybrid (BM25 + vector + graph-boost + recency/importance, RRF, degradation ladder, progressive disclosure).
5. `cairn.ingest` Claude Code transcript backbone + `remember` → redact → dedup → importance-gate → agent-loop distill (non-lossy).
6. `cairn.mcp` (`remember/recall/search/build_context/recent`) + `cairn` CLI (`reindex/sweep/doctor`).
7. Security baseline + LongMemEval-S benchmark harness.
8. **Demoable wedge:** hand-edit a fact in Obsidian → `cairn reindex` honors it; nuke the index → rebuilds with zero loss.

**Deferred (hooks reserved now):** bi-temporal validity (v1.1) · reranker default-on (v1.1) · cloud/Ollama tiers (v1.1) · richer lifecycle skills + 4-tier consolidation (v1.2) · Obsidian plugin surface (v2) · MotherDuck sync (v2) · LLM entity extraction (v2) · multi-harness locators (incremental).

## 16. Prior art borrowed

basic-memory (markdown contract, `memory://`, build_context, forward-ref resolution, lifecycle-as-skills) · ssp.sh Obsidian-RAG-DuckDB (notes/chunks/embeddings/links schema, graph-boost, semantic-anchor chunk prefix) · claude-mem (progressive-disclosure retrieval — architecture rejected) · DuckDB hybrid pattern (BM25 + cosine CTEs fused with RRF) · reference MCP "memory" server + Mem0/OpenMemory (remember/recall + CRUD tool idiom).

---

*Research grounding (competitor teardown, DuckDB VSS/FTS/concurrency/WASM limits, local-embedding benchmarks, prior art) was conducted 2026-06-08; full findings + sources retained in the session record.*
