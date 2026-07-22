# Jieumchat: Backend & RAG System Architecture Interview Guide

This guide describes the backend design, pipeline mechanics, and engineering decisions of **Jieumchat** to help you explain the architecture to an interviewer.

---

## 1. System Overview & Purpose

Jieumchat is an enterprise **Retrieval-Augmented Generation (RAG)** platform designed to answer natural language questions using knowledge synced from Atlassian instances (**Jira** issues and **Confluence** pages). It solves three primary challenges for enterprise setups:
1.  **Strict Security (Access Control Lists)**: Restricting document visibility in vector space based on the querying user's active permissions.
2.  **Rate-Limited Ingestion**: Synchronizing millions of pages without triggering API rate limits (150 requests/min).
3.  **Context-Aware Retrieval**: Merging adjacent document chunks and applying recency-biased cross-encoder reranking to prevent stale answers.

---

## 2. Microservices Overview

The backend is composed of two primary FastAPI services running in Docker/Kubernetes:

| Service | Port | Primary Responsibility | Key Technologies |
| :--- | :--- | :--- | :--- |
| **Data Collection** | `8034` | Incremental syncing, document parsing, chunking, embedding, vector store insertions. | `FastAPI`, `APScheduler`, `PostgreSQL`, `Weaviate`, `tiktoken` |
| **Query Handler** | `8031` | User query parsing, tool-agent search, ACL injection, chunk merging, reranking, LLM response generation. | `FastAPI`, `Weaviate Async`, `bge-m3-reranker`, `Qwen 3`, `Langfuse` |

---

## 3. Data Collection Service (Ingestion Pipeline)

> [!NOTE]
> For a step-by-step walkthrough of each stage, queue, subprocess boundary, and lock, see the [Data Ingestion Pipeline Deep Dive](file:///home/ubuntu/Development/pr_backend/jieum-chat/docs/data_ingestion_deep_dive.md).

The collection service follows **Clean Architecture & DDD (Domain-Driven Design)** principles:
*   **Domain (`/domain`)**: Core entities (frozen dataclasses like `DataSource`, `Pat`, `StorageItem`) and structural interfaces (`typing.Protocol` repos). It contains no I/O or framework dependencies.
*   **Application (`/application`)**: Use cases orchestrating sync jobs (`discovery_job`, `permission_job`, `collection_job`).
*   **Infrastructure (`/infrastructure`)**: Database-specific access (PostgreSQL/Weaviate drivers), multi-process workers, and REST clients.


```
[Atlassian Cloud/MX] 
       │ (Phase 1: Detect IDs/Versions)
       ▼
 [PostgreSQL Metadata] ──(Set-Diff)──> [Identify Changed IDs]
                                              │
                                              │ (Phase 2: Dispatch Page Chunks)
                                              ▼
                                     [Download Workers]
                                              │
                                              │ (Phase 3: Embed & Store)
                                              ▼
                                     [Weaviate Vector DB]
```

### High-Throughput Syncing Strategy (Two-Phase Fetch)
Atlassian Cloud rate limits are strict. Jieumchat mitigates this with a two-phase ingestion cycle:
1.  **Phase 1 (Detect Changes)**: The service pulls lightweight lists of IDs and modification epochs (versions) from the Atlassian APIs. It performs a **set-difference** in memory against local PostgreSQL records to isolate new or modified documents.
2.  **Phase 2 (Dispatch & Parallel Download)**: It partitions the modified document IDs into `CollectionJob` payloads and dispatches them across a pool of parallel downloader workers (`COLLECTION_WORKER_COUNT = 16`).
3.  **Phase 3 (Embed & Store)**: The workers split document text into chunks (`CHUNK_SIZE = 512` with `0.3` overlap) and send them to the embedding model (e.g., `embeddinggemma-300m`). Chunks are loaded in batches (`WEAVIATE_BATCH_SIZE = 1000`) to maximize write throughput.

---

## 4. Query Handler Service (RAG Pipeline)

The query handler is designed as a multi-stage search agent:

```
[User Query] ──> [Query Rewriting] 
                     │
                     ▼
             [Hybrid Search] ──> (Acl Injection Filter) ──> [Weaviate DB]
                     │
                     ▼
             [Chunk Merging] (Deduplicate overlapping windows)
                     │
                     ▼
             [Rerank + Recency] (Multiplicative Reranker ^ α * Recency ^ β)
                     │
                     ▼
             [Context Compaction] (Soft Token Limit check)
                     │
                     ▼
             [LLM Generation] ──> (Server-Sent Events Stream)
```

### Key RAG Pipeline Operations
*   **Query Rewriting**: The LLM analyzes the chat history and the current prompt to rewrite the query into clear keyword and semantic search terms.
*   **Permission-Aware Search (ACL Injection)**: Before executing the search in Weaviate, the system queries PostgreSQL to retrieve the list of data source UUIDs the user has permissions to see (`data_source_access_grants`). This permission list is dynamically injected as a filter parameter in the vector store query.
*   **Adjacent Chunk Merging**: To avoid sending fragmented text blocks to the LLM, search chunks that originate from the same document and lie within close proximity are merged together.
*   **Recency-Biased Reranking**: Documents in a knowledge base decay over time. To ensure users get up-to-date answers, a multiplicative fusion algorithm is used:
    $$\text{Final Score} = (\text{Reranker Score})^{\alpha} \times (\text{Recency Score})^{\beta}$$
    *   $\alpha$ scales the semantic cross-encoder score (`bge-m3-reranker`).
    *   $\beta$ penalizes older documents based on their update timestamp.
*   **Context Compaction**: The system monitors the token buffer using a local `tiktoken` configuration. If historical context approaches the limit, the system summarizes historical turns to stay within the model's performance context window.
*   **Server-Sent Events (SSE)**: The final answer is streamed back via SSE with custom message envelopes (e.g., `session`, `message`, `end`) containing source citations.

---

## 5. Architectural Trade-offs & Engineering Hardening

Prepare to discuss these technical challenges during your interview:

### Rate-Limiting & Backoff
*   **Problem**: Exceeding the 150 request/minute Atlassian SA account limit blocks the ingestion pipeline.
*   **Solution**: We built a custom throttle decorator `with_async_retry` that intercepts `HTTP 429` statuses, reads Atlassian's `Retry-After` headers, and triggers exponential backoff. In the detect phase, we restrict page fetch concurrency to stay within limits.

### PostgreSQL/PgBouncer Transaction Pooling
*   **Problem**: PgBouncer transaction pooling closes connections in a way that breaks standard PostgreSQL prepared statements.
*   **Solution**: We disabled the default prepared statement caching inside `asyncpg` by configuring `statement_cache_size=0`. This ensures all queries run as raw statements and do not return phantom errors.

### Observability with Langfuse
*   **Problem**: In an agentic tool loop, it's hard to identify which step (retrieval, reranking, or LLM generation) is bottlenecking latency.
*   **Solution**: Integrated Langfuse middleware which automatically profiles the prompt rewriting, vector search retrieval payload size, reranker inference overhead, and final token usage.

---

## 5. Job Scheduling & Concurrency Protection

The Data Collection service automates ingestion and administrative syncs using a hybrid scheduling and manual trigger system:

### Core Scheduler (`app.py`)
*   **Technology**: `AsyncIOScheduler` from `APScheduler` is instantiated and managed directly within the FastAPI **lifespan** startup/shutdown hooks.
*   **Background Jobs**:
    1.  **`auto_discovery`**: Scans for new Jira projects and Confluence spaces at configured intervals (`DISCOVERY_SYNC_INTERVAL` default: 6 hours).
    2.  **`permission_sync`**: Synchronizes user permission changes and credential bindings (`PERMISSION_SYNC_INTERVAL` default: 6 hours).
    3.  **`per_pat_collection`**: Orchestrates the incremental data collection pipeline (`COLLECTION_SCHEDULE_INTERVAL` default: 1 minute).
    4.  **`garbage_collection`**: A cron trigger running nightly at 20:00 UTC (05:00 KST) to clean up orphaned data sources and unlinked vector store chunks.

### Unified Trigger System & Concurrency Protection (`scheduler.py`)
*   **The Problem**: If a manual sync is triggered via HTTP while a scheduled background job is already running, they could race, competing for database locks and Atlassian API quotas.
*   **The Solution**: Both background scheduled jobs and REST endpoints (`POST /scheduler/trigger-*`) call a unified `run_once()` helper:
    *   **In-Memory Lock registry**: `run_once` registers active job IDs in a `_running_events` dictionary backed by `asyncio.Event` primitives.
    *   **Concurrency Guard (`max_instances=1`)**: If a job is already executing:
        *   The scheduler silently skips the redundant run.
        *   Manual trigger endpoints can cleanly `await` the completion event of the active run before returning, preventing duplicate executions.

---

## 6. Interview Talking Points (Shorthand Cheat Sheet)

*   **Ingestion Pipeline**: "I designed a Clean Architecture data sync pipeline utilizing a two-phase fetch pattern (change detection followed by parallel collection) to stay under Atlassian's strict rate limits."
*   **Vector Database Access**: "I implemented permission-based retrieval (ACL filters) directly inside the Weaviate vector search query to prevent data leakage between user scopes."
*   **Search Relevance**: "We didn't just use standard vector retrieval. We optimized relevance by passing chunks through a hybrid search (vector + keyword) followed by a cross-encoder reranker that factors in document recency."
*   **System Performance**: "To support PgBouncer transaction pooling in Kubernetes, I disabled asyncpg's statement cache to avoid transaction mismatch errors."
*   **Background Scheduling**: "I integrated APScheduler within the FastAPI lifecycle and built a unified `run_once` trigger wrapper utilizing `asyncio.Event` states. This enforces strict concurrency limits (`max_instances=1`) across both cron jobs and manual HTTP sync requests, preventing race conditions and API rate limit exhaustion."
*   **Real-time Output**: "We stream RAG agent output to the client using Server-Sent Events (SSE) combined with real-time citation links mapping back to original Confluence spaces and Jira tickets."
