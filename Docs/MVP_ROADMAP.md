# MVP Roadmap & Phasing

**Goal:** Get from "Zero" to "Working Audio Knowledge Graph" in 12 weeks.

---

## Phase 1: The "Recorder" (Weeks 1-4)
**Focus:** Infrastructure, Audio Ingestion, and Basic Transcription.
*Objective: A user can upload an MP3 and see a text transcript 2 minutes later.*

### Week 1: Foundation
*   [Backend] Init FastAPI repo, Docker Compose, PostgreSQL schema.
*   [Infra] Set up MinIO (Local S3) and Redis (Queue).
*   [Frontend] Init React + Tailwind. Basic Auth (Login/Signup).

### Week 2: The Worker Factory
*   [Worker] Write the Python script for `faster-whisper`.
*   [Cloud] Deploy the worker to **Modal.com** (or setup local GPU dev env).
*   [Backend] Implement `POST /upload/presign` and Job Queue logic.

### Week 3: Integration
*   [Frontend] Build File Upload UI with progress bar.
*   [Backend] Connect the "Upload Complete" webhook to trigger the Worker.
*   [Frontend] Create "Transcript View" page (Text only, no graph yet).

### Week 4: Diarization & Cleanup
*   [Worker] Integrate `pyannote.audio` for Speaker Diarization ("Speaker A" labels).
*   [Backend] Add PII Redaction step (Microsoft Presidio) before saving text.
*   **Milestone 1:** *Alpha Release (Internal). We have a self-hosted "Otter.ai" clone.*

---

## Phase 2: The "Extractor" (Weeks 5-8)
**Focus:** LLM Logic, Graph Database, and Visualization.
*Objective: Turn the text transcript into a navigable node-network.*

### Week 5: Graph Init
*   [Infra] Spin up Neo4j container. Define Indexes/Constraints.
*   [Backend] Implement LLM Client (OpenAI/LangChain).
*   [Logic] Write the "Extraction Prompt" (System instructions for the LLM).

### Week 6: The Pipeline
*   [Worker] Create the `Extractor` job (Run LLM on transcript segments).
*   [Backend] Logic to save extracted Nodes/Edges to Neo4j.
*   [Data] Implement "Entity Resolution" (Check if node exists before creating).

### Week 7: Visualization
*   [Frontend] Integrate `react-force-graph` or similar library.
*   [Frontend] Build the "Graph Explorer" view. Users can click nodes to see definitions.
*   [Frontend] Add "Play Audio" button on Node click (Jump to timestamp).

### Week 8: The "Human Loop"
*   [Frontend] "Review Mode": Users see "Draft Concepts" and click [Approve] or [Reject].
*   **Milestone 2:** *Beta Release. Users can see their audio visualized as a graph.*

---

## Phase 3: The "Agent" & Mobile (Weeks 9-12)
**Focus:** Q&A Interface and Live Audio.
*Objective: Chat with the data and record on the go.*

### Week 9: Vector Search (RAG)
*   [Infra] Spin up Qdrant (Vector DB).
*   [Worker] Add Embedding step (Generate vectors for every transcript chunk).
*   [Backend] Implement `/agent/query` endpoint with RAG logic.

### Week 10: Agent UI
*   [Frontend] Build Chat Interface ("Ask your audio").
*   [Backend] Format citations (Link answers back to Source Audio).

### Week 11: Mobile Lite
*   [Mobile] Init React Native (Expo).
*   [Mobile] Implement LiveKit SDK for "Live Room" recording.
*   [Backend] Handle LiveKit `recording_finished` webhooks.

### Week 12: Polish & Ship
*   [DevOps] Deploy to Production (AWS/DigitalOcean/Render).
*   [QA] Load testing (upload 10 files at once).
*   [Docs] Finalize user guide.
*   **Milestone 3:** *MVP Launch.*

---

## 7. Success Metrics (KPIs)

1.  **Transcription Success Rate:** >99% of uploaded files process successfully.
2.  **Processing Speed:** 1 Hour of Audio processed in <5 Minutes.
3.  **Cost Efficiency:** Cost per hour of audio < $0.10.
4.  **Graph Quality:** User rejection rate of extracted concepts < 20% (indicates LLM is accurate).

---

## 8. Out of Scope for MVP (Post-Launch)
*   User-to-User Direct Messaging.
*   Public "Explore" Feed of other communities.
*   Monetization / Stripe Integration.
*   Fine-tuning custom LLMs.
*   SSO / Enterprise Auth.
