# VoxScreen — Voice Interview Agent

A real-time conversational screening interviewer that conducts structured multi-stage interviews over bidirectional voice audio, adapts follow-up questions to candidate answers, supports mid-sentence interruption and pause/resume workflows, persists full transcripts, and generates rubric-scored candidate reports.

## Capabilities

- **Structured Multi-Stage Interviews**: Guides every candidate through five stages: `opening`, `technical_core`, `adaptive_deep_dive`, `behavioral_scenario`, and `wrap_up`.
- **Contextual Follow-Up & Adaptive Flow**: Listens to candidate answers and probes specific technical decisions, trade-offs, and metrics rather than stepping through a static script. Stage transitions and adaptation notes are tracked via tool calls (`advance_stage`, `conclude_interview`).
- **Real-Time Voice Interaction**: Streams 16 kHz 16-bit PCM microphone audio over WebSockets and plays back gapless 24 kHz PCM voice responses scheduled through Web Audio API buffers.
- **Interruption (Barge-In) Handling**: Detects when a candidate speaks over the interviewer (or triggers a manual interrupt), immediately halts queued playback nodes, flags the interrupted turn in the transcript, and shifts back to listening.
- **Pause & Resume Handling**: Suspends active audio streams, persists the full transcript, elapsed duration, and current stage to disk, and hydrates the full conversation context when the candidate rejoins so answered questions are never repeated.
- **Multi-Role Support**: Ships with rubrics for Senior Backend Engineer, Frontend Systems Engineer, Voice Systems Engineer, Machine Learning Engineer, and Product Manager, plus an API and UI for adding custom roles.
- **Persistent Transcript Storage**: Saves timestamped speaker turns, stage labels, interruption counts, and resume counts to a local JSON store (`interviews_store.json` in standalone mode, `data/store.json` in full workspace mode).
- **Structured Candidate Reports**: Generates JSON-schema-validated evaluations containing an overall score (0–100), hiring recommendation (`Strong Hire`, `Hire`, `Lean Hire`, `No Hire`), executive summary, per-competency scores with verbatim transcript evidence quotes, strengths, concerns, question-by-question depth ratings, and next-round focus areas.

## Architecture

```
Browser (Web Audio API)
  ├── Mic Capture (16kHz PCM LE) ──► WebSocket (/ws/interview) ──► Server ──► Live Voice Model
  ├── Playback Queue (24kHz PCM) ◄── WebSocket (/ws/interview) ◄── Server ◄── Audio + Transcripts
  └── REST Client (/api/*) ────────► Express API ──► Persistent JSON Store & Evaluation Engine
```

### Key Components

1. **Audio Pipeline**
   - Client captures microphone input with echo cancellation and noise suppression, resamples to 16 kHz mono 16-bit little-endian PCM, and streams base64 frames over `/ws/interview`.
   - Server forwards incoming PCM frames to the real-time voice session and relays 24 kHz PCM output chunks back to the client.
   - Client schedules each output chunk sequentially on a dedicated 24 kHz `AudioContext` using a running `nextStartTime` cursor to avoid gaps or clicks.

2. **Interruption & Resume State Machine**
   - On barge-in (`interrupted` event or manual interrupt action), every active `AudioBufferSourceNode` in the playback set is stopped immediately and the queue cursor resets.
   - On pause, pending transcript buffers are flushed and saved with `status: "paused"`.
   - On resume, a fresh voice session opens with the saved transcript and current stage injected into the system instructions, allowing seamless continuation.

3. **Evaluation Pipeline**
   - Calling `POST /api/sessions/:id/evaluate` submits the complete saved transcript and role competencies to a schema-constrained evaluator that outputs structured JSON matching the `CandidateEvaluation` interface.

## Project Files

- `voice_interview_agent.ts`: Complete standalone single-file implementation containing the Express REST API, WebSocket voice bridge, persistent store, structured evaluator, and embedded web client.
- `README.md`: Architecture overview, setup instructions, API reference, and storage schema.

## Setup & Running

### Prerequisites

- Node.js 20+
- A valid `GEMINI_API_KEY` environment variable

### Install Dependencies

```bash
npm install express ws @google/genai dotenv
npm install -D tsx typescript @types/express @types/ws @types/node
```

### Configure Environment

Create a `.env` file in the project root:

```env
GEMINI_API_KEY="your_api_key_here"
PORT=3001
```

### Run the Single-File Agent

```bash
npx tsx voice_interview_agent.ts
```

Open `http://localhost:3001` in a browser, grant microphone permission, select a role, and start an interview.

## REST & WebSocket Reference

### HTTP Endpoints

| Method | Path | Description |
| :--- | :--- | :--- |
| `GET` | `/api/roles` | List built-in and custom job role rubrics |
| `POST` | `/api/roles` | Add a custom role with competencies and seed questions |
| `GET` | `/api/sessions` | List saved interview sessions |
| `GET` | `/api/sessions/:id` | Fetch full transcript and report for a session |
| `POST` | `/api/sessions` | Create a new candidate screening session |
| `POST` | `/api/sessions/:id/evaluate` | Generate and persist a structured evaluation report |

### WebSocket Protocol (`/ws/interview`)

**Client to Server messages:**
- `{ "type": "start", "sessionId": "int-...", "resume": false }`
- `{ "type": "audio", "audio": "<base64-pcm16-16khz>" }`
- `{ "type": "text", "text": "<optional-keyboard-response>" }`
- `{ "type": "manual_interrupt" }`
- `{ "type": "pause", "elapsedSeconds": 184 }`
- `{ "type": "finish", "elapsedSeconds": 420 }`

**Server to Client messages:**
- `{ "type": "ready", "stage": "opening" }`
- `{ "type": "audio", "audio": "<base64-pcm16-24khz>" }`
- `{ "type": "transcript_committed", "entry": { ... } }`
- `{ "type": "interrupted" }`
- `{ "type": "stage_updated", "stage": "adaptive_deep_dive" }`
- `{ "type": "error", "message": "..." }`

## Transcript & Report Schema

Each stored session in `interviews_store.json` follows this structure:

```json
{
  "id": "int-a1b2c3d4",
  "candidateName": "Jordan Patel",
  "roleId": "role-backend",
  "roleTitle": "Senior Backend Engineer",
  "seniority": "Senior",
  "voiceName": "Kore",
  "status": "completed",
  "currentStage": "wrap_up",
  "adaptationNotes": [
    "Probed cache invalidation strategy after candidate mentioned Redis write-through."
  ],
  "interruptionCount": 2,
  "resumeCount": 1,
  "elapsedSeconds": 512,
  "transcript": [
    {
      "id": "turn-1234abcd",
      "speaker": "interviewer",
      "text": "Hi Jordan, thanks for joining today...",
      "timestamp": "2026-11-25T08:40:00.000Z",
      "stage": "opening",
      "interrupted": false
    }
  ],
  "evaluation": {
    "overallScore": 84,
    "recommendation": "Hire",
    "executiveSummary": "...",
    "competencyScores": [],
    "strengths": [],
    "concerns": [],
    "questionBreakdown": [],
    "nextRoundFocus": []
  }
}
```
