# Jieumchat: Data Ingestion Pipeline Deep Dive

This document details the exact, step-by-step technical lifecycle of the **Jieumchat Data Ingestion (Collection) Service**. It outlines the data structures, concurrency controls, thread/process boundaries, caching mechanisms, and error recovery flows.

---

## Ingestion Overview

The ingestion service runs on port `8034` (entry point `collection_main.py`). It is responsible for incrementally synchronizing Confluence spaces and Jira projects into the **Weaviate** Vector Database while storing state and metadata in **PostgreSQL**.

The ingestion lifecycle is divided into **4 distinct stages**:
1.  **Stage 1: DETECT** (Main Process) — Change detection using set differences.
2.  **Stage 2: DISPATCH** (Main Process) — Sharding document IDs into collection chunks.
3.  **Stage 3: COLLECT** (Main Process Workers) — Fetching full content from Atlassian APIs.
4.  **Stage 4: STORE** (Separate Subprocess Workers) — Tokenizing, embedding, and batch writing.

---

## Step-by-Step Execution Lifecycle

### Step 1: Triggering and Lifecycle
*   **Initialization**: During FastAPI startup, the `AsyncIOScheduler` (from `APScheduler`) is initialized and started inside the application `lifespan` hook in `data_collection/app.py`.
*   **The Ingestion Trigger**: The `per_pat_collection` interval job is scheduled to run every 60 seconds (configured via `COLLECTION_SCHEDULE_INTERVAL`).
*   **Concurrency Lock**: To prevent scheduled runs and manual triggers (`POST /scheduler/trigger-per-pat-collection`) from overlapping, all invocations pass through `scheduler_router_mod.run_once()`. This helper maintains an in-memory lock map `_running_events` backed by `asyncio.Event` primitives. If a run is already active, the new trigger is discarded (`max_instances=1`).
*   **Backpressure Check**: Before fetching targets, the dispatcher calls `self._worker_manager.are_queues_empty()`. If any documents are still in the collection or storage queues, the dispatcher skips the iteration to prevent backlog accumulation.

---

### Step 2: The Change Detection Phase (Stage 1: DETECT)
For each approved data source retrieved from PostgreSQL via `DataSourceRepo.get_by_filters(approval_status="approved")`:
1.  **Collector Isolation**: The dispatcher calls `_collector_rows_for(ds_uuid)` to isolate the active users who registered this data source (`principal_type == 'user'` and `grant_source == 'self'`). This ensures only the credentials of data source owners are utilized for fetching, keeping invited/public-only viewers strictly separated.
2.  **Concurrency Control**: The dispatcher processes multiple data sources concurrently, bounded by a semaphore profile (`detect_concurrency` defaults to 4 or 8 depending on the organization).
3.  **Local Metadata Query**: The database versions of previously indexed documents are loaded into an in-memory dictionary `db_versions: dict[doc_id, version]` from PostgreSQL.
4.  **Lightweight API Scan**:
    *   The dispatcher accesses the owner's personal access token (PAT) from `PatRepo` and ensures its `initial_sync_status == "completed"`.
    *   It converts `last_collected_at` into the Atlassian server's timezone (using timezone rules from `common/config.SOURCE_SERVER_TZ` since JQL/CQL date filters are naive and timezone-dependent).
    *   It queries Atlassian's `get_total_count(resource_id, since, token, until)` to see how many items changed.
    *   If `total_count > 0`, it calls `fetch_ids_with_versions(...)` to pull a lightweight list of document IDs and version numbers (avoiding heavy payload fields like body or comments).
5.  **Change Isolation**:
    *   `detect_changes(source_versions, db_versions)` diffs the versions.
    *   It identifies new documents (present in source but not DB) and updated documents (where the source version number is greater than the DB version number).
    *   It returns `all_doc_ids` (the set of changed documents) and `doc_to_users` (mapping each changed document to the set of owner PATs eligible to download it).

---

### Step 3: Sharding & Job Dispatching (Stage 2: DISPATCH)
If changes are detected:
1.  **Eligibility Sharding**: The system shards the list of changed `doc_ids` based on the set of users who have permissions to view them (`doc_to_users`). This ensures that during collection, a worker only requests a document using a token from a user who has active visibility privileges on that document.
2.  **Chunking**: Each permission shard is sorted deterministically and divided into chunks of size `COLLECTION_CHUNK_SIZE` (default: 50).
3.  **Job Enqueueing**:
    *   Each chunk is packaged into a `CollectionJob` dataclass with a unique ID (e.g. `data_source_0/4_shard_0_doc_0-50/200`).
    *   The job is placed onto the dedicated `asyncio.Queue` for that specific `(org, source_type)` pool.
    *   **Rate-Limit Queue Isolation**: Because each combination (e.g. `joyent/jira`, `mx/confluence`) has its own queue and independent worker tasks, a rate-limit block or crash in one source does not block collection tasks in another.

---

### Step 4: Parallel Content Collection (Stage 3: COLLECT)
A pool of async task workers (`CollectionWorker` tasks) runs in the main FastAPI process:
1.  **Job Retrieval**: A worker pulls a `CollectionJob` from its queue.
2.  **Token Selection & Throttling**:
    *   It calls `self._token_cache.select_token()` to pick a valid PAT token from the pool of eligible user IDs.
    *   The `TokenCache` acts as a rate-limit buffer. It keeps track of the remaining requests for each token based on returned `x-ratelimit-remaining` headers. If a token is near its limit, it is deprioritized or temporarily skipped.
3.  **Full Document Download**:
    *   The worker calls `client.fetch_by_ids(resource_id, doc_ids, token)` to download the complete body content, comments, and attachments for the chunk of 50 documents.
    *   **Mismatch Handling**: If some documents are missing from the returned results (due to hard deletions at the source or mid-sync permission changes), it logs a warning listing the missing IDs.
4.  **Handoff to Storage Process**:
    *   The retrieved documents are compiled into `StorageItem` objects.
    *   The worker wraps them in a `StorageJob` dataclass.
    *   To prevent clogging the main event loop with CPU-heavy multiprocessing serialization, the worker delegates enqueueing to an executor thread: `loop.run_in_executor(None, self._storage_mp_queue.put, storage_job)`.

---

### Step 5: CPU-Isolated Storage Subprocess (Stage 4: STORE)
Because chunking, tokenizing (via tiktoken), and embedding are heavily CPU-bound and subject to Python's Global Interpreter Lock (GIL), all storage writes are isolated to a separate OS process (`StorageProcess`):
1.  **Process Spawn**: On initialization, the worker manager starts `StorageProcess` using the `"spawn"` context (ensuring connection handles like gRPC or asyncpg pool locks are not unsafely inherited from a fork).
2.  **Connection Setup**: Upon starting, the subprocess creates its own isolated `async_weaviate_client` and PostgreSQL connection pool.
3.  **Internal Pipeline Queueing**:
    *   A bridge thread executes `stdlib_queue.get()` to pull `StorageJob`s from the `multiprocessing.Queue` and pushes them into an internal `asyncio.Queue` called `embedding_queue`.
    *   The subprocess runs its own asyncio loop directing two distinct worker pools: `EmbeddingWorker`s and `WeaviateWriter`s.

---

### Step 6: Text Processing & Embedding (`EmbeddingWorker`)
A pool of `EmbeddingWorker` tasks (default: 8) pulls from the `embedding_queue`:
1.  **Document-Atomic Chunking**:
    *   Documents are processed one by one.
    *   The text is sent to `make_chunks(text, CHUNK_SIZE=512, OVERLAP_RATIO=0.3)`. It computes overlap boundaries based on the local tokenizer configuration.
    *   **Memory Bounds**: Chunks are grouped into sub-batches. If adding the chunks of the next document would exceed `EMBEDDING_BATCH_SIZE` (default: 64), the worker flushes the current sub-batch to keep the in-memory payload bounded.
2.  **API Embeddings**:
    *   The worker calls the external Embedding API (`get_embeddings_async`) to vectorize the chunk texts.
    *   **Unembeddable Chunk Handling**: If a chunk fails validation (e.g. 400/413/418 error due to illegal encoding or payload size), the API returns a `None` sentinel. The worker safely logs the error and discards the chunk rather than writing a corrupted zero-vector that would break vector searches.
3.  **Data Generation**:
    *   **Chunks**: Generates a deterministic UUID using `uuid.uuid5(uuid.NAMESPACE_URL, ...)` to prevent duplication if the same chunk is processed again.
    *   **Records/Entities**: Extracts and builds properties for metadata records and entity associations.
    *   **Delete Filter**: Builds a `DeleteFilter` object specifying the data source and document IDs in the sub-batch.
4.  **Write Hand-Off**: The fully vectorized sub-batch is wrapped into an `EmbeddedBatch` and pushed to the `weaviate_write_queue`.

---

### Step 7: Vector Store Writing (`WeaviateWriter`)
A pool of `WeaviateWriter` tasks (default: 8) pulls from the `weaviate_write_queue`:
1.  **Identical Check (Optimization)**:
    *   The writer queries Weaviate to compare the incoming `updated_on` timestamp against the stored chunk index 0.
    *   If the timestamps are identical, the document content hasn't changed. The writer skips writing this document entirely.
2.  **Check-Then-Delete**:
    *   For documents that *have* changed, it issues a `delete_many(where=Filter.by_property("document_id").contains_any(changed_doc_ids))` to clear out the old vector chunks, records, and entity associations.
    *   This prevents orphaned chunks from lingering when a page's size is reduced.
3.  **Batch Insertion**:
    *   Inserts the new chunks, records, and entities into Weaviate using `insert_many()`.
    *   If duplicates exist or individual inserts fail, it parses `result.has_errors` and logs failures (ignoring safe "already exists" warnings).
4.  **Count Tracking**: Pushes the number of successfully written documents onto a `counter_queue`.

---

### Step 8: Progress Aggregation & Database Finalization
1.  **Counter Aggregator**: A single `CounterWorker` task pulls counts from the `counter_queue`.
2.  **Database Batching**: Rather than executing an update query for every document, it aggregates counts in memory and writes batch updates back to PostgreSQL (e.g. `ds_repo.increment_job_processed_count()`) at a controlled interval (default: 5 seconds).
3.  **Finalization**: In the main parent process, once the target count (`job_target_count`) equals the processed count (`job_processed_count`), the dispatcher calls `_finalize()`. It updates the sync timestamps (`last_started_at` and `last_collected_at`) in the `user_data_source_collections` table, completing the sync run.
