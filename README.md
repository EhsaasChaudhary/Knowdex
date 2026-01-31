# Knowdex: The Audio-to-Knowledge Engine

**Turn ephemeral voice conversations into a living, queryable technical knowledge graph.**

---

## 1. The Executive Summary

**Knowdex** is an platform that transforms unstructured audio—from live conference rooms, voice memos, or uploaded recordings—into structured, verifiable knowledge.

While traditional tools (like Otter.ai or Zoom) stop at generating a flat text transcript, Knowdex goes further. It uses low-cost, high-performance AI to analyze the conversation, extract atomic concepts, map their relationships, and build a persistent **Knowledge Graph**. This allows both humans and AI agents to query *reasoning*, not just keyword matches.

**The Workflow:**
`Audio Input` $\rightarrow$ `Cost-Efficient Transcription` $\rightarrow$ `LLM Extraction` $\rightarrow$ `Knowledge Graph` $\rightarrow$ `Agent Interface`

---

## 2. The Problem: The "Dark Data" of Audio

Valuable intellectual property is generated every day in voice conversations, yet it remains inaccessible.

1.  **Voice is Ephemeral:** Great architectural decisions, brainstorming breakthroughs, and complex debugging happen in voice channels (Discord, Google Meet, Hallway tracks). Once the call ends, the context evaporates.
2.  **Transcripts are "Noise":** A 1-hour meeting generates 7,000 words of text. It is full of repetition, filler words, and unstructured banter. No one reads raw transcripts.
3.  **Static Documentation Lacks Context:** A README file tells you *what* the code does. It rarely tells you *why* option A was chosen over option B. That reasoning is buried in a recording somewhere, inaccessible to search.
4.  **The "Cost" Barrier:** converting audio to useful data at scale is traditionally expensive, relying on APIs that charge per minute, making it prohibitive for 24/7 community audio.

---

## 3. The Solution: Structured Audio Intelligence

Knowdex is designed to bridge the gap between **Audio Streams** and **Structured Knowledge Bases**.

### A. Audio-First Ingestion
Whether it is a live room hosted on our mobile app or a massive folder of uploaded MP3s, Knowdex treats audio as the primary source of truth. It links every piece of knowledge back to the exact timestamp where it was spoken.

### B. The Knowledge Graph (Not just notes)
Instead of summarizing a meeting into a paragraph, Knowdex breaks it down into **Nodes** and **Edges**.
*   *Example:* Instead of a text summary saying "We discussed databases," Knowdex creates:
    *   **Node:** `PostgreSQL`
    *   **Node:** `MongoDB`
    *   **Edge:** `PostgreSQL` *is_better_for* `Relational Data` (Confidence: 0.9)
    *   **Edge:** `MongoDB` *was_rejected_because* `Lack of ACID compliance` (Source: 12:45 timestamp)

### C. Low-Cost "Serverless" Processing
We solve the cost barrier by utilizing a **Self-Hosted Batch Strategy**. Instead of expensive APIs, Knowdex orchestrates fleeting GPU workers to transcribe and process audio at a fraction of the market cost (approx. $0.20/hour of audio vs. industry standard $0.36-$1.00).

---

## 4. Core Pillars

### 1. Provenance & Trust
AI hallucinates. Humans make mistakes. In Knowdex, every node in the graph cites its source. You can click a concept and immediately listen to the 30-second audio clip where it was defined.
*   *Motto:* "Don't trust the summary; verify the source."

### 2. Human-in-the-Loop Governance
The AI proposes; the Human disposes. The extraction engine generates **Draft Concepts**. Community moderators or room owners verify and merge these drafts into the canonical graph. This prevents the graph from becoming a "trash heap" of bad data.

### 3. Agent-Ready Data
We are not just building this for humans. We are building it for AI Agents. By storing data in a Graph + Vector format, future AI agents can reason over the data ("Why did we choose React?") rather than just retrieving text chunks.

---

## 5. Concrete User Scenarios

### Scenario A: The Open Source Maintainer
*   **Context:** A team holds a weekly 2-hour governance call on Discord.
*   **Action:** They route the audio to Knowdex.
*   **Result:** Knowdex builds a graph of "Decisions made."
*   **Later:** A new contributor asks, "Why aren't we using TypeScript?" The Agent answers, referencing the specific timestamp from a meeting 3 months ago where the complexity trade-off was discussed.

### Scenario B: The Conference Organizer
*   **Context:** A tech conference has 50 hours of recorded talks.
*   **Action:** Bulk upload to Knowdex.
*   **Result:** A navigable "Wiki" of the conference.
*   **Usage:** A user asks, "Who talked about Zero Knowledge Proofs?" Knowdex visualizes the 4 speakers and how their definitions of ZK-Proofs overlapped or differed.

### Scenario C: The R&D Team
*   **Context:** Engineers perform a "Post-Mortem" on a failed deployment in a room.
*   **Action:** Record via Knowdex Mobile App.
*   **Result:** The root causes are extracted as nodes linked to the specific error logs discussed.

---

## 6. Strategic Differentiation

| Feature | Otter.ai / Zoom AI | Notion / Obsidian | **Knowdex** |
| :--- | :--- | :--- | :--- |
| **Input** | Audio | Text / Manual Typing | **Audio** |
| **Output** | Flat Text / Summary | Static Documents | **Knowledge Graph** |
| **Data Structure** | Unstructured | Tree / Folder | **Networked (Nodes/Edges)** |
| **Queryability** | Keyword Search | Keyword Search | **Semantic Reasoning** |
| **Cost Model** | Expensive SaaS Sub | Free / SaaS | **Low-Cost** |
| **Verification** | None | Manual | **Community / AI Hybrid** |

---

## 7. Success Definition (MVP)

The MVP is successful if:
1.  A user can create a **Room**.
2.  A user can upload an **Audio File**.
3.  The system automatically generates a **Transcript** (via low-cost worker).
4.  The system extracts at least **5 coherent Concepts** and links them.
5.  A user can **Ask a Question** and get an answer that cites the timestamp.
