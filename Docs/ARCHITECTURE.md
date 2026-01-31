# System Architecture & Technical Design

**A Hub-and-Spoke architecture designed for asynchronous processing, scalability, and cost efficiency.**

---

## 1. High-Level Design Principles

1.  **Audio First:** The system is optimized for handling binary audio streams and large files, not just text JSON.
2.  **Asynchronous by Default:** Transcription and LLM extraction are slow operations. The user interface must never block while waiting for them. We use queues for everything.
3.  **Clean Separation of Concerns:**
    *   **Core API:** Manages users, permissions, and business logic.
    *   **Worker Fleet:** Stateless, ephemeral machines that handle heavy GPU compute (Transcription/AI).
    *   **Real-Time Layer:** Dedicated service for live audio streaming.
4.  **Cost-Aware Scaling:** Heavy compute (GPUs) scales to zero when not in use.

---

## 2. System Diagram

```text
                                    INTERNET / EXTERNAL
                                            |
+---------------------+           +---------+---------+
|     WEB CLIENT      |           |   MOBILE CLIENT   |
|   (React/Vite)      |           |   (React Native)  |
+----------+----------+           +---------+---------+
           |                                |
           | (File Uploads)                 | (Live Audio Stream)
           v                                v
+---------------------+           +---------------------+
|    LOAD BALANCER    |           |    MEDIA SERVER     |
+----------+----------+           | (LiveKit / WebRTC)  |
           |                      +---------+-----------+
           v                                |
+---------------------+                     |
|      CORE API       | <-------------------+ (Webhooks: Room Ended)
|  (FastAPI Python)   |                     |
+----+-----+----------+                     | (Save Recording)
     |     |                                v
     |     |    (Presigned URLs)  +---------------------+
     |     +--------------------> |   OBJECT STORAGE    |
     |                            |   (S3 / MinIO)      |
     | (1. Enqueue Job)           +---------+-----------+
     v                                      ^
+---------------------+                     |
|   REDIS JOB QUEUE   |                     |
+----------+----------+                     |
           |                                |
           | (2. Consume Job)               |
           v                                |
+-------------------------------------------+-----------+
|               WORKER FLEET (The Factory)              |
|                                                       |
|  [ Worker: STT ] <---(Read Audio)--------+            |
|     (GPU / Whisper)                                   |
|           |                                           |
|           v                                           |
|  [ Worker: PII ] (Redact Sensitive Info)              |
|           |                                           |
|           v                                           |
|  [ Worker: LLM ] (Extract & Embed)                    |
+-----------+-------------------------------------------+
            |
            | (3. Write Data)
            v
+-------------------------------------------------------+
|                   DATA PERSISTENCE                    |
|                                                       |
|  +--------------+  +--------------+  +-------------+  |
|  |  PostgreSQL  |  |    Neo4j     |  |   Qdrant    |  |
|  | (Meta/Users) |  | (Know.Graph) |  | (Embeddings)|  |
|  +--------------+  +--------------+  +-------------+  |
+-------------------------------------------------------+
```

---

## 3. Component Breakdown

### A. The Client Layer (Frontend)
*   **Web Portal:**
    *   **Stack:** React, TypeScript, TailwindCSS, Vite.
    *   **Role:** User dashboard, file uploader, graph visualizer (using libraries like `react-force-graph`), and governance interface.
*   **Mobile App:**
    *   **Stack:** React Native (Expo).
    *   **Role:** The "Microphone." Allows users to join live rooms or record voice memos. Integrates with LiveKit Client SDK for real-time audio.

### B. The Core API (Backend)
*   **Stack:** Python (FastAPI).
*   **Why Python?** Native integration with AI libraries (LangChain, LlamaIndex) and easier type-sharing with the workers.
*   **Responsibilities:**
    *   REST API endpoints.
    *   Authentication & Authorization (RBAC).
    *   Presigned URL generation for S3 uploads.
    *   Job orchestration (sending tasks to Redis).

### C. The Ingestion Layer
*   **Object Storage (S3):** Stores raw audio files (`.mp3`, `.wav`) and processed JSON artifacts.
*   **Live Audio Server (LiveKit):**
    *   Self-hosted WebRTC server.
    *   Handles the complexity of jitter buffers, packet loss, and real-time room management.
    *   **Egress Service:** A component of LiveKit that records tracks and saves them to S3 when a room closes.

### D. The Worker Fleet ("The Factory")
*   **Infrastructure:** Python scripts running in Docker containers. Designed to run on "Serverless GPU" providers (Modal, RunPod) or a local GPU machine.
*   **Worker 1: The Transcriber (GPU)**
    *   **Model:** `faster-whisper` (Large-v3).
    *   **Input:** S3 Audio URL.
    *   **Output:** JSON Transcript with timestamps and speaker diarization (using `pyannote.audio`).
*   **Worker 2: The Knowledge Extractor (CPU/API)**
    *   **Logic:** LangChain / LlamaIndex pipeline.
    *   **Task:** Reads clean transcripts, calls an LLM (GPT-4o or specialized open model), extracts entities/relationships, and computes embeddings.

### E. The Data Layer
1.  **PostgreSQL (Relational Source of Truth):**
    *   Stores `Users`, `Rooms`, `Permissions`, `JobStatuses`, and raw `Transcripts`.
2.  **Neo4j (The Knowledge Graph):**
    *   Stores `Nodes` (Concepts) and `Relationships` (Edges).
    *   Allows Cypher queries like: *"Find all concepts related to 'Database' discussed by 'User A'"*.
3.  **Qdrant or Milvus (Vector Store):**
    *   Stores embeddings of audio snippets and concept definitions.
    *   Enables semantic search ("Show me where they talked about server costs").
4.  **Redis:**
    *   Message broker for the job queue (BullMQ or Celery).

---

## 4. Detailed Data Flow Scenarios

### Flow 1: File Upload (Async Ingestion)
1.  **User** drops a 50MB audio file on the Web UI.
2.  **API** validates the user and generates a presigned S3 PUT URL.
3.  **Browser** uploads file directly to **S3** (bypassing backend bandwidth).
4.  **Browser** notifies **API**: "Upload Complete."
5.  **API** creates an entry in `AudioSessions` table (Status: `PENDING`) and pushes a `transcribe_job` to **Redis**.
6.  **WorkerSTT** picks up the job, downloads audio, runs Whisper.
7.  **WorkerSTT** saves transcript to Postgres and pushes `extract_job` to Redis.
8.  **WorkerLLM** picks up job, extracts concepts, updates Neo4j/VectorDB.
9.  **API** receives a "Job Complete" event and pushes a WebSocket notification to the User's browser.

### Flow 2: Live Room (Real-time to Archive)
1.  **User** starts a room on Mobile App.
2.  **App** connects to **LiveKit Server**.
3.  **LiveKit** manages the audio session.
4.  **User** clicks "End Room."
5.  **LiveKit Egress** compiles the streams into a single audio file and uploads to **S3**.
6.  **LiveKit** fires a webhook to **Core API**: `recording_finished`.
7.  **Core API** triggers the standard extraction pipeline (same as Step 5 in Flow 1).

### Flow 3: The "Agent" Query
1.  **User** asks: "What did we decide about the caching strategy?"
2.  **API** converts query to Vector Embedding.
3.  **Vector DB** returns top 5 relevant transcript snippets.
4.  **Graph DB** returns connected nodes (e.g., `Redis` -> `Cache`, `TTL` -> `Strategy`).
5.  **LLM Agent** synthesizes these inputs into an answer:
    > "On Sept 5th, User Alice proposed using Redis with a 5-minute TTL. (Source: Room 'Backend Sync', 14:02)"

---

## 5. Technology Stack Decisions

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Backend Framework** | **FastAPI** (Python) | High performance, async native, best-in-class AI libraries. |
| **Frontend Framework** | **React** + **Vite** | Ecosystem standard. |
| **Database (Relational)** | **PostgreSQL** | Reliable, robust JSONB support for flexibility. |
| **Database (Graph)** | **Neo4j** (Community) | Most mature graph query language (Cypher). |
| **Database (Vector)** | **Qdrant** | High performance, written in Rust, easy to self-host via Docker. |
| **Task Queue** | **BullMQ** or **Celery** | Reliable background processing. |
| **Transcription** | **faster-whisper** | 4x faster than standard Whisper, runs on consumer GPUs. |
| **LLM Orchestration** | **LangChain** | Standardizes the "chains" of thought for extraction. |
| **Infrastructure** | **Docker Compose** | Easy local dev; path to Kubernetes for production. |

---

## 6. Security Considerations

1.  **Data Isolation:**
    *   Each `Room` has a unique ID. All DB queries are scoped `WHERE room_id = X`.
    *   Vector Search filters must strictly enforce `room_id` or `workspace_id` to prevent data leakage between tenants.
2.  **API Security:**
    *   All endpoints protected via JWT (JSON Web Tokens).
    *   Presigned URLs expire after 15 minutes.
3.  **Worker Security:**
    *   Workers do not have public IP addresses. They communicate only with S3 and the Databases via internal networks.

---

## 7. Scalability Limits & Bottlenecks

*   **Transcription:** This is the bottleneck. A single GPU can handle ~15-20 concurrent streams (non-realtime). We scale by adding more GPU workers horizontally.
*   **Graph Write Speed:** Writing complex relationships to Neo4j can be slow. We mitigate this by batching updates (write graph updates once per file, not per sentence).
*   **Vector Search:** Qdrant scales well to millions of vectors.
