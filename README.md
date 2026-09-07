# Sales Voice Agent

An autonomous voice agent that conducts real outbound sales calls for e-commerce website development.

Author: Manisharma Sugurthi

---

## What This System Does

Priya is a voice salesperson. She calls a prospect, pitches e-commerce website development, holds a natural conversation, figures out how serious the buyer is, and acts accordingly before the call even ends.

Specifically:

- Greets prospects and pitches ElevateBox's web development service
- Detects language from the first sentence (Telugu, Hindi, English) and mirrors it throughout the call, including natural code-switching
- Asks discovery questions woven into conversation, not as a checklist: budget, what they sell, how many products, required timeline, specific features
- Classifies the lead as hot, warm, or cold based on indirect signals, not keywords
- Fires a WhatsApp notification mid-call the moment a lead crosses the hot threshold (before the call ends)
- Parses vague spoken callback times like "tomorrow morning" into a concrete IST datetime and books the callback
- Sends a post-call summary WhatsApp when the call ends

---

## Architecture

The system has two tiers: a Python backend deployed on Heroku that handles all voice and AI logic, and a Next.js frontend deployed on Vercel that reads from the database in real-time.

```
Caller
  |
  v
Vapi (telephony + STT + TTS)
  |
  |-- POST /chat/completions --> FastAPI on Heroku
  |                               |
  |                               |-- Gemini 2.5 Flash (LLM brain)
  |                               |-- State machine (in-memory per call)
  |                               |-- asyncio write-through to Turso
  |                               |-- WhatsApp fire-and-forget
  |
  |-- POST /vapi/webhook     --> FastAPI on Heroku
                                  |
                                  |-- Post-call WhatsApp summary
                                  |-- State cleanup

Turso Edge Database (SQLite)
  |
  v
Next.js on Vercel (real-time telemetry dashboard)
  |-- SSE stream: polls Turso every 1s, streams transcript delta to browser
  |-- Live classification badge: Hot / Warm / Cold with confidence %
  |-- Extracted intelligence panel: budget, sells, timeline, features
  |-- System actions panel: WhatsApp dispatched, callback booked
```

**Core design principle:** The LLM extracts and understands. Deterministic Python decides and acts. One Gemini call per turn, then plain Python for state mutations, routing, triggering, and scheduling.

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| LLM | Gemini 2.5 Flash | Best free-tier multilingual support for Telugu and Hindi. Low latency on flash model. |
| LLM fallback | Groq (Llama) | Swap via `LLM_PROVIDER=groq` in .env. Useful if Gemini quota runs out. |
| Voice + telephony | Vapi | Handles STT, TTS, barge-in, and call lifecycle. We plug in as a Custom LLM. |
| WhatsApp | Meta Cloud API | Free tier, template messaging to evaluator's phone. |
| Backend server | FastAPI + Uvicorn | Async-native Python. Low overhead. Works well with Vapi's tight 3s latency requirement. |
| Backend hosting | Heroku | Stable public HTTPS endpoint for Vapi webhooks. Free tier (eco dynos) is sufficient. |
| Database | Turso Edge SQLite | Persistent call state across dyno restarts. Globally distributed. Free hobby tier. |
| Frontend | Next.js 15 (App Router) | Native SSE support, React Server Components, zero-config Vercel deployment. |
| Frontend hosting | Vercel | Instant deployments from git push. |
| CSS | CSS Modules (vanilla) | No Tailwind. Full control. Bespoke telemetry aesthetic without generated utility bloat. |

---

## Project Structure

```
elevatebox/
  app/
    main.py          # FastAPI app. Four endpoints, lifespan handler, all orchestration logic.
    llm.py           # Gemini adapter. One function: think(). Has 5s timeout + fallback.
    prompts.py       # System prompt builder. Rebuilds every turn from current state.
    state.py         # Per-call state machine. In-memory dict keyed by Vapi call_id.
    scheduler.py     # Callback time parser. Pure Python. No LLM involved.
    whatsapp.py      # Meta Cloud API adapter. Fire-and-forget with retry.
    db.py            # Turso client. init_db() on startup, sync_state_to_turso() per turn.
    config.py        # Environment loader. Graceful degradation if keys are missing.
  frontend/
    src/app/
      page.tsx                      # Main dashboard. Shows all active calls. Polls every 2s via SWR.
      page.module.css               # Bespoke telemetry grid styles.
      [id]/page.tsx                 # Live call view. Opens SSE connection to stream transcript.
      [id]/call.module.css          # Split-pane styles.
      api/calls/route.ts            # GET /api/calls — fetches all calls from Turso.
      api/calls/[id]/route.ts       # GET /api/calls/:id — fetches one call's full state.
      api/stream-call/[id]/route.ts # GET /api/stream-call/:id — SSE stream, polls Turso every 1s.
    src/lib/
      turso.ts                      # Turso @libsql/client singleton.
  tests/             # Unit and integration tests. All external services are mocked.
  Procfile           # Heroku process definition.
  requirements.txt   # Pinned Python dependencies.
```

---

## API Endpoints (Backend)

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Heroku uptime check. Returns 200 with config status flags. |
| POST | `/chat/completions` | Vapi Custom LLM endpoint. Called every conversation turn. Must respond under 3s. |
| POST | `/vapi/webhook` | Vapi lifecycle events: assistant-request (inbound), end-of-call-report, status-update. |
| POST | `/call/trigger` | Initiates an outbound call via Vapi REST API. Requires a paid Vapi phone number. |

---

## API Endpoints (Frontend)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/calls` | Returns last 50 calls from Turso, ordered by updated_at. |
| GET | `/api/calls/:id` | Returns full state for a single call including transcript. |
| GET | `/api/stream-call/:id` | Opens a Server-Sent Events stream. Polls Turso every 1s and pushes new transcript lines to the browser. |

---

## How the Voice Turn Works

Every time the prospect speaks, Vapi sends the full conversation history to `/chat/completions`. Here is what happens in sequence:

1. Extract the `call_id` from `body.call.id`. Get or create a `CallState` for that call.
2. Build the system prompt from current state. This tells Gemini exactly which discovery fields are still missing, what language to use, and what the current classification is.
3. Call `llm.think()` with a 5-second timeout. If it times out, return a fallback line and keep the call alive.
4. Merge the LLM output into the call state. Language locks on first detection and never changes. Discovery fields only fill in once; captured values are protected from hallucination overwrite.
5. If a callback phrase was detected, parse it deterministically with `scheduler.parse_callback_time()`. No LLM arithmetic.
6. Fire `asyncio.create_task(sync_state_to_turso(st))`. This is a write-through cache: state updates to the database without blocking the voice response.
7. If `fire_whatsapp_now=true` and the WhatsApp has not been sent yet, fire `_fire_hot_lead_whatsapp()` as a background task.
8. Return the spoken line in OpenAI format. Vapi speaks it.

---

## Engineering Decisions

**One LLM call per turn.** The system prompt, discovery extraction, classification, and action flags all come back in a single JSON from one Gemini request. We never make a second LLM call to "verify" anything. This keeps the turn latency under 3 seconds.

**Structured JSON output via `response_mime_type`.** We set `response_mime_type="application/json"` in the Gemini config. This forces the model to return parseable JSON without markdown fences, which eliminates a whole class of parsing failures.

**Discovery field protection.** Once Priya captures that the prospect sells artificial jewelry, that value is locked. If Gemini hallucinates a different value on the next turn, `update_from_llm()` ignores it. This is enforced with simple `if not state.discovery.budget` guards, not regex.

**Language lock with native script enforcement.** We detect language on the first user turn and lock it. After the lock, the prompt tells Gemini to stay in that language. Critically, when Telugu or Hindi is used, the prompt mandates native script output (Telugu: తెలుగు లిపి, Hindi: देवनागरी) because the Naina TTS engine will mispronounce Romanized Tanglish severely.

**Deterministic scheduling.** Callback time parsing is pure Python using `python-dateutil`. "Tomorrow morning" becomes the next day at 10:00 AM IST. This is not passed to the LLM. LLMs cannot do reliable datetime arithmetic.

**Fire-and-forget WhatsApp.** WhatsApp sends use `asyncio.create_task()`, not `await`. The voice response returns immediately. WhatsApp delivery happens concurrently. The `whatsapp_sent` flag is set to `True` before the network call so a second turn arriving before delivery completes does not fire a duplicate.

**Write-through Turso cache.** Every turn fires `sync_state_to_turso()` as an `asyncio.create_task()`. In-memory state is the source of truth during the call (fast). Turso is the persistence layer (survives dyno restarts and is readable by the frontend). The write never blocks the voice response.

**Frontend uses native SSE, not the Vercel AI SDK.** The Vercel AI SDK (`useChat`, `streamText`) is designed for frontend-to-LLM direct streaming. Our LLM is on the Python backend for latency reasons. The Next.js frontend is a telemetry viewer, not an AI client. We use `ReadableStream` and `TextEncoder` directly, which is exactly what the Vercel AI SDK uses internally, without the unnecessary abstraction.

**No Tailwind.** The dashboard uses CSS Modules with CSS custom properties. This gives full control over the aesthetic without generated utility class noise in the HTML. The telemetry look was a deliberate choice: dark base (`#09090b`), `JetBrains Mono` for data fields, and three-color classification system (emerald for hot, amber for warm, blue for cold).

---

## Local Setup

```bash
git clone https://github.com/sharma-sugurthi/sales-voice-agent.git
cd sales-voice-agent
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# Fill in GEMINI_API_KEY, TURSO_DATABASE_URL, TURSO_AUTH_TOKEN at minimum

uvicorn app.main:app --reload --port 8000
```

For the frontend:

```bash
cd frontend
npm install
# Create frontend/.env.local with TURSO_DATABASE_URL and TURSO_AUTH_TOKEN

# Requires Node 20+
nvm use 20
npm run dev
```

---

## Testing

```bash
pytest tests/ -v
```

All tests mock external services (Gemini, Vapi, WhatsApp). No API calls are made. No cost incurred.

---

## Deployment

**Backend:** Heroku eco dyno. `Procfile` runs `uvicorn app.main:app --host 0.0.0.0 --port $PORT`. Set all env vars via Heroku config vars dashboard.

**Frontend:** Vercel. Set root directory to `frontend`. Add `TURSO_DATABASE_URL` and `TURSO_AUTH_TOKEN` as environment variables. Every push to `master` triggers a new deployment automatically.

**Database schema:** Run `init_db()` once to create the `calls` table in Turso. The lifespan handler in `app/main.py` calls this on every backend startup, so the schema is always created if it does not exist.

---

## Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Gemini 2.5 Flash access |
| `TURSO_DATABASE_URL` | Yes | Turso database endpoint (`libsql://...`) |
| `TURSO_AUTH_TOKEN` | Yes | Turso auth token |
| `PUBLIC_BASE_URL` | Yes | Public HTTPS URL of the Heroku backend |
| `EVALUATOR_PHONE` | Yes | Phone number to receive WhatsApp notifications |
| `MY_PHONE` | Yes | Developer phone shown in post-call summary |
| `WHATSAPP_TOKEN` | No | Meta Cloud API token. WhatsApp disabled if missing. |
| `WHATSAPP_PHONE_NUMBER_ID` | No | Meta Cloud API phone ID. WhatsApp disabled if missing. |
| `VAPI_API_KEY` | No | Vapi API key. Outbound calling disabled if missing. |
| `VAPI_PHONE_NUMBER_ID` | No | Vapi phone number for outbound calls. |
| `GROQ_API_KEY` | No | Groq API key for LLM fallback. |
| `LLM_PROVIDER` | No | `gemini` (default) or `groq`. |
| `GCP_PROJECT_ID` | No | If set, Gemini uses Vertex AI instead of the direct API. |
