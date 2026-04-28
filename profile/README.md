<div align="center">

<img src="https://github.com/xrcc-retrace/.github/raw/main/profile/assets/retrace-banner.png" alt="Retrace" width="100%" />

### Record an expert once. Coach every learner forever.

An AI coaching system that turns one expert demonstration into voice-guided, visually-verified, hands-free training for everyone else.

Built for the **XRCC Berlin 2026** hackathon.

</div>

---

## Live now

| Component | Status | Where |
| --- | --- | --- |
| Backend API | **Live** | `https://retracexrcc.duckdns.org` &nbsp; — &nbsp; AWS Lightsail (Ubuntu 22.04, `ap-southeast-1`, migrating to `eu-central-1` before the Berlin demo) |
| iOS app | **Live** | TestFlight (invite-only beta) |
| Continuous deploy | **Active** | GitHub Actions → SSH into Lightsail → `git pull` + `pip install` + `systemctl restart` + `/health` check on every push to `main` |
| TLS | **Auto-renewed** | Caddy + Let's Encrypt, fronting `localhost:8000` |

---

## What it actually does

An **expert** wearing Ray-Ban Meta glasses (or holding an iPhone) records themselves doing a hands-on task once — fixing a humidifier, replacing a fridge filter, calibrating a 3D printer. Retrace's backend feeds that video to **Gemini 2.5 Pro** which extracts a structured procedure: title, description, ordered steps, per-step *visual completion criteria*, tips, warnings. `ffmpeg` slices a reference clip for each step.

A **learner** then opens the app, picks the procedure, and starts a coaching session. Their iPhone opens a **direct WebSocket to Gemini Live** (`gemini-3.1-flash-live-preview`) — voice and ~0.5 fps video both flow straight from the phone to Google, never through our backend. Gemini sees what the learner sees, hears what they say, talks back in real time, and **calls a server-side `advance_step` function** the moment the visual completion criteria are met. The learner gets hands-free voice coaching that auto-progresses through the procedure as they work.

If the learner doesn't *have* a procedure for their problem — say their humidifier is broken and they don't know why — they can launch **Troubleshoot mode** instead. Gemini diagnoses the product on camera, searches the existing library, and if nothing matches, runs a Gemini Pro + Google Search pipeline to fetch a fresh, citation-backed procedure on the fly, then hands off into a normal coaching session.

---

## Architecture

```
┌────────────────────────┐         ┌──────────────────────┐         ┌──────────────────────┐
│  Capture transport     │         │      Phone (iOS)     │         │   Backend (FastAPI)  │
│ ─────────────────────  │         │ ──────────────────── │         │ ──────────────────── │
│  Ray-Ban Meta glasses  │ ──────► │  Capture orchestrator│ ◄─────► │  Procedure pipeline  │
│  (DAT SDK 0.6)         │         │  Gemini Live client  │  REST   │  Session control     │
│        OR              │         │  Token actor         │         │  Tool dispatcher     │
│  iPhone camera + mic   │         │  Activity feed UI    │         │  Token minting       │
│  (AVCaptureSession)    │         │  Ray-Ban HUD UI      │         │  Bonjour discovery   │
└────────────────────────┘         └──────────┬───────────┘         └──────────┬───────────┘
                                              │                                │
                                              │ direct WebSocket               │ Files API
                                              │ (BidiGenerateContent           │ + Pro REST
                                              │  Constrained, v1alpha)         │
                                              ▼                                ▼
                                   ┌──────────────────────────────────────────────────────┐
                                   │                  Gemini API (Google)                 │
                                   │     Live (3.1 Flash Live) · 2.5 Pro · 2.5 Flash      │
                                   └──────────────────────────────────────────────────────┘
```

**Why voice and video go direct, phone → Gemini:** routing them through the backend would add 100–200 ms per exchange and break conversational feel. The backend is strictly the control plane — procedure storage, prompt assembly, tool execution, ephemeral token minting, session resumption.

---

## The three flows

| Flow | Trigger | What the backend does | What the phone does |
| --- | --- | --- | --- |
| **Expert** | Multipart video or PDF manual upload | Probes duration with ffprobe → uploads to Gemini Files API → Gemini 2.5 Pro returns structured JSON with per-step `completion_criteria` → ffmpeg cuts per-step clips → SQLite | Records via glasses or iPhone, polls for procedure status, edits titles/tips/warnings |
| **Learner** | `POST /api/learner/session/start` | Builds a **current-step-only** system prompt, mints an ephemeral token bound to model + prompt + tools + voice + compression + resumption | Opens direct Gemini Live WebSocket, streams audio + JPEG frames, intercepts `advance_step` / `get_reference_clip` / `go_to_step` tool calls and forwards them to the server |
| **Troubleshoot** | `POST /api/troubleshoot/session/start` (no procedure required) | Same minting flow with a different prompt + tool set; on `web_search_for_fix`, runs a two-step Gemini Pro pipeline (GoogleSearch grounding → structured-output JSON) and returns a fresh procedure with cited URLs | Drives `identify_product` → `confirm_identification` → `search_procedures` → optional `web_search_for_fix` → `handoff_to_learner` to spawn a coaching session |

---

## Feature showcase

### Capture & ingest

| | |
| --- | --- |
| **Two capture transports** | Ray-Ban Meta glasses via Meta's DAT SDK 0.6, **or** iPhone camera via AVCaptureSession. Picked per-session, transport-agnostic from the coaching pipeline's POV. |
| **Video → SOP** | Gemini 2.5 Pro structured-output extraction, with timestamp clamping against the true ffprobe-measured duration so Gemini-imagined ranges can't break ffmpeg. |
| **PDF → SOP** | Manual ingest pipeline: PDF → Gemini Files API → structured procedure JSON → per-step source-page rasterization. Manual-sourced procedures plug into the same clip-serving endpoints as video-sourced ones. |
| **Per-step clips** | H.264+AAC, faststart, BT.709 color tags for clean iOS VideoToolbox playback. Per-step ffmpeg failures are tolerated — the procedure lands `completed_partial` instead of failing the whole upload. |
| **Iconify icon library** | Any `iconify` ID becomes a white-on-transparent PNG via cairosvg, disk-cached, served at `GET /{prefix}/{name}.png` so the iOS HUD references icons without bundling assets. |

### Real-time coaching

| | |
| --- | --- |
| **Direct Gemini Live socket** | Phone ↔ Gemini WebSocket using ephemeral, single-use, server-locked auth tokens. The Gemini API key never leaves the backend. |
| **Visual completion detection** | Gemini Live watches the learner's camera feed and fires `advance_step` the moment the step's `completion_criteria` are observed. Auto-advance tags completions as `method="visual"`; manual mode is `"verbal"`. |
| **Eight Gemini Live voices** | Puck, Charon, Kore, Fenrir, Aoede, Zephyr, Enceladus, Leda — server-authoritative, with pre-rendered WAV previews. |
| **Picture-in-picture reference clip** | Coaching UI overlays the current step's expert clip on top of the live camera feed when the learner asks "show me." |
| **Live transcripts** | Both input (learner) and output (AI) audio transcribed by Gemini, rendered into the iOS activity feed alongside tool calls and step transitions. |
| **Barge-in** | Server-side VAD detects the learner talking over the AI; iOS flushes the playback buffer so the open-ear glasses speakers don't stomp the next turn. |
| **Indefinite session length** | Context window compression (trigger 100k → target 40k) plus session resumption via `sessionResumptionUpdate.new_handle`. The "2-minute audio+video" Gemini limit is a context-fill limit, not a hard timer. |
| **Reconnect & re-orient** | On `goAway` the phone mints a handle-baked token, reopens the socket, then injects the server's compact `context-summary` as a user turn so the model wakes up on the real current step instead of step 1. |

### Diagnostic Troubleshoot mode

| | |
| --- | --- |
| **No procedure required at start** | User describes a broken product; Gemini takes it from there. |
| **Phase machine** | `identify → diagnose → fetching-manual → ready-to-coach`, surfaced in the iOS UI as `DiagnosticPhaseBar`. |
| **Library search** | `search_procedures` runs a Gemini 2.5 Flash-ranked search over the existing procedure library by product + symptom. No separate full-text index — fine at hackathon scale. |
| **On-the-fly procedure synthesis** | When nothing matches, `web_search_for_fix` runs a two-step Gemini Pro pipeline: GoogleSearch grounding for a research brief, then structured-output for a final procedure JSON with cited source URLs. |
| **Handoff to coaching** | `handoff_to_learner` promotes the chosen procedure into a normal learner session and returns the full session payload so the iOS client opens a coaching socket without a second round-trip. |
| **Bundled fixture** | A Levoit LV600HH humidifier manual ships with the backend and ingests on first start so the demo library is never empty. |

### Ray-Ban HUD design system

| | |
| --- | --- |
| **Square 600 × 600 viewport** | Hard-clipped lens canvas, centered placement — matches what will eventually render on the actual glasses HUD. |
| **Glass-panel surface recipe** | Standardized blur, tint, stroke, and shadow tokens for every overlay. |
| **Hover-then-select interaction** | Driven by the `HandTracking/` substrate (Apple Vision pose extraction → micro-gesture + pinch-drag recognizers → `HandGestureService` state machine). |
| **Spring-token animation vocabulary** | Single shared transition token for every overlay; Core-Animation-driven continuous progress (no `@State` polling timers). |
| **First-tap-highlights for destructive actions** | Default-focus rules built into the focus graph; double-tap-to-dismiss; hold-to-confirm. |
| **Recede-and-arrive overlay pattern** | Canonical animation for any modal that takes over the lens. Documented with worked examples in `Views/RayBanHUD/DESIGN.md`. |

### Backend & infra

| | |
| --- | --- |
| **FastAPI control plane** | Async upload pipelines, in-memory session stores (learner + troubleshoot), shared tool dispatcher, Pydantic models everywhere. |
| **SQLite WAL** | `procedures` + `steps` tables with ALTER-TABLE migrations on boot. JSON-encoded `tips` / `warnings`. |
| **Bonjour LAN discovery** | `_retrace._tcp.local.` advertised on the same Wi-Fi; iOS auto-discovers and caches the URL. Manual override available in Server Settings. |
| **AWS Lightsail production** | $12/mo Ubuntu VM, static IP, 2 GB swap, Caddy + Let's Encrypt TLS in front of `uvicorn` on `127.0.0.1:8000`, systemd unit `retrace.service`. |
| **GitHub Actions CI/CD** | Push to `main` → SSH deploy → `git pull` + `pip install` + `systemctl restart` + `/health` gate. ~20-second deploys. |
| **Probe scripts** | `probe_*.py` family: which Live models this API key accepts, which `LiveConnectConfig` fields the current model rejects, end-to-end production-config verification, tool-call HIT/MISS rate measurement, system-prompt honoring canary. |
| **Interactive prompt playground** | `python -m gemini_live_playground` — REPL against the same model + system prompt + tool definitions + compression + voice as production. Tool calls dispatch in-process. No FastAPI needed. |

---

## Tech stack

| Layer | Pick | Why |
| --- | --- | --- |
| Speech + vision model | **Gemini 3.1 Flash Live** (preview) | Audio in, audio out, video in, function calling, context compression, session resumption — all one socket. |
| Procedure extraction | **Gemini 2.5 Pro** with structured-output JSON schema | Turns a long video into a clean stepwise SOP in one shot; falls back to 2.5 Flash on 503. |
| Web research | **Gemini 2.5 Pro** + GoogleSearch grounding | Returns a structured procedure with cited URLs — replaced the brittle PDF-fetch flow we tried first. |
| Backend | **Python 3.11 / FastAPI / SQLite / ffmpeg** | Cheap to host, fast to iterate, no schema migration tooling needed at hackathon scale. |
| iOS | **SwiftUI / Combine / Swift Concurrency** | `actor`s for token management, `@MainActor` view models, `AsyncSequence` for DAT SDK streams. |
| Smart glasses | **Meta Ray-Ban via DAT SDK 0.6** (`MWDATCore` + `MWDATCamera`) | First-class Apple-style API; `MockDeviceKit` lets us develop without hardware. |
| TLS + reverse proxy | **Caddy** | Three lines of config; auto-provisions and auto-renews Let's Encrypt certs. |
| Hosting | **AWS Lightsail** | VPS with training wheels; flat $12/mo; 5-min snapshot-and-relaunch in any region. |
| DNS | **DuckDNS** | Free subdomain pointing at the static IP; zero overhead. |
| CI/CD | **GitHub Actions** | One workflow file, three secrets, every push deploys. |

---

## Repositories

| Repo | What's in it |
| --- | --- |
| [`main-local-server-test`](https://github.com/xrcc-retrace/main-local-server-test) | FastAPI backend, Gemini integration, expert/learner/troubleshoot pipelines, ephemeral token minting, Bonjour, manual ingest, deployment workflow. |
| [`dat-ios-test`](https://github.com/xrcc-retrace/dat-ios-test) | SwiftUI iOS client. Forked from Meta's `CameraAccess` DAT SDK sample, extended into the full Retrace phone app — capture, coaching, troubleshoot, Ray-Ban HUD. |

---

## Critical architecture decisions

A short list of the calls that shape everything else:

1. **Voice + video go direct** to Gemini Live — never through the backend. The 100–200 ms server hop kills conversational feel.
2. **Visual verification runs inline** in the same Gemini Live session that's already speaking — not as a separate clip-to-clip pass.
3. **Audio + video share one Live session.** Compression + resumption unlock indefinite duration; the documented "2-minute" cap is a context-fill limit, not a timer.
4. **Current-step-only system prompt.** Future steps reach the model exclusively through `advance_step` tool responses, which prevents narration-ahead and forces the tool call.
5. **Ephemeral tokens, not API keys.** The Gemini API key never leaves the backend. Tokens are locked to a specific model + prompt + tools + voice + compression + resumption handle, so a leaked token can't be repurposed.
6. **Two capture transports, one coaching pipeline.** The coaching VM, audio session manager, and Live client are transport-agnostic — they see "a JPEG source" and "a mic buffer source," nothing more.
7. **Bonjour for discovery, Lightsail for fallback.** LAN-first when both devices are on the same Wi-Fi; cloud when they're not.

---

<div align="center">

Built by the XRCC Retrace team for the Berlin 2026 hackathon.

</div>
