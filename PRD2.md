# Time‑Series Pattern Retrieval (RAG) System — PRD

Version: 1.0  
Owner: TBD  
Status: Draft

> Disclaimer: This system is for research and educational purposes only. It is not a trading system, does not provide financial advice, and should not be used to make investment decisions.

## 1) Overview
A system that applies Retrieval Augmented Generation (RAG) concepts to time‑series data. It ingests raw financial candlestick data, transforms fixed‑length windows of OHLC(V) bars into numerical embeddings, stores them in a vector database, and enables fast similarity search to retrieve historical windows most similar to a query window (e.g., the most recent bars for a symbol). A lightweight UI and API expose search, visualization, and configuration.

Primary objective: help users explore historically similar price patterns at varying time scales to inform analysis and research. This is not an automated strategy or signal generator.

## 2) Problem Statement
Analysts often want to quickly find historical segments that “look like” recent market behavior. Manual browsing is slow and biased. We need a reproducible way to represent price‑action patterns numerically and retrieve nearest neighbors at scale, enabling fast, explainable pattern discovery across large histories and symbols.

## 3) Goals
- Provide an end‑to‑end workflow for:
  - Ingesting raw OHLC(V) time‑series for configurable symbols and intervals
  - Generating fixed‑length windows with configurable stride
  - Normalizing and embedding those windows
  - Indexing embeddings for fast similarity search
  - Returning ranked, visualized matches with metadata
- Make configuration (window size, stride, metrics, filters) easily tunable.
- Ship a minimal, intuitive UI and API suitable for local or small‑team use.
- Ensure results are fast, reproducible, and interpretable for research.

## 4) Non‑Goals
- No automated trading, order routing, or brokerage integrations.
- No price prediction, alpha claims, or guaranteed performance.
- No long‑horizon backtesting harness beyond retrieval QA.
- No proprietary market data distribution.

## 5) Target Users and Use Cases
- Quant/ML researchers: quickly retrieve historical analogs to study regimes and patterns.
- Discretionary analysts: browse similar windows to contextualize current price action.
- Educators: demonstrate time‑series retrieval concepts and normalization effects.

Key use cases:
- “Find windows similar to the last N bars of symbol X.”
- “Find windows similar to a specific historical segment.”
- “Explore patterns across multiple symbols/markets and different time frames.”

## 6) Success Metrics (MVP)
- Median query latency for top‑20 results: ≤ 500 ms on 250k windows (single node).
- Embedding build throughput: ≥ 5k windows/minute on commodity hardware.
- Relevance sanity: ≥ 80% of top‑10 results judged visually plausible by reviewers.
- Zero critical errors over 1 week of continuous ingestion + query in dev.

## 7) Functional Requirements
### FR1 — Data Ingestion
- Fetch OHLC(V) bars for configured symbols, intervals (e.g., daily, hourly), and date ranges.
- Pluggable provider adapters (e.g., public datasets/APIs); at least one adapter included.
- Persist raw bars with schema and indexing suitable for time‑range queries.

### FR2 — Windowing
- Generate fixed‑length windows (e.g., 5, 10, 20, 100 bars) with configurable stride (e.g., 1–10 bars).
- Exclude incomplete windows (missing bars) or fill per policy (skip vs. forward‑fill).
- Store window metadata: symbol, start/end timestamps, window_size, stride, interval.

### FR3 — Normalization
- Apply per‑window min‑max scaling on OHLC (optionally volume) to emphasize shape over scale.
- Configurable normalization strategies: per‑window min‑max (default), z‑score, log‑returns, or none.
- Persist normalization parameters for reproducibility.

### FR4 — Embedding Generation
- Transform normalized windows into dense vectors (baseline: flattened OHLC[+V] sequence).
- Support an embedding interface with pluggable models (baseline numeric shape; future deep encoders).
- Record embedding version, model name, vector dimension, and creation timestamp.

### FR5 — Vector Indexing & Search
- Store embeddings in a vector database supporting cosine and L2 distance; configurable metric.
- Provide top‑K nearest neighbor search with filters:
  - by symbol include/exclude
  - by date range
  - exclude the query’s overlapping time neighborhood to avoid trivial matches
- Return matches with: similarity score, symbol, start/end timestamps, preview sparkline, and linkable context.

### FR6 — API
- REST (or GraphQL) endpoints:
  - POST /ingest/run: trigger ingestion for symbols/date range
  - POST /embeddings/build: build (or rebuild) embeddings for a symbol/date range
  - POST /search/similar: body includes query window spec or “use most recent N bars”
  - GET /symbols, GET /healthz, GET /stats
- Pagination and limits on search responses.

### FR7 — UI
- View: “Similar to recent” — for a selected symbol and N bars.
- View: “Similar to selected range” — user chooses historical start/end.
- Controls: symbol, interval, window size, stride (read‑only if precomputed), top‑K, filters.
- Results list with inline mini‑charts, score, and quick compare modal (overlay or side‑by‑side).
- Prominent non‑advice disclaimer.

### FR8 — Operations & Scheduling
- CLI commands to run ingestion and embedding jobs.
- Idempotent processing per (symbol, interval, date range, embedding_version).
- Optional periodic jobs (e.g., nightly) for new data updates.

### FR9 — Observability & QA
- Metrics: number of bars/windows/embeddings, search latency, job durations, failures.
- Structured logs for ingestion, windowing, embedding, and search.
- Sanity check suite: synthetic patterns (e.g., V‑shape, channel) retrieve similar shapes.

## 8) Non‑Functional Requirements
- Performance: as per MVP metrics; degrade gracefully with dataset size.
- Scalability: linear scaling via sharding by symbol/date; pluggable vector store that supports ANN.
- Reliability: job retries, partial resume, and integrity checks.
- Security: API keys for write operations; read endpoints rate‑limited.
- Portability: local‑first deployment; optional containerization.
- Reproducibility: versioned embeddings and deterministic normalization.

## 9) Architecture (High‑Level)
- Data Store (raw bars): time‑series‑optimized relational store (e.g., time‑partitioned Postgres) with indexes on (symbol, timestamp).
- Processing: batch jobs for windowing, normalization, and embedding creation.
- Vector Store: pluggable backend. Baseline options may include a local vector DB or an extension to a relational DB for vectors.
- API Service: stateless app serving search and job orchestration.
- UI: lightweight web app consuming the API.

Data flow:
1) Ingest raw bars → 2) Window & normalize → 3) Embed & index → 4) Search via API → 5) UI visualization.

## 10) Data Model (Baseline)
### Raw Bars Table
- id (uuid)
- symbol (text)
- ts (timestamptz)
- open, high, low, close (numeric)
- volume (numeric, optional)
- interval (text)
- source (text)
- PRIMARY KEY (symbol, ts)

Indexes: (symbol, ts), time partitioning per interval.

### Window Embeddings Table
- id (uuid)
- symbol (text)
- start_ts (timestamptz)
- end_ts (timestamptz)
- interval (text)
- window_size (int)
- stride (int)
- normalization (jsonb: method + params)
- embedding_version (text)
- vector (vector / array<float>)
- metric (text: cosine|l2)
- created_at (timestamptz)

Indexes: (symbol, start_ts), vector index (per backend), (embedding_version, metric).

## 11) Algorithms & Configuration
- Windowing: sliding windows over [start, end], step = stride; no look‑ahead leakage.
- Normalization: default per‑window min‑max on OHLC; optional include volume.
- Embedding (baseline): concatenate normalized OHLC(V) into a vector of length window_size × features.
- Distance/Similarity: cosine similarity (default) with monotonic mapping to [0, 1].
- Filtering: remove windows within ±W bars of the query range to prevent trivial overlaps; optional cross‑symbol filters.

## 12) Quality and Evaluation
- Visual plausibility review on curated test queries (weekly).
- Synthetic retrieval tests: known shapes return >80% shape‑consistent neighbors.
- Stability tests: scores consistent across rebuilds with same version.
- Score calibration: ensure scores are bounded, consistent with distance, and monotonically interpretable.

## 13) Risks & Mitigations
- High symbol correlation → add symbol/sector diversity filters and date exclusion windows.
- Poor baseline embeddings → provide pluggable encoders; evaluate deep encoders in vNext.
- Data gaps/holidays → validation and gap policies; skip or fill per configuration.
- Cost/scale constraints → support compact vectors and ANN indexes; shard by symbol.

## 14) Milestones
- M1: Ingestion + raw storage (bars) with one provider adapter; CLI + API healthz.
- M2: Windowing + normalization + embedding pipeline; metadata versioning.
- M3: Vector indexing + top‑K search; filters; score calibration.
- M4: UI (similar to recent + select range); overlay compare; disclaimers.
- M5: Observability; synthetic tests; relevance review loop.
- M6: Packaging for local deployment; documentation and examples.

## 15) Open Questions
- Default window sizes and stride per interval (daily vs intraday)?
- Include volume by default in embeddings?
- Normalization choice defaults and user overrides?
- Minimum dataset size before enabling UI search?
- How to handle corporate actions (splits/dividends) if provider doesn’t adjust?

## 16) Future Enhancements
- Learned embeddings: autoencoders or time‑series foundation models.
- Multi‑resolution retrieval: hierarchical windows (e.g., 5/20/60 bars combined).
- Cross‑market retrieval (equities, FX, crypto, futures) with configurable adapters.
- Temporal context constraints (e.g., match similar regimes, volatility buckets).
- Incremental/online updates for low‑latency intraday use.
- Export match sets for downstream modeling/annotation.

## 17) Compliance & Ethics
- Prominent non‑advice disclaimer across UI and docs.
- Respect data provider terms; do not redistribute restricted data.
- Provide clear methodology descriptions to reduce misuse and over‑interpretation.

## 18) Acceptance Criteria (MVP)
- End‑to‑end flow succeeds on at least 50 symbols × 5 years daily bars.
- Top‑20 query for a recent window returns within ≤ 500 ms median on a single node.
- UI shows at least 10 visually plausible matches with correct timestamps and symbols.
- Scores displayed ∈ [0,1], monotonic with chosen distance; no obvious scaling bugs.
- Reproducible: rerunning embedding build with same version yields identical vectors and results.

---

Appendix: Glossary
- Window: fixed‑length contiguous sequence of OHLC(V) bars.
- Stride: step size when sliding window across the time series.
- Embedding: numeric vector representing a window suitable for similarity search.
- ANN: approximate nearest neighbor indexing for fast vector retrieval.

