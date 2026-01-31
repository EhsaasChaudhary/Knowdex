# Low-Cost Audio Pipeline Strategy

**How to achieve professional-grade transcription and diarization at 1/30th the cost of SaaS APIs.**

---

## 1. The Cost Problem
If we use standard commercial APIs (OpenAI Whisper API, Google Speech-to-Text, Deepgram) to process audio, the business model breaks at scale.

*   **Commercial API Cost:** ~$0.36 per hour (OpenAI) to $1.00+ per hour (Google/AWS).
*   **Scenario:** A community uploads 1,000 hours of conference talks.
*   **Bill:** $360 - $1,000 just for text. This is unsustainable for a startup/community tool.

## 2. The Solution: "Serverless" Batch Processing
We will use **Open Source Models** hosted on **On-Demand GPU Infrastructure**.

*   **Model:** `faster-whisper` (Large-v3).
*   **Infrastructure:** Modal.com or RunPod Serverless.
*   **Our Cost:** ~$0.01 - $0.02 per hour of audio processed.
*   **Savings:** ~95% cheaper than OpenAI API.

---

## 3. The Technology Stack

### A. The Engine: `faster-whisper`
We do not use the original OpenAI implementation. We use [Systran/faster-whisper](https://github.com/SYSTRAN/faster-whisper).
*   **Why:** It uses CTranslate2, a fast inference engine for Transformer models.
*   **Speed:** It is 4x faster than standard Whisper.
*   **Memory:** It uses significantly less VRAM (Video RAM), allowing us to use cheaper GPU tiers.

### B. The Diarizer: `pyannote.audio`
Standard transcription gives you a wall of text. We need to know *who* spoke to attribute knowledge correctly.
*   **Tool:** [pyannote/speaker-diarization](https://huggingface.co/pyannote/speaker-diarization).
*   **Function:** It analyzes the audio waveform to detect speaker changes and assigns labels (SPEAKER_00, SPEAKER_01).
*   **Integration:** We align the Whisper timestamps with Pyannote segments to produce a transcript like:
    > **Speaker A (00:00):** "Welcome to the meeting."
    > **Speaker B (00:05):** "Thanks."

### C. The Hardware Provider
We need GPUs (NVIDIA T4 or A10G), but renting a server 24/7 costs ~$400/mo. We only need the GPU for the 2 minutes it takes to process a file.
*   **Provider Choice:** **Modal.com** (Recommended for ease of use) or **RunPod Serverless**.
*   **Billing Model:** Pay-per-second of execution time. If no one uploads audio, cost is $0.

---

## 4. Implementation Logic (The Pipeline)

This is the exact logic the Worker Script performs.

### Step 1: Pre-processing (CPU)
*   **Input:** MP3/WAV/M4A file from S3.
*   **Action:** Convert audio to 16kHz Mono WAV (standard requirement for Whisper).
*   **Tool:** `ffmpeg`.

### Step 2: Transcription (GPU)
*   **Action:** Run `faster-whisper` model `large-v3`.
*   **Output:** List of segments with `start`, `end`, and `text`.
*   *Note:* We use `beam_size=5` for best accuracy.

### Step 3: Diarization (GPU)
*   **Action:** Run `pyannote` pipeline on the audio.
*   **Output:** List of `turn` segments: `start`, `end`, `speaker_label`.

### Step 4: Alignment (The "Zipper" Merge)
This is the hardest algorithmic part. Whisper segments and Pyannote segments rarely match perfectly.
*   **Algorithm:** We iterate through Whisper words. If the timestamp of the word falls inside a Pyannote speaker segment, we assign that word to that speaker.
*   **Result:** A JSON object grouped by speaker turns.

```json
[
  {
    "speaker": "SPEAKER_01",
    "start": 0.0,
    "end": 4.5,
    "text": "We need to discuss the database migration."
  },
  {
    "speaker": "SPEAKER_02",
    "start": 4.6,
    "end": 10.0,
    "text": "I agree. Postgres is the target."
  }
]
```

---

## 5. Handling Live Audio (The "Live" Strategy)

We **do not** transcribe in real-time. Real-time transcription requires a dedicated GPU spinning 24/7 for every active room, which destroys the cost model.

**Strategy: "Near-Time" Processing**
1.  **Record:** LiveKit records the audio stream to a buffer.
2.  **Chunk:** Every 5 or 10 minutes (configurable), the chunk is saved to S3.
3.  **Trigger:** The system triggers the Transcription Worker on that chunk.
4.  **Update:** The UI updates with the new text.
5.  **Experience:** Users see the transcript appear on a slight delay (e.g., 2 minutes behind live), which is acceptable for a "Knowledge Capture" tool (vs. a live captioning tool).

---

## 6. Cost Analysis (Example)

**Scenario:** You process 1 hour (60 minutes) of audio.

**Using OpenAI API:**
*   $0.006 per minute * 60 minutes = **$0.36**

**Using Knowdex Pipeline (Modal.com with NVIDIA T4):**
*   `faster-whisper` speed factor on T4 GPU: ~15x realtime (1 hour audio takes ~4 minutes to process).
*   NVIDIA T4 Cost: ~$0.000164 per second.
*   Processing time: 240 seconds.
*   Total Cost: 240 * $0.000164 = **$0.039**

**Result:** **~9x Cheaper** (and this gap widens with faster GPUs like A100s for batching).

---

## 7. Retention & Privacy Policy

Since we are processing raw voice data, we must define retention strictness to manage storage costs and privacy.

1.  **Raw Audio (S3):**
    *   *Policy:* Deleted after 30 days by default (using S3 Lifecycle Rules).
    *   *Option:* Room Owners can pay/opt-in to "Archival Storage" (Cold Storage) to keep audio longer.
2.  **Transcripts (Text):**
    *   *Policy:* Kept forever in the Database.
3.  **PII Redaction:**
    *   Before the text is saved to the DB or sent to the LLM, we run a "PII Scrubber" (using Microsoft Presidio) to replace phone numbers and emails with `[REDACTED]`.

---

## 8. Failure Modes & Recovery

1.  **OOM (Out of Memory):** Large audio files (>2 hours) can crash the RAM during Diarization.
    *   *Fix:* The worker script checks duration. If >1 hour, it splits the audio into 1-hour chunks, processes them in parallel, and stitches the JSON back together.
2.  **Hallucinations:** Whisper sometimes hallucinates text during silence.
    *   *Fix:* Use the VAD (Voice Activity Detection) filter built into `faster-whisper` to skip silent segments entirely.
