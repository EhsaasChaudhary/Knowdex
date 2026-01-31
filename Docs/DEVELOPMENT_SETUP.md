# Developer Guide & Local Setup

**Everything you need to spin up the Knowdex architecture on your local machine.**

---

## 1. Prerequisites

Before starting, ensure your machine has the following installed:

*   **Docker Desktop** (Engine 24+) - Required for running databases.
*   **Python 3.10+** - For Backend and Workers.
*   **Node.js 18+ (LTS)** - For Frontend and Mobile.
*   **Git** - For version control.
*   **(Optional) NVIDIA Drivers** - If you want to test GPU acceleration locally (otherwise, the worker runs in CPU mode).

---

## 2. Repository Structure

```text
/knowdex
  ├── /apps
  │   ├── /web-client       # React Frontend
  │   ├── /mobile-app       # React Native (Expo)
  │   └── /api-server       # FastAPI Core Backend
  ├── /services
  │   ├── /stt-worker       # Whisper Transcription (Python)
  │   └── /graph-worker     # LLM Extraction Logic (Python)
  ├── /infra
  │   ├── /docker           # Docker Compose files
  │   └── /k8s              # Kubernetes Manifests (Prod)
  └── /scripts              # Helper scripts (seed db, clean up)
```

---

## 3. Quick Start: The "Hybrid" Workflow

We recommend the **Hybrid Workflow**: Run infrastructure (Databases, S3, Queues) in Docker, but run the Code (API, Frontend) on your host machine for hot-reloading.

### Step 1: Clone & Configure
```bash
git clone https://github.com/your-org/knowdex.git
cd knowdex

# Copy the example env file
cp .env.example .env
```

### Step 2: Configure Environment Variables
Open `.env` and fill in the basics. For local dev, the defaults usually work.

```ini
# .env content

# Database Config (Docker defaults)
POSTGRES_URL=postgresql://user:password@localhost:5432/knowdex
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password
REDIS_URL=redis://localhost:6379
QDRANT_URL=http://localhost:6333

# Object Storage (MinIO Local)
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=knowdex-audio

# AI Services
OPENAI_API_KEY=sk-...  # Required for Graph Extraction
HF_TOKEN=hf_...        # Required to download Whisper Model
WORKER_DEVICE=cpu      # Use 'cuda' if you have an NVIDIA GPU
```

### Step 3: Start Infrastructure
Spin up the backing services (Postgres, Neo4j, Redis, MinIO, Qdrant).
```bash
cd infra/docker
docker-compose up -d
```
*Wait ~30 seconds for Neo4j and Postgres to initialize.*

### Step 4: Run the Backend (API)
Open a new terminal.
```bash
cd apps/api-server

# Create venv
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Run Database Migrations
alembic upgrade head

# Start Server
uvicorn main:app --reload --port 8000
```
*API is now running at `http://localhost:8000`*

### Step 5: Run the Worker (The AI Engine)
Open a new terminal. This simulates the GPU worker locally.
```bash
cd services/stt-worker

# Create venv (recommended separate from API)
python -m venv venv
source venv/bin/activate

# Install dependencies (pytorch, whisper, etc.)
pip install -r requirements.txt

# Run Worker
python worker.py
```
*You should see logs: "Worker listening on queue: transcribe_job"*

### Step 6: Run the Frontend
Open a new terminal.
```bash
cd apps/web-client

# Install
npm install

# Run
npm run dev
```
*Frontend is now running at `http://localhost:5173` (or 3000).*

---

## 4. Port Mapping Reference

| Service | Port | Username/Pass (Default) |
| :--- | :--- | :--- |
| **Web Client** | 5173 | - |
| **API Server** | 8000 | - |
| **PostgreSQL** | 5432 | `user` / `password` |
| **Neo4j Browser** | 7474 | `neo4j` / `password` |
| **Neo4j Bolt** | 7687 | - |
| **MinIO Console** | 9001 | `minioadmin` / `minioadmin` |
| **MinIO API** | 9000 | - |
| **Redis** | 6379 | - |
| **Qdrant** | 6333 | - |

---

## 5. Testing the Flow

1.  **Open Frontend:** Go to `http://localhost:5173`.
2.  **Login:** Use the dev login (or register).
3.  **Create Room:** Name it "Test Room".
4.  **Upload:** Upload a small `.mp3` file (try a 30-second sample).
    *   *Check API Logs:* See the upload request.
    *   *Check Worker Logs:* See "Downloading from MinIO...", "Transcribing...", "Extracting...".
5.  **Verify:**
    *   The Frontend should update status to "Processing" -> "Completed".
    *   Go to `http://localhost:7474` (Neo4j Browser) and run `MATCH (n) RETURN n` to see if nodes were created.

---

## 6. Database Management

### Database Migrations (Alembic)
If you modify the SQL models in `api-server/models.py`:
```bash
# Generate migration script
alembic revision --autogenerate -m "Added column X"

# Apply migration
alembic upgrade head
```

### Resetting Data
To wipe everything and start fresh:
```bash
# Stop containers and remove volumes
cd infra/docker
docker-compose down -v

# Restart
docker-compose up -d

# Re-run migrations in API folder
alembic upgrade head
```

---

## 7. Troubleshooting

**Issue: Worker crashes with "Out of Memory"**
*   **Cause:** You are running `large-v3` model on a machine with low RAM.
*   **Fix:** In `.env`, set `WHISPER_MODEL_SIZE=tiny` or `base` for local development. It is less accurate but runs on any laptop.

**Issue: MinIO Connection Refused**
*   **Cause:** Docker networking issues.
*   **Fix:** Ensure `S3_ENDPOINT` in `.env` is `http://localhost:9000` (for local code) but `http://minio:9000` (if running code inside Docker).

**Issue: "CUDA not found"**
*   **Cause:** No NVIDIA GPU detected.
*   **Fix:** Ensure `WORKER_DEVICE=cpu` is set in `.env`.

---

## 8. Deployment (Production)

*   **API/Frontend:** Push to a standard PaaS (Render, Railway, AWS App Runner).
*   **Databases:** Use managed services (AWS RDS, Neo4j Aura, Qdrant Cloud).
*   **Workers:**
    *   Do not deploy the worker to a standard web server.
    *   Deploy `services/stt-worker` to **Modal.com** or **RunPod**.
    *   Update `REDIS_URL` in the cloud worker to point to your managed Redis instance.
