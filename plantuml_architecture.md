# Jieumchat: PlantUML System Diagrams

This document contains the official **PlantUML** source code for Jieumchat's backend architecture and pipeline flows. You can copy these code blocks directly into any PlantUML viewer (such as [PlantText](https://www.planttext.com/) or the VS Code PlantUML extension) to render and export them.

---

## 1. System Component & Architecture Diagram

```plantuml
@startuml
!theme black-knight
skinparam docker true
skinparam componentStyle uml2

package "Client Layer" {
    [Next.js Frontend] as WebUI
}

package "Query Handler Service (Port 8031)" {
    [FastAPI Query Handler] as QH
    [RAG Agent Loop] as QH_Agent
    [BGE-M3 Reranker] as Reranker
}

package "Data Collection Service (Port 8034)" {
    [FastAPI Data Collection] as DC
    [APScheduler] as Sched
    [Collection Dispatcher] as Dispatcher
    
    package "Stage 3 (Main Process)" {
        [Collection Workers Pool] as CW_Pool
        [Token Cache] as TC
    }
    
    package "Stage 4 (Storage Process - Isolated OS Process)" {
        [Storage Process Loop] as SP
        [Embedding Workers Pool] as EW_Pool
        [Weaviate Writers Pool] as WW_Pool
        [Counter Worker] as CW
    }
}

database "PostgreSQL" as DB
database "Weaviate Vector DB" as Vec
[Embedding API Server] as EmbedAPI
[Atlassian APIs (Jira/Confluence)] as Atlassian

' Connections
WebUI --> QH : POST /api/query
QH --> QH_Agent
QH_Agent --> Vec : Vector Search + ACLs
QH_Agent --> Reranker : Rerank

WebUI --> DC : Admin Triggers
Sched --> Dispatcher : Trigger every 60s
Dispatcher --> DB : Read Approved DS & PATs
Dispatcher --> CW_Pool : Create CollectionJobs

CW_Pool --> TC : Read PATs
TC --> Atlassian : Validate / Rate limit
CW_Pool --> Atlassian : Fetch content
CW_Pool --> SP : Enqueue StorageJob (mp.Queue)

SP --> EW_Pool
EW_Pool --> EW_Pool : Chunk & Tokenize
EW_Pool --> EmbedAPI : Batch Embed
EW_Pool --> WW_Pool : EmbeddedBatch (asyncio.Queue)
WW_Pool --> Vec : Check-then-Delete Old Chunks
WW_Pool --> Vec : Batch Insert Chunks
WW_Pool --> CW : Progress counts
CW --> DB : Batch increment processed
@enduml
```

---

## 2. Ingestion Sequence Diagram

```plantuml
@startuml
!theme black-knight
autonumber

actor "APScheduler" as S
participant "Collection\nDispatcher" as D
database "PostgreSQL" as PG
participant "Collection\nWorkers" as CW
entity "Atlassian API" as AP
participant "Storage Process\n(Isolated Process)" as SP
database "Weaviate DB" as WV

S -> D : Trigger collect_per_pat()
activate D

D -> PG : Fetch approved sources & valid PATs
activate PG
PG --> D : Return Source Metadata & PAT Owner list
deactivate PG

== Stage 1: DETECT ==
D -> AP : Query get_total_count() & fetch_ids_with_versions()
AP --> D : Return doc IDs & last-update versions
D -> D : Set-diff against local state\n(isolate changed IDs)

== Stage 2: DISPATCH ==
D -> D : Shard doc IDs by PAT eligibility
D -> CW : Enqueue CollectionJobs (50 docs per chunk)
deactivate D

== Stage 3: COLLECT ==
loop For each CollectionJob
    activate CW
    CW -> CW : Resolve active PAT from TokenCache
    CW -> AP : fetch_by_ids(doc_ids, PAT)
    AP --> CW : Return full page content / issues
    CW -> SP : Enqueue StorageJob (via mp.Queue)
    deactivate CW
end

== Stage 4: STORE ==
loop In StorageProcess Event Loop
    activate SP
    SP -> SP : Chunk text (size=512, overlap=0.3)
    SP -> SP : Request Embeddings from Embedding API
    SP -> WV : Query & compare updated_on timestamps
    note right of SP: Skip identical docs;\ndelete changed docs
    SP -> WV : Batch insert new vector chunks
    SP -> PG : Batch increment processed counts
    deactivate SP
end
@enduml
```

---

## 3. Query Execution Sequence Diagram

```plantuml
@startuml
!theme black-knight
autonumber

actor User
participant "Query Handler API" as QH
participant "LLM Server (Qwen)" as LLM
database "PostgreSQL" as PG
database "Weaviate Vector DB" as WV
participant "BGE Reranker" as RR

User -> QH : POST /api/query { query, x-user-id }
activate QH

QH -> LLM : Rewrite query based on context history
LLM --> QH : Return search keywords

QH -> PG : Fetch visible data_source_uuids for x-user-id
activate PG
PG --> QH : Return permitted source UUID list (ACL)
deactivate PG

QH -> WV : Query hybrid search (BM25 + vector)\n+ Filter by ACL UUIDs
activate WV
WV --> QH : Return relevant matching text chunks
deactivate WV

QH -> QH : Merge adjacent/overlapping chunks
QH -> RR : Send merged chunks to reranker
activate RR
RR --> QH : Return semantic similarity scores
deactivate RR

QH -> QH : Apply Recency decay multiplier to scores
QH -> QH : Check context size & compact prompt history

QH -> LLM : Send final prompt (history + context)
activate LLM

loop Streaming response chunks
    LLM --> QH : Stream tokens
    QH --> User : SSE stream event (data: {...})
end

deactivate LLM
deactivate QH
@enduml
```
