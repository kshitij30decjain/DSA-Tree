# Jieumchat Configuration Analysis & Interview Prep Spec

This document provides a production-grade systems analysis of the Jieumchat environment configurations (`.env`) to support technical interview preparation for engineering roles.

---

## 1. High-Level Overview

The configuration file (`.env`) manages the runtime environment, credentials, scaling parameters, and operational bounds of the Jieumchat RAG system. It serves as the single source of truth for:
*   **AI Models & Reranking Settings**: Endpoint URLs and model names for Qwen3 LLM, Gemma embedding, and the BGE-M3 cross-encoder reranker, including context constraints and temperature controls.
*   **Database Infrastructure**: Port bindings and connection pooling guidelines for both PostgreSQL (mediated by PgBouncer) and the Weaviate vector database.
*   **Data Collection Pipeline**: Concurrency limits (workers, batch sizes) and execution frequencies (scheduler intervals) governing the Atlassian (Jira/Confluence) ingestion loop.
*   **SaaS/On-Premise Access Credentials**: Personal Access Tokens (PATs) that authorize system crawlers to read Jira and Confluence spaces across cloud and datacenter instances.

---

## 2. Detailed Parameter breakdown

### 2.1. Ingestion Concurrency & Tuning
```env
EMBEDDING_BATCH_SIZE=64
EMBEDDING_WORKER_COUNT=8
WEAVIATE_WRITER_COUNT=8
DISPATCHER_DETECT_CONCURRENCY=4
```
*   `EMBEDDING_BATCH_SIZE`: Groups text chunks into batches of 64 before making HTTP requests to the Nomic embedding API, minimizing network roundtrips.
*   `EMBEDDING_WORKER_COUNT`: Spawns 8 parallel worker threads within the dedicated `StorageProcess` to parse, clean, and request embeddings for document chunks.
*   `WEAVIATE_WRITER_COUNT`: Launches 8 parallel writer threads executing gRPC batch writes to Weaviate to ensure high write throughput.
*   `DISPATCHER_DETECT_CONCURRENCY`: Resolves differences (diffs) between Atlassian IDs and database records using 4 concurrent workers during change detection.

### 2.2. Background Job Scheduling (Time in Seconds)
```env
DISCOVERY_SYNC_INTERVAL=21600   # 6 Hours
PERMISSION_SYNC_INTERVAL=21600  # 6 Hours
COLLECTION_SCHEDULE_INTERVAL=60 # 1 Minute
```
*   `DISCOVERY_SYNC_INTERVAL`: Runs a discovery job every 6 hours to scan Atlassian instances for new projects or Confluence spaces, updating `DATA_SOURCE_CONFIG` in PostgreSQL.
*   `PERMISSION_SYNC_INTERVAL`: Synchronizes Confluence user lists and permission mappings every 6 hours to keep Access Control Lists (ACLs) up-to-date.
*   `COLLECTION_SCHEDULE_INTERVAL`: Checks for document changes (incremental updates) every 60 seconds.

### 2.3. Reranker & Scoring Formula
```env
RERANKER_ALPHA="2.0"
RECENCY_BETA="0.6"
RECENCY_MIN_RANGE_MONTHS="3"
```
*   `RERANKER_ALPHA` & `RECENCY_BETA`: Powers the hybrid scoring formula combining semantic relevance and document freshness:
    $$\text{Final Score} = \text{RerankerScore}^{\alpha} \times \text{RecencyScore}^{\beta}$$
    $\alpha$ (2.0) places high weight on semantic alignment, while $\beta$ (0.6) boosts documents modified within the last 3 months (`RECENCY_MIN_RANGE_MONTHS`).

### 2.4. Context & LLM Parameters
```env
CONTEXT_WINDOW_LIMIT="131072"
CONTEXT_COMPACTION_SOFT_FILL_RATIO=0.85
LLM_TEMPERATURE="0.6"
```
*   `CONTEXT_WINDOW_LIMIT`: Declares a 128k context token limit for the Qwen3 model.
*   `CONTEXT_COMPACTION_SOFT_FILL_RATIO`: Triggers historical prompt context compaction when the conversation exceeds 85% (approx. 111,000 tokens) of the context window.
*   `LLM_TEMPERATURE`: Controls randomness in generation. Set to `0.6` to balance agent reasoning stability and text fluidness.

### 2.5. PostgreSQL Connection Pool
```env
POSTGRES_PORT="54322"
POSTGRES_POOL_MIN="2"
POSTGRES_POOL_MAX="20"
```
*   `POSTGRES_PORT`: Connects to `54322` which routing traffic through **PgBouncer** rather than directly to PostgreSQL (port `5432`).
*   `POSTGRES_POOL_MIN` & `POSTGRES_POOL_MAX`: Establishes the pool range (2 to 20 connections) per service replica.

---

## 3. Production Risks & System Impacts of Misconfiguration

| Misconfigured Parameter | Value Deviancy | System Impact / Production Outage Vector |
| :--- | :--- | :--- |
| `EMBEDDING_WORKER_COUNT` | Set too high (e.g. `64`) | Oversaturates local CPU cores and Nomic API limits, resulting in `429 Too Many Requests` or connection dropouts. |
| `POSTGRES_POOL_MAX` | Exceeds PgBouncer slots | Causes database connection exhaustion: `FATAL: remaining connection slots are reserved...` which crashes web requests. |
| `COLLECTION_SCHEDULE_INTERVAL` | Set too low (e.g. `5`) | Causes overlapping scheduler jobs, piling up CPU-bound change checks and rate-limiting the SaaS connector. |
| `RERANKER_ALPHA` | Set too high | Suppresses newer documents in search results in favor of old documents with minor keyword matches. |
| `CONTEXT_WINDOW_LIMIT` | Exceeds model limits | Triggers raw `400 Bad Request` or context overflow errors from the LLM provider, breaking agent loops. |

---

## 4. Key Systems Interview Questions & Model Answers

### Question 1: Connection Pooling & Proxy Architectures
> **Interviewer:** *"I see you configured both `POSTGRES_PORT=54322` and connection pooling (`POOL_MAX=20`). Why is this configured to use port 54322 instead of the standard 5432, and how does that influence connection pooling when scaling your FastAPI services?"*

**Ideal Candidate Answer:**
> "Port `54322` points to **PgBouncer**, which is deployed in front of our main PostgreSQL database. In a serverless or containerized environment (like Kubernetes), scaling our Query and Collection services horizontally means each container replica creates its own connection pool (up to 20 connections in this configuration). 
> 
> If we connected directly to Postgres on `5432` without PgBouncer, 10 container replicas could easily exceed Postgres's physical connection limits (typically set to 100 or 150). PgBouncer acts as a transaction-level proxy, letting our application containers quickly open and close connections locally while reusing a small, warm pool of multiplexed connections to the actual database. This protects Postgres from connection exhaustion during traffic spikes."

---

### Question 2: Ingestion Concurrency & Rate Limit Recovery
> **Interviewer:** *"In your ingestion settings, how do you handle rate-limit recovery when your 8 concurrent `EmbeddingWorkers` and 8 `WeaviateWriters` get throttled by external model endpoints or Weaviate batch constraints?"*

**Ideal Candidate Answer:**
> "To manage this, our infrastructure workers do not operate with a naive write loop. First, the 8 `EmbeddingWorkers` process documents in batches of 64 (`EMBEDDING_BATCH_SIZE`) to minimize round-trip requests.
> 
> Second, both the embedding fetches and Weaviate gRPC batch calls are wrapped in an asynchronous retry wrapper (`with_async_retry`) implementing **Exponential Backoff with Jitter**. If Weaviate returns a write conflict or Nomic throws a rate limit error, the worker backs off exponentially ($delay = base \times 2^{attempt} + random\_jitter$), spreading out retry requests. If a worker fails consistently after 10 attempts, the storage job is flagged as `failed` in PostgreSQL and written to the `index_error` log, allowing the pipeline to continue indexing other documents without crashing the main `StorageProcess` loop."
