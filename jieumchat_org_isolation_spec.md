# Jieumchat Organization Data Isolation Specification

This document details the multi-tenant architecture and logical isolation strategy used to separate data between **Joyent** and **MX** systems within the Jieumchat RAG database.

---

## 1. Storage Strategy: Shared Indices with Logical Boundaries

Jieumchat does not use separate Weaviate instances or Weaviate's native tenant namespace feature to isolate data between organizations. Instead, it utilizes a **shared-index model with query-time filtering**:

*   **Weaviate Vector DB**: All text chunks, issues, pages, and identity cards for both `joyent` and `mx` reside inside the same physical Weaviate collections (`UNIFIED_SCHEMA`, `RECORD_SCHEMA`, `ENTITY_SCHEMA`).
*   **PostgreSQL Relational DB**: Serves as the source of truth for the organization hierarchy, credential ownership, and access grants.

---

## 2. Structural Mapping

```
+-----------------------------------------------------------------------------------+
|                            POSTGRESQL RELATIONAL SCHEMA                           |
|                                                                                   |
|  [data_source_config]                                                             |
|  +--------------------+-------+-------------+-------------+                       |
|  | uuid (PK)          | org   | source_type | resource_id |                       |
|  +--------------------+-------+-------------+-------------+                       |
|  | uuid-joyent-wiki   | joyent| confluence  | DEVOPS      |                       |
|  | uuid-mx-jira       | mx    | jira        | SSO         |                       |
|  +--------------------+-------+-------------+-------------+                       |
+-----------------------------------------------------------------------------------+
          |
          | (Links via data_source_uuid)
          v
+-----------------------------------------------------------------------------------+
|                            WEAVIATE VECTOR DATABASE                               |
|                                                                                   |
|  [UNIFIED_SCHEMA] (Shared Index)                                                  |
|  +---------------------+-------------------+------------------------------------+  |
|  | data_source_uuid    | document_id       | chunk_text (Vectorized)            |  |
|  +---------------------+-------------------+------------------------------------+  |
|  | uuid-joyent-wiki    | CONF-4412         | "To configure Joyent DNS servers..."|  |
|  | uuid-mx-jira        | SSO-102           | "The MX SSO gateway needs..."       |  |
|  +---------------------+-------------------+------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

---

## 3. Query Execution & Permission Resolution Flow

When a user executes a RAG search, organizational isolation is enforced at runtime by resolving permissions in PostgreSQL before querying Weaviate.

### Step 1: Ingress Authentication
The ingress gateway passes the user's ID in the `x-user-id` header (e.g. `bk21.choi`).

### Step 2: PostgreSQL Permission Resolution
The API service runs a database query to identify the data sources the user is authorized to view. This query:
1.  Checks the user's registered tokens in `user_pat` to identify active organizations (e.g. if the user only has a token registered for `joyent`, they only have access to `joyent` resources).
2.  Queries access grants in `data_source_access_grants` (for shared spaces) and active filters in `user_enabled_data_sources`.
3.  Returns the allowed `data_source_uuid` list (e.g., `['uuid-joyent-wiki']`).

### Step 3: Weaviate Filter Injection
The resolved UUID list is injected directly into Weaviate's search query as a metadata filter:

```python
from weaviate.classes.query import Filter

# 1. Resolved authorized UUIDs from PostgreSQL
allowed_uuids = ["uuid-joyent-wiki"]

# 2. Inject filter into Weaviate hybrid query
collection = weaviate_client.collections.get("UnifiedSchema")
response = collection.query.hybrid(
    query="SSO configuration guidelines",
    filters=Filter.by_property("data_source_uuid").contains_any(allowed_uuids),
    alpha=0.5
)
```

Because of this filter, Weaviate's search engine isolates the query to the specified data sources, preventing cross-organization data leakage.

---

## 4. Architectural Trade-offs & Interview Script

### Why we avoided native Weaviate Multi-Tenancy:
*   **Duplicate Vector Costs**: Weaviate's native tenant feature isolates data by physical tenant, which works well if each user maps to exactly one tenant. However, in our system, users have access to a dynamically changing mix of data sources based on group memberships and chatbot sharing. If we used native tenants, we would have to duplicate vector chunks across multiple tenants, leading to high storage and embedding costs.
*   **Index Overhead**: Creating separate collections or tenants for every permutation of sharing configurations would lead to index resource bloat.

### Interview Pitch Script:
> *"In our RAG pipeline, we isolate data between organizations (Joyent and MX) logically at the application layer rather than physically separating them in our vector store. We store all vector chunks in a shared Weaviate collection where each record is tagged with a `data_source_uuid`. 
> 
> The organizational mappings and access permissions are managed in PostgreSQL. When a user runs a search, our API queries PostgreSQL to retrieve their authorized data source UUIDs and injects them as a metadata filter into our Weaviate query. This ensures strict data isolation, prevents cross-organization data leakage, and avoids the storage overhead of duplicating vectors across tenant indices."*
