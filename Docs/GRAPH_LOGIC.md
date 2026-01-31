# Graph Logic & Entity Resolution Strategy

**The algorithms required to maintain a clean, high-quality Knowledge Graph and ensure user privacy.**

---

## 1. The Core Challenge: The "messy graph" Problem

In a naive implementation, LLMs will extract whatever they hear.
*   **Transcript 1:** "We use ReactJS." -> Creates node `ReactJS`.
*   **Transcript 2:** "I like React." -> Creates node `React`.
*   **Transcript 3:** "The frontend library." -> Creates node `Frontend Library`.

This results in a disconnected graph where `React` and `ReactJS` are not linked. Our logic must solve this via **Entity Resolution**.

---

## 2. The Entity Resolution Pipeline

This logic runs inside the `Graph Worker` *after* transcription but *before* writing to Neo4j.

### Step A: Extraction (The LLM Pass)
We ask the LLM to identify concepts.
*   **Input:** "I think Next.js is great for SEO."
*   **LLM Output:** `{"candidate": "Next.js", "type": "Technology"}`

### Step B: Vector Search (The Deduplication Pass)
Before creating the node "Next.js", we check if it already exists in the Graph.
1.  **Embed:** Generate a vector embedding for the string "Next.js".
2.  **Query:** Search the Vector DB (`concept_definitions` collection) for the nearest neighbors within the same `Workspace/Room`.
3.  **Threshold Logic:**
    *   **Distance < 0.1 (Exact/Near Match):**
        *   *Example:* "Next.js" vs "NextJS".
        *   **Action:** Do NOT create a new node. Use the `UUID` of the existing node. Create a new link to the current audio source.
    *   **Distance 0.1 - 0.3 (Ambiguous Match):**
        *   *Example:* "Next.js" vs "Nuxt.js".
        *   **Action:** Create a new node, BUT automatically create a `[:POSSIBLY_RELATED]` edge between them. Flag for human review.
    *   **Distance > 0.3 (No Match):**
        *   **Action:** Create a new unique Node.

### Step C: Synonym Mapping (The Alias List)
Every `Concept` node in Neo4j has an `aliases` array property.
*   **Node:** `React`
*   **Aliases:** `['React.js', 'ReactJS', 'The Facebook Library']`
*   **Logic:** When checking existence, we search against the Name AND the Aliases list.

---

## 3. Relationship Logic

### 1. Confidence Scores
Every Edge in the graph must have a `confidence` property (0.0 - 1.0).
*   **Implicit (0.5):** "I think A is related to B."
*   **Explicit (0.9):** "A inherits from B."
*   **Consensus (1.0):** Multiple speakers agree, or a Human Moderator verified it.

### 2. Temporal Evolution
Knowledge changes.
*   **Jan 1st:** "Use Redux." (Node: `Redux`, Status: `Adopted`)
*   **March 1st:** "Redux is too heavy, let's use Zustand." (Node: `Redux`, Status: `Deprecated`; Node: `Zustand`, Status: `Adopted`).
*   **Logic:** Edges have `created_at`. When querying "Current Stack," the Agent filters for the most recent decisions.

---

## 4. PII Redaction & Privacy (The Safety Layer)

We must ensure we don't permanently store credit card numbers, personal phone numbers, or secrets mentioned in audio.

### The "Clean Room" Workflow

1.  **Raw Audio** (S3) -> **Whisper** -> **Raw Text**.
    *   *Note:* Raw Text exists *only* in the Worker's memory temporarily.
2.  **PII Filter (Microsoft Presidio):**
    *   Run the Raw Text through Presidio (local NLP model).
    *   Detect entities: `PHONE_NUMBER`, `EMAIL_ADDRESS`, `CREDIT_CARD`, `SSN`.
    *   **Action:** Replace with placeholder `<REDACTED_PHONE>`.
3.  **Save to DB:** Only the **Redacted Text** is saved to Postgres and Vector DB.
4.  **Graph Extraction:** The LLM sees only the Redacted Text.

### Presidio Configuration (Python)
```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

def scrub_text(text):
    results = analyzer.analyze(text=text, entities=["PHONE_NUMBER", "EMAIL_ADDRESS"], language='en')
    anonymized_result = anonymizer.anonymize(text=text, analyzer_results=results)
    return anonymized_result.text
```

---

## 5. The "Graph Merging" UI (Human-in-the-Loop)

Even with Vector Search, duplicates will happen. We need a UI flow to fix them.

### The Merge Operation
User selects Node A (`NextJS`) and Node B (`Next.js`) and clicks **"Merge"**.
1.  **Backend Logic:**
    *   Identify Target Node (A) and Source Node (B).
    *   Move all relationships from B to A.
        *   `MATCH (b)-[r]->(x) CREATE (a)-[new_r]->(x) DELETE r`
    *   Add B's name to A's `aliases` list.
    *   Delete Node B.
    *   Update Vector DB: Point embeddings for "Next.js" to Node A's UUID.

---

## 6. Cold Start Management (UX Logic)

Since we use Serverless GPUs, there is a delay (Cold Start) when the first user uploads a file.

**Worker Lifecycle States:**
1.  **Sleeping:** Cost = $0.
2.  **Booting:** (10-40 seconds).
3.  **Running:** Processing audio.
4.  **Cooldown:** Stays active for 5 mins after last job, then sleeps.

**UI Feedback Loop:**
*   Instead of a generic spinner, the WebSocket sends specific status codes:
    *   `STATUS: QUEUED` (Waiting for worker)
    *   `STATUS: PROVISIONING` (Worker waking up - show "Waking up AI..." in UI)
    *   `STATUS: PROCESSING` (Progress bar moves)
    *   `STATUS: SAVING` (Writing to Graph)

---

## 7. Garbage Collection

The graph can get cluttered with "orphan nodes" (concepts mentioned once, never connected to anything).

**The Nightly Job:**
1.  Identify nodes with `degree == 0` (No edges).
2.  Identify nodes created > 7 days ago.
3.  Check `status`: If `draft` (unverified), DELETE node.
4.  This keeps the graph high-signal.

This logic ensures your Knowledge Graph remains a valuable asset, rather than a noisy dumping ground of transcripts.
