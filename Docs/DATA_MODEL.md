# Data Model & Schema Specification

**A hybrid data strategy combining Relational, Graph, and Vector paradigms.**

---

## 1. Database Strategy Overview

*   **PostgreSQL:** Handles User Auth, Room Management, Billing, Permissions, and Raw Transcripts. (The Source of Truth).
*   **Neo4j:** Stores the Knowledge Graph (Concepts, Relations, and their links to source Audio). (The Reasoning Engine).
*   **Qdrant / Milvus:** Stores high-dimensional vectors for semantic search. (The Retrieval Engine).

---

## 2. Relational Schema (PostgreSQL)

### A. Users & Auth
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    avatar_url TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    role VARCHAR(50) DEFAULT 'user' -- 'user', 'admin'
);
```

### B. Workspace / Rooms
```sql
CREATE TABLE rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    visibility VARCHAR(20) DEFAULT 'private', -- 'public', 'private', 'unlisted'
    is_live BOOLEAN DEFAULT FALSE,
    livekit_room_name VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Access Control
CREATE TABLE room_members (
    room_id UUID REFERENCES rooms(id),
    user_id UUID REFERENCES users(id),
    role VARCHAR(20) DEFAULT 'viewer', -- 'viewer', 'editor', 'moderator'
    PRIMARY KEY (room_id, user_id)
);
```

### C. Audio Ingestion
```sql
CREATE TABLE audio_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID REFERENCES rooms(id),
    uploader_id UUID REFERENCES users(id), -- Nullable if system recorded
    s3_key TEXT NOT NULL,
    file_name VARCHAR(255),
    duration_seconds INTEGER,
    status VARCHAR(50) DEFAULT 'queued', -- 'queued', 'processing', 'completed', 'failed'
    error_message TEXT,
    recorded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### D. Transcripts (The Raw Text)
```sql
CREATE TABLE transcript_segments (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID REFERENCES audio_sessions(id) ON DELETE CASCADE,
    speaker_label VARCHAR(50), -- "Speaker A", or mapped to User Name if known
    start_time FLOAT NOT NULL,
    end_time FLOAT NOT NULL,
    text_content TEXT NOT NULL,
    embedding_id VARCHAR(255) -- Reference to Vector DB ID
);
-- Index for fast retrieval by time
CREATE INDEX idx_transcript_session ON transcript_segments(session_id, start_time);
```

---

## 3. Graph Schema (Neo4j)

The graph represents the *distilled knowledge*. We do not dump the whole transcript here; only extracted entities.

### A. Nodes (Labels)

1.  **`Concept`**
    *   `id`: UUID
    *   `name`: "Event Driven Architecture" (Canonical Name)
    *   `definition`: "A software architecture paradigm promoting the production..."
    *   `status`: "draft" | "verified"

2.  **`Person`** (Optional mapping to Users)
    *   `id`: UUID
    *   `name`: "Alice"

3.  **`SourceChunk`** (Links Graph back to Audio)
    *   `id`: UUID
    *   `session_id`: Postgres AudioSession UUID
    *   `start_time`: 120.5
    *   `text`: "I think we should use Redis for caching."

### B. Relationships (Edges)

1.  **Semantic Relations (Concept to Concept)**
    *   `(:Concept)-[:RELATED_TO {type: "dependency", confidence: 0.9}]->(:Concept)`
    *   `(:Concept)-[:CONTRADICTS {confidence: 0.8}]->(:Concept)`
    *   `(:Concept)-[:IS_A]->(:Concept)` (Taxonomy, e.g., Redis IS_A Database)

2.  **Provenance Relations (Concept to Source)**
    *   `(:Concept)-[:DEFINED_IN]->(:SourceChunk)`
    *   `(:Concept)-[:DISCUSSED_IN]->(:SourceChunk)`

3.  **Social Relations (Person to Source)**
    *   `(:Person)-[:SPOKE]->(:SourceChunk)`

---

## 4. Vector Schema (Qdrant)

We need two collections for different search strategies.

### Collection 1: `raw_transcripts`
Used for "Find me where they said X".
*   **Payload:**
    ```json
    {
      "text": "We decided to drop feature X because of budget.",
      "session_id": "uuid...",
      "room_id": "uuid...",
      "start_time": 450.0,
      "speaker": "Alice"
    }
    ```
*   **Vector:** 1536-dim (OpenAI `text-embedding-3-small` or similar).

### Collection 2: `concept_definitions`
Used for Entity Resolution (checking if a concept already exists).
*   **Payload:**
    ```json
    {
      "concept_name": "Next.js",
      "definition": "A React framework for production...",
      "concept_id": "uuid..."
    }
    ```

---

## 5. Entity Resolution Logic (The "Merge" Strategy)

When the LLM extracts a new term, e.g., "The React Library", we must decide: **Is this a new node or an existing one?**

**Algorithm:**
1.  **Extract:** LLM finds entity "The React Library".
2.  **Embed:** Create vector for "The React Library".
3.  **Search:** Query `concept_definitions` collection in Vector DB (scoped to `room_id`).
4.  **Compare:**
    *   If top match distance < 0.1 (Very close): It matches existing node `React`. **Action:** Link to `React`.
    *   If top match distance < 0.3 (Somewhat close): It might be `React Native` vs `React`. **Action:** Create "Candidate Node" and flag for Human Review.
    *   If top match distance > 0.3 (Far): **Action:** Create new node `The React Library`.

---

## 6. Data Lifecycle & Pruning

*   **Transcription:** Raw `audio_sessions` (MP3s) in S3 have a lifecycle rule to expire (delete) after 30 days.
*   **Vector/Graph:** These persist indefinitely. They are the "Value" of the platform.
*   **Orphaned Nodes:** A nightly Cron Job runs in Neo4j to find `Concept` nodes that have 0 incoming/outgoing edges and delete them to keep the graph clean.

---

## 7. JSON Payloads (LLM Interface)

When the worker sends text to the LLM for extraction, it expects this JSON structure back to populate the graph:

```json
{
  "concepts": [
    {
      "name": "Microservices",
      "definition": "An architectural style that structures an application as a collection of services.",
      "aliases": ["Micro-services", "MSA"]
    }
  ],
  "relationships": [
    {
      "source": "Microservices",
      "target": "Monolith",
      "relation_type": "CONTRASTS_WITH",
      "reasoning": "Speaker A mentioned moving away from Monolith to Microservices."
    }
  ]
}
```
