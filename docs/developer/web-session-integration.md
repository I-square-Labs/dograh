# Web Session Integration Guide

This guide walks you through running the Dograh server locally, exposing the new
`POST /public/agent/{uuid}/web-session` endpoint to the internet, and wiring it
into your web app so users can join a live voice session directly from a browser.

---

## What This Endpoint Does

```
Your Backend  ──POST /public/agent/{uuid}/web-session──►  Dograh API
                X-API-Key: <org_api_key>
                Body: { "initial_context": { ... } }

Dograh API returns:
  {
    "run_id": 12345,
    "session_token": "emb_session_...",
    "ws_url": "wss://your-dograh-host/ws/public/signaling/emb_session_...",
    "turn_credentials": { "username": "...", "password": "...", "uris": [...] }
  }

Your Frontend  ──WebSocket→ ws_url──►  Dograh WebRTC signaling
                                        ↓
                                   Pipecat pipeline (STT → LLM → TTS)
                                        ↓
                                   User's microphone / speaker
```

No telephony (no phone number) is required. The call lives entirely in the browser.

---

## Prerequisites

| Tool | Version |
|---|---|
| Docker + Docker Compose | 24+ |
| Python | 3.11+ (for the Python SDK or scripts) |
| Node.js | 18+ (for the Next.js frontend) |
| ngrok (or Cloudflare Tunnel) | Any recent version |

---

## Step 1 — Run the Dograh Server Locally

### 1a. Clone and configure

```bash
git clone https://github.com/dograh-hq/dograh.git
cd dograh
cp api/.env.example api/.env
```

Edit `api/.env` and set at minimum:

```env
# LLM (pick one)
OPENAI_API_KEY=sk-...
# or ANTHROPIC_API_KEY=...

# STT (pick one)
DEEPGRAM_API_KEY=...
# or ASSEMBLYAI_API_KEY=...

# TTS (pick one)
ELEVENLABS_API_KEY=...
# or leave blank to use OpenAI TTS

# TURN server — required for browser WebRTC through NAT
TURN_SECRET=any-random-string-here
```

### 1b. Start all services

```bash
docker compose -f docker-compose-local.yaml up -d
```

This starts PostgreSQL, Redis, MinIO, and the coturn TURN server. Then start the
FastAPI backend:

```bash
python -m venv venv && source venv/bin/activate
pip install -e "api[dev]"
uvicorn api.app:app --reload --port 8000
```

Verify it is up:

```bash
curl http://localhost:8000/api/v1/health
# {"status":"ok","turn_enabled":true,...}
```

---

## Step 2 — Expose the Server to the Internet

The browser needs a publicly reachable `wss://` URL to open the WebRTC signaling
WebSocket. Use ngrok (simplest) or any reverse proxy.

### ngrok

```bash
ngrok http 8000
# Forwarding: https://abc123.ngrok-free.app -> http://localhost:8000
```

Set `BACKEND_API_ENDPOINT` so Dograh generates correct `ws_url` values:

```bash
# In api/.env (then restart uvicorn)
BACKEND_API_ENDPOINT=https://abc123.ngrok-free.app
```

### Cloudflare Tunnel (persistent URL)

```bash
cloudflared tunnel --url http://localhost:8000
```

Set the same `BACKEND_API_ENDPOINT` env var to the tunnel URL.

---

## Step 3 — Create and Publish a Workflow

1. Open the Dograh UI at `http://localhost:3000`
2. Click **New Workflow**
3. Add an **API Trigger** node — this is the entry point; it generates the `{uuid}`
4. Add an **LLM** node. Example coaching system prompt:

```
You are an English fluency coach. Your student is {student_name}, a {level} learner
whose native language is {native_language}. Speak naturally. Gently correct grammar
mistakes by repeating the correct form in your next sentence. Focus on: {focus_area}.
```

5. Connect: API Trigger → LLM → (end)
6. Click **Publish**
7. Open the API Trigger node panel and copy the **Trigger UUID** — you'll use it as
   `{uuid}` in the endpoint URL

---

## Step 4 — Get an API Key

1. In the Dograh UI go to **Settings → API Keys**
2. Click **Create API Key** and copy the key (shown once)
3. Keep it server-side — never expose it in browser JavaScript

---

## Step 5 — Call the Endpoint from Your Backend

The call must originate from your server (to keep the API key secret). Your
frontend then receives the token and opens the WebSocket.

### curl (quick test)

```bash
curl -s -X POST https://abc123.ngrok-free.app/api/v1/public/agent/<trigger-uuid>/web-session \
  -H "X-API-Key: <your-org-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "initial_context": {
      "student_name": "Ana",
      "level": "B1",
      "native_language": "Spanish",
      "focus_area": "business vocabulary"
    }
  }'
```

Expected response:

```json
{
  "run_id": 42,
  "session_token": "emb_session_Xp3...",
  "ws_url": "wss://abc123.ngrok-free.app/ws/public/signaling/emb_session_Xp3...",
  "turn_credentials": {
    "username": "1748123456:api:7",
    "password": "base64-hmac...",
    "ttl": 86400,
    "uris": [
      "turn:localhost:3478",
      "turn:localhost:3478?transport=tcp"
    ]
  }
}
```

### Python (server-side handler)

```python
import httpx

DOGRAH_URL = "https://abc123.ngrok-free.app"
DOGRAH_API_KEY = "dg_..."        # from env, never hard-coded
TRIGGER_UUID = "abc-def-..."     # from workflow API Trigger node

async def create_voice_session(student: dict) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{DOGRAH_URL}/api/v1/public/agent/{TRIGGER_UUID}/web-session",
            headers={"X-API-Key": DOGRAH_API_KEY},
            json={"initial_context": student},
            timeout=10,
        )
        resp.raise_for_status()
        return resp.json()

# In a FastAPI route:
# session = await create_voice_session({"student_name": "Ana", "level": "B1", ...})
# return session  # send to frontend
```

### Python SDK (if using dograh-sdk)

The Python SDK (`sdk/python`) provides a typed `DograhClient`. Wire up the call
via the raw HTTP client until the SDK adds a first-class `web_session` method:

```python
from dograh_sdk import DograhClient

client = DograhClient(base_url=DOGRAH_URL, api_key=DOGRAH_API_KEY)

# Until SDK adds native support, use the underlying httpx client:
resp = client._http.post(
    f"/api/v1/public/agent/{TRIGGER_UUID}/web-session",
    json={"initial_context": {"student_name": "Ana", "level": "B1"}},
)
session = resp.json()
```

---

## Step 6 — Connect the Browser via WebRTC

Your backend returns the session payload to your frontend. The frontend opens the
WebSocket and performs the standard WebRTC offer/answer/ICE handshake.

### Vanilla TypeScript / JavaScript

```typescript
interface TurnCredentials {
  uris: string[];
  username: string;
  password: string;
}

interface WebSession {
  run_id: number;
  session_token: string;
  ws_url: string;
  turn_credentials: TurnCredentials | null;
}

async function startVoiceSession(session: WebSession): Promise<void> {
  const iceServers = session.turn_credentials
    ? [{
        urls: session.turn_credentials.uris,
        username: session.turn_credentials.username,
        credential: session.turn_credentials.password,
      }]
    : [{ urls: "stun:stun.l.google.com:19302" }];

  const pc = new RTCPeerConnection({ iceServers });

  // 1. Capture microphone
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  stream.getTracks().forEach(track => pc.addTrack(track, stream));

  // 2. Play the agent's voice
  pc.ontrack = (event) => {
    const audio = new Audio();
    audio.srcObject = event.streams[0];
    audio.play();
  };

  // 3. Open signaling WebSocket
  const ws = new WebSocket(session.ws_url);

  // 4. Send offer once connected
  ws.onopen = async () => {
    const pcId = crypto.randomUUID();
    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    ws.send(JSON.stringify({
      type: "offer",
      payload: { type: "offer", sdp: offer.sdp, pc_id: pcId },
    }));

    // 5. Forward ICE candidates
    pc.onicecandidate = (e) => {
      if (e.candidate) {
        ws.send(JSON.stringify({
          type: "ice-candidate",
          payload: { pc_id: pcId, candidate: e.candidate },
        }));
      }
    };
  };

  // 6. Apply server answer
  ws.onmessage = async (msg) => {
    const data = JSON.parse(msg.data);
    if (data.type === "answer") {
      await pc.setRemoteDescription(
        new RTCSessionDescription({ type: "answer", sdp: data.payload.sdp })
      );
    } else if (data.type === "ice-candidate") {
      await pc.addIceCandidate(new RTCIceCandidate(data.payload.candidate));
    } else if (data.type === "error") {
      console.error("Signaling error:", data.payload);
    }
  };

  ws.onerror = (e) => console.error("WebSocket error", e);
}
```

> For a production-grade reference implementation (reconnect logic, mic mute,
> level indicators) copy `ui/src/components/embed/` from this repo directly into
> your app.

### React hook example

```tsx
import { useRef, useCallback } from "react";

export function useVoiceSession() {
  const pcRef = useRef<RTCPeerConnection | null>(null);
  const wsRef = useRef<WebSocket | null>(null);

  const start = useCallback(async (session: WebSession) => {
    await startVoiceSession(session);  // function from above
  }, []);

  const stop = useCallback(() => {
    wsRef.current?.close();
    pcRef.current?.close();
  }, []);

  return { start, stop };
}
```

---

## Step 7 — End-to-End Flow in Your App

```
User clicks "Start Session"
  → Your frontend calls your own API route (e.g. POST /api/coaching-session)
  → Your backend calls Dograh: POST /public/agent/{uuid}/web-session
  → Dograh returns { session_token, ws_url, turn_credentials, run_id }
  → Your backend forwards the payload to the frontend (never expose API key)
  → Frontend calls startVoiceSession(session)
  → Browser ↔ Dograh WebRTC session is live
  → Agent speaks; user responds; STT → LLM → TTS loop runs
  → Session ends when user closes tab or agent reaches an end node
```

---

## Step 8 — Retrieve the Post-Session Transcript

After the call ends, fetch results using `run_id` from Step 7:

```bash
# Authenticated with API key
curl -H "X-API-Key: <your-org-api-key>" \
  "https://abc123.ngrok-free.app/api/v1/workflow/<workflow_id>/runs/<run_id>"
```

Key fields in the response:

| Field | Description |
|---|---|
| `transcript_url` | Link to download the conversation transcript |
| `recording_url` | Link to the audio recording (if recording is enabled) |
| `gathered_context` | Variables collected by the LLM node (scores, notes, etc.) |
| `cost` | Token and audio usage for the run |

Download a transcript:

```bash
curl -L "https://abc123.ngrok-free.app/public/download/workflow/<token>/transcript" \
  -o transcript.txt
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `401 Invalid API key` | Check the `X-API-Key` header; ensure the key belongs to the same org as the workflow |
| `404 Agent trigger not found` | Verify the `{uuid}` is the trigger node's UUID (not the workflow UUID) and the workflow is published |
| `402 Quota exceeded` | The workflow owner's account is out of credits; top up in Dograh org settings |
| WebSocket closes immediately (code 1008) | Session token expired (1-hour TTL) or the `ws_url` host is unreachable — check ngrok is running and `BACKEND_API_ENDPOINT` is set |
| No audio in browser | TURN server not reachable: verify coturn is running (`docker ps`) and `TURN_SECRET` is set in `api/.env` |
| `wss://` connection fails on Safari | Safari requires valid TLS. Use ngrok (provides HTTPS) rather than a raw IP |
| ICE gathering stalls | Set `FORCE_TURN_RELAY=true` in `api/.env` to force relay-only mode for debugging |

---

## Environment Variable Reference

| Variable | Required | Description |
|---|---|---|
| `BACKEND_API_ENDPOINT` | Yes (for remote) | Public HTTPS URL of the Dograh API (e.g. ngrok URL). Dograh uses this to build the `ws_url` returned to clients. |
| `TURN_SECRET` | Recommended | Shared secret for coturn time-limited credentials. Without it, TURN credentials are not returned and WebRTC may fail through strict NAT. |
| `TURN_HOST` | Recommended | Hostname or IP of the TURN server (default: `localhost`). |
| `TURN_PORT` | Optional | TURN server UDP/TCP port (default: `3478`). |
| `FORCE_TURN_RELAY` | Optional | Set to `true` to force relay-only ICE. Useful for debugging NAT issues. |
