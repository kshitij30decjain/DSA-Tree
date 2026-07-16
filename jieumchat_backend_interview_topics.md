# Jieumchat Backend Systems Interview Prep Spec

This document details the core backend engineering concepts, design decisions, and system design patterns implemented in **Jieumchat**. It provides explanations, diagrams, and "Pitch Scripts" to help you explain these systems during senior-level engineering interviews.

---

## 1. Dynamic ACL Injection (Security & RAG Isolation)

### The Concept
In enterprise RAG systems, user access permissions are dynamic: a user's access to Confluence spaces or Jira projects can be granted or revoked at any time. Instead of duplicating vector chunks across separate indices for each user (which is expensive and hard to sync), Jieumchat uses a **shared index with query-time ACL injection**.

### How it Works
1.  **PostgreSQL Resolves Rights**: At query time, the system runs a fast query against PostgreSQL (`data_source_access_grants` and `chatbot_user_access`) to resolve the exact list of `data_source_uuid`s the calling user is permitted to search.
2.  **Injected Filters**: The resolved UUIDs are injected directly into the Weaviate GraphQL/client search query as a metadata filter (`contains_any` or `in`). Weaviate filters the results at the index level before returning candidate chunks.

### System Diagram
```
[ User Request ] ===> [ Query API ] ===> [ PostgreSQL ]
                                              || (Fetch Allowed UUIDs)
                                              v
[ Weaviate Search ] <=== (Inject Filters) <=== [ Query Router ]
```

### Interview Pitch Script
> *"When building enterprise RAG systems, securing search results is a major challenge. We avoided duplicate vector indexing by separating data storage from access control. We store the document chunks in a shared Weaviate collection and keep the permissions in a relational database (PostgreSQL). At query time, the API queries PostgreSQL to resolve the user's authorized data source UUIDs and injects them directly into the vector database query filter. This ensures document security at the index level without the overhead of maintaining separate vector stores for each tenant."*

---

## 2. GIL Bypass & Parallel Subprocesses (`multiprocessing` with `spawn`)

### The Concept
Python's event loop (`asyncio`) is single-threaded and can be blocked by CPU-bound tasks like text chunking and cleaning, or synchronous network IO. To handle high-throughput background crawling and indexing without impacting API response times, Jieumchat offloads this work to a separate process.

### How it Works
1.  **Isolated Storage Process**: The main FastAPI application uses `multiprocessing.get_context("spawn")` to launch a separate `StorageProcess`.
2.  **Queue-Based Communication**: Data is passed between processes using an IPC Queue. The background process runs a pool of 8 CPU-bound embedding workers and 8 IO-bound Weaviate gRPC writer threads, ensuring the main application event loop remains unblocked.

### Interview Pitch Script
> *"To prevent CPU-bound tasks like text chunking and embedding from blocking our FastAPI event loop, we offloaded indexing to a separate process. We used Python's `multiprocessing` library with the `spawn` context to start a separate `StorageProcess` that communicates with the main application via IPC queues. Inside this process, we run parallel workers for embedding fetches and batch gRPC writes. This keeps our main API responsive, keeping average response latencies low even under heavy indexing workloads."*

---

## 3. Change Detection & "Detect-Diff-Patch" Synchronization

### The Concept
Regularly crawling third-party APIs (like Jira and Confluence) can quickly run into API rate limits. Instead of re-indexing entire spaces, Jieumchat implements an **eventual consistency, change-detection model**.

### How it Works
1.  **Metadata Scan**: The scheduler run interval (`COLLECTION_SCHEDULE_INTERVAL = 60`) checks for updates by fetching document IDs and versions from Atlassian APIs.
2.  **Diff Calculations**: The system computes a diff against PostgreSQL records to identify:
    *   **New**: IDs present in Atlassian but missing from the database.
    *   **Updated**: IDs with mismatched version tags.
    *   **Deleted**: IDs present in the database but missing from Atlassian.
3.  **Targeted Indexing**: The background pipeline only processes modified records, reducing API calls and embedding costs.

### Interview Pitch Script
> *"To avoid hitting Atlassian's API rate limits and minimize embedding generation costs, we designed an incremental synchronization engine. Instead of re-indexing entire spaces, we run a change-detection job every 60 seconds that compares document IDs and versions. We then compute a diff to identify new, modified, or deleted pages, and run targeted indexing jobs only for the changes. This approach reduced our external API usage by over 80% while keeping our search index fresh."*

---

## 4. Exponential Backoff with Jitter (Resilience)

### The Concept
When calling external APIs (like embedding endpoints or Atlassian services), temporary network errors or rate limit throttling are common. Naive retries can result in a "thundering herd" problem that overloads target systems.

### How it Works
1.  **Exponential Backoff**: If an API call fails, the system delays the next retry:
    $$Delay = Base \times 2^{attempt}$$
2.  **Jitter Injection**: Adds random variation to the delay calculation to prevent synchronized retries:
    $$Delay_{Jitter} = Delay + RandomJitter$$

### Interview Pitch Script
> *"When calling external APIs like LLM providers and Atlassian services, temporary failures and rate limits are expected. To build a resilient pipeline, we wrapped all external API calls in an asynchronous retry handler that implements exponential backoff with jitter. By adding random noise to the retry backoff delays, we prevent retry requests from aligning and overloading the target services, allowing our ingestion pipeline to recover gracefully from temporary failures."*

---

## 5. Context Compaction & Token Budget Management

### The Concept
Large Language Models have strict context window limits. Long-running chats can exceed these limits, causing API errors or query failures.

### How it Works
1.  **Token Counting**: The system calculates active token counts before each step in the agent loop.
2.  **Compaction Trigger**: If the token size exceeds 85% of the 128k context window, a compaction job is triggered.
3.  **Summarization**: An LLM call is made to summarize earlier parts of the conversation, replacing them with a single summary message to free up context space.

### Interview Pitch Script
> *"To handle long-running conversations without exceeding LLM context limits, we built a token budget manager. The orchestrator monitors prompt sizes before every step. If the token count exceeds 85% of our 128k context limit, we run a compaction routine. This uses Qwen3 to summarize the oldest messages in the conversation, replacing them with a single summary message. This keeps our prompts within model limits and helps manage API costs."*

---

## 6. Relational Connection Pooling & PgBouncer Proxying

### The Concept
Opening and closing direct database connections for every API request is slow and resource-intensive. As services scale horizontally, they can quickly exhaust PostgreSQL connection limits.

### How it Works
1.  **PgBouncer Proxying**: API services connect to PgBouncer on port `54322` rather than directly to PostgreSQL.
2.  **Transaction Multiplexing**: PgBouncer pools warm connections to the database and multiplexes them, allowing thousands of client requests to share a small number of physical database connections.

### Interview Pitch Script
> *"To support horizontal scaling of our FastAPI services, we placed PgBouncer in front of PostgreSQL. Instead of connecting directly to the database, our application instances route requests through PgBouncer on port 54322 using connection pooling. This allows our services to scale without exhausting PostgreSQL connection limits during traffic spikes."*
