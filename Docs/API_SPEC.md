# API Specification (REST)

**Base URL:** `https://api.knowdex.io/v1` (or `http://localhost:8000/v1` for dev)
**Protocol:** HTTPS / JSON
**Authentication:** Bearer Token (JWT) in `Authorization` header.

---

## 1. Authentication

*   **POST** `/auth/login`
    *   **Description:** Login via Email/Password or exchange OAuth code.
    *   **Body:** `{ "email": "...", "password": "..." }`
    *   **Response:** `{ "access_token": "ey...", "token_type": "bearer", "user": { ... } }`

*   **POST** `/auth/refresh`
    *   **Description:** Refresh an expired access token.

---

## 2. Room Management

*   **GET** `/rooms`
    *   **Description:** List all rooms the user has access to.
    *   **Response:** `[ { "id": "...", "name": "Eng Sync", "role": "owner" }, ... ]`

*   **POST** `/rooms`
    *   **Description:** Create a new room.
    *   **Body:** `{ "name": "Architecture Talk", "description": "Weekly sync", "visibility": "private" }`
    *   **Response:** `{ "id": "uuid", "created_at": "..." }`

*   **GET** `/rooms/{room_id}`
    *   **Description:** Get room details + stats (e.g., total audio hours).

*   **DELETE** `/rooms/{room_id}`
    *   **Description:** Archive/Delete a room (Owner only).

---

## 3. Audio & Uploads

*   **POST** `/rooms/{room_id}/uploads/presign`
    *   **Description:** Request a presigned S3 URL to upload a file directly from the client.
    *   **Body:** `{ "filename": "meeting.mp3", "content_type": "audio/mpeg", "size_bytes": 50000000 }`
    *   **Response:**
        ```json
        {
          "upload_url": "https://s3.aws.com/ bucket/key?signature=...",
          "file_key": "rooms/123/uploads/meeting.mp3",
          "session_id": "new-session-uuid"
        }
        ```

*   **POST** `/rooms/{room_id}/uploads/confirm`
    *   **Description:** Notify backend that S3 upload is finished. Triggers the Transcription Worker.
    *   **Body:** `{ "session_id": "uuid" }`
    *   **Response:** `{ "status": "queued", "job_id": "redis-job-id" }`

---

## 4. Live Audio (Mobile App Integration)

*   **POST** `/rooms/{room_id}/join`
    *   **Description:** Request a LiveKit access token to join the real-time audio room.
    *   **Response:** `{ "token": "ey...", "ws_url": "wss://livekit.knowdex.io" }`

*   **WEBHOOK** `/webhooks/livekit/recording-finished`
    *   **Description:** Endpoint called by LiveKit server when a room closes and recording is ready. (Internal/Secured).
    *   **Body:** LiveKit Egress JSON payload.

---

## 5. Transcripts & Sessions

*   **GET** `/rooms/{room_id}/sessions`
    *   **Description:** List all audio sessions in a room.
    *   **Response:** `[ { "id": "...", "status": "completed", "duration": 120, "created_at": "..." } ]`

*   **GET** `/sessions/{session_id}/transcript`
    *   **Description:** Get the full text transcript with timestamps.
    *   **Response:**
        ```json
        {
          "segments": [
            { "speaker": "Speaker A", "start": 0.0, "end": 5.2, "text": "Hello world" }
          ]
        }
        ```

---

## 6. Knowledge Graph (Visual & Data)

*   **GET** `/rooms/{room_id}/graph`
    *   **Description:** Fetch nodes and edges for visualization (filtered by search or limit).
    *   **Query Params:** `?limit=100&search=database`
    *   **Response:**
        ```json
        {
          "nodes": [ { "id": "1", "label": "Concept", "properties": { "name": "Redis" } } ],
          "edges": [ { "from": "1", "to": "2", "label": "RELATED_TO" } ]
        }
        ```

*   **GET** `/concepts/{concept_id}`
    *   **Description:** Get detailed info for a single concept, including audio citations.
    *   **Response:**
        ```json
        {
          "name": "Redis",
          "definition": "In-memory data store...",
          "citations": [
            { "session_id": "...", "start_time": 120, "text_snippet": "We use Redis for caching." }
          ]
        }
        ```

*   **PUT** `/concepts/{concept_id}`
    *   **Description:** Update/Verify a concept (Human-in-the-loop).
    *   **Body:** `{ "status": "verified", "definition": "Updated definition..." }`

---

## 7. Agent / RAG Search

*   **POST** `/agent/query`
    *   **Description:** Ask a natural language question about the room's knowledge.
    *   **Body:**
        ```json
        {
          "room_id": "uuid",
          "query": "What did we say about the deadline?",
          "history": [] // Optional chat history
        }
        ```
    *   **Response:** (Streamed or Static)
        ```json
        {
          "answer": "The deadline was moved to Friday.",
          "sources": [
            { "id": "uuid", "timestamp": 500, "confidence": 0.95 }
          ]
        }
        ```

---

## 8. WebSocket Events

**Endpoint:** `wss://api.knowdex.io/ws`

Used for real-time updates on:
1.  **Transcription Progress:** `{"type": "job_update", "progress": 50, "status": "processing"}`
2.  **New Graph Nodes:** `{"type": "graph_update", "node": {...}}`

---

## 9. Error Handling Standard

All error responses follow this format:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The session ID provided does not exist.",
    "details": {}
  }
}
```

*   **400:** Bad Request (Validation failed).
*   **401:** Unauthorized (Invalid token).
*   **403:** Forbidden (You don't have access to this room).
*   **404:** Not Found.
*   **429:** Too Many Requests (Rate limit).
*   **500:** Internal Server Error.
