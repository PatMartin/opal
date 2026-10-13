# OPAL Suite User Guide

**OPAL** is a modular, local-first platform for building voice- and vision-enabled intelligent assistants
on Linux and edge hardware (such as the Raspberry Pi 5).

Instead of bundling speech recognition, text-to-speech, computer vision, and reasoning into a monolithic
daemon or relying on cloud APIs, OPAL decomposes an assistant into five specialized, loosely coupled
services connected by an asynchronous event bus:

| Service | Sensory / Cognitive Role | Default Port | Core Technology |
|---|---|---|---|
| [**OPAL Listen**](#opal-listen--hearing-and-speech-understanding) | Ears (Wakeword, STT, Intent, Slots) | `8000` | Vosk, optional faster-whisper, SQLite |
| [**OPAL Talk**](#opal-talk--voice-and-speech-synthesis) | Voice (TTS, progressive PCM streaming) | `8001` | Piper, Kokoro-ONNX, SQLite |
| [**OPAL Event**](#opal-event--nervous-system-and-action-dispatcher) | Nervous System (Event bus, Rules, Actions) | `8080` (ZMQ `5555`/`5556`) | ZeroMQ XSUB/XPUB, PyZMQ, SQLite |
| [**OPAL Look**](#opal-look--vision-and-scene-understanding-contract) | Eyes (Vision, Detection, OCR, MJPEG) | *(Contract v1.3.0)* | Neutral vision engine adapter |
| [**OPAL Think**](#opal-think--reasoning-agents-and-memory-contract) | Brain (LLM inference, Agents, Memory) | *(Contract v1.3.0)* | Neutral agent/inference runtime |

Every service provides full functional parity across three interfaces:
1. **REST API**: Authoritative OpenAPI 3.1 specification, with interactive documentation at `/docs`.
2. **Unified CLI**: Terminal management and script automation (`opal <service> <command>`).
3. **Web GUI**: Responsive single-page applications for interactive configuration, monitoring, and testing.

---

## Table of contents

1. [Concepts](#1-concepts)
2. [Installation](#2-installation)
3. [Architecture](#3-architecture)
4. [Configuration reference](#4-configuration-reference)
5. [REST API reference](#5-rest-api-reference)
6. [Transports, streaming, and event bus](#6-transports-streaming-and-event-bus)
7. [CLI reference](#7-cli-reference)
8. [Web GUI guide](#8-web-gui-guide)
9. [Recipes and workflows](#9-recipes-and-workflows)
10. [Integrating with your application](#10-integrating-with-your-application)
11. [Testing and verification](#11-testing-and-verification)
12. [Troubleshooting and FAQ](#12-troubleshooting-and-faq)
13. [Appendix: operation ↔ CLI ↔ GUI map](#13-appendix-operation--cli--gui-map)

---

## 1. Concepts

### The local-first design philosophy

Modern voice assistants typically stream raw audio and video to centralized cloud servers. OPAL is
built on the opposite premise:

- **Zero cloud lock-in:** All speech recognition, text-to-speech synthesis, vision inference, and event routing execute on your local hardware.
- **Privacy by default:** Audio and video streams never leave the host. Media files and snapshots have short, explicit retention windows (uploaded media expires after 24 hours).
- **Graceful offline operation:** Network outages do not degrade local home-automation, wakeword detection, or intent execution.
- **Hardware-friendly resource bounds:** Worker queues, ring buffers, and log tables are strictly bounded to prevent out-of-memory panics on constrained devices like the Raspberry Pi 5.

### Service responsibilities

#### OPAL Listen: hearing and speech understanding
`opal-listen` captures live microphone audio, monitors for configurable wakewords, runs grammar-constrained
speech-to-text (using Vosk or faster-whisper), normalizes utterances, extracts slot values, and resolves
intents. When a voice command matches an intent, it persists a recognition log and publishes an event
to the ZeroMQ bus (`opal.listen.intent.match`). Unmatched utterances are recorded as *failed utterances*
for iterative training phrase promotion.

#### OPAL Talk: voice and speech synthesis
`opal-talk` provides an engine-neutral text-to-speech layer over host-installed synthesis engines like
Piper and Kokoro. It manages voice registrations, speech profiles (rate, volume), text chunking, and
audio encoding. It outputs completed WAV/PCM files or streams live raw PCM (`s16le`) progressively,
allowing audio playback to begin immediately while subsequent sentences are being generated.

#### OPAL Event: nervous system and action dispatcher
`opal-event` is the central message router and orchestration engine. It runs an in-process ZeroMQ
XSUB/XPUB broker and accepts HTTP event publications. It evaluates incoming events against declarative
rules (topic matching and typed boolean conditions) and dispatches actions across a bounded worker pool.
Action handlers include non-interpolated `shell` commands, outbound `http` webhooks (protected by DNS
allowlists), and isolated `python` plugins.

#### OPAL Look: vision and scene understanding (contract)
`opal-look` defines an implementation-neutral vision contract covering object detection, image
classification, instance/semantic segmentation, pose estimation, and OCR. It supports USB/Pi cameras,
RTSP feeds, and uploaded media, streaming structured detection results over Server-Sent Events (SSE)
and live video over MJPEG.

#### OPAL Think: reasoning, agents, and memory (contract)
`opal-think` defines a contract for multi-turn conversational agents, model inference, tool execution,
and long-term memory. It isolates model reasoning behind strict tool authorization policies, routing
external side effects through `opal-event`.

### The OPAL Suite Standard (v1.2 & v1.3)

All five services adhere to the normative OPAL Suite Standard:

1. **Correlation headers:** Every request accepts and echoes four tracing headers:
   - `X-Opal-Session-Id`: Identifies the logical conversational session.
   - `X-Opal-Transaction-Id`: Correlates distributed operations across services.
   - `X-Opal-Request-Id`: Unique per HTTP request/response exchange.
   - `X-Opal-Causation-Id`: Identifies the parent event or triggering request.
2. **Optimistic concurrency:** Mutable configuration endpoints require strong ETags via `If-Match`. Stale updates return `HTTP 412`; missing preconditions return `HTTP 428`.
3. **Uniform error envelope:** Failures return a structured JSON body with a machine-readable `error_code`, human-readable `message`, and contextual `details`.
4. **Standard operational routes:** Every service exposes `/api/v1/health/`, `/health/live`, `/health/ready`, `/api/v1/settings/`, and `/api/v1/backup/` endpoints.
5. **Deterministic pagination:** Configuration catalogs are unpaginated; event and log histories use opaque cursor pagination (`items`, `next_cursor`, `has_more`).

---

## 2. Installation

### System prerequisites

OPAL is supported on Linux (x86_64 and aarch64, including Raspberry Pi OS / Debian 12) and macOS.

#### Required system packages (Debian / Ubuntu / Raspberry Pi OS)

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv build-essential \
                    libasound2-dev portaudio19-dev sox ffmpeg libzmq3-dev
```

#### Node.js (for Web SPAs)
Node.js ≥ 18 and npm are required to build or develop the Web SPAs:

```bash
# Node.js LTS via NodeSource or nvm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

---

### Step-by-step installation

We recommend setting up all services in a shared development environment.

#### 1. Clone the repository and create a virtual environment

```bash
git clone https://github.com/opal/opal.git /gh/opal
cd /gh/opal
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

#### 2. Install the services in editable mode

```bash
# 1. Install OPAL Event (event bus, rules, action executor)
pip install -e "./opal-event[dev]"

# 2. Install OPAL Listen (voice engine, Vosk STT)
pip install -e "./opal-listen[dev]"

# 3. Install OPAL Talk (text-to-speech core, Piper, Kokoro adapters)
pip install -e "./opal-talk[engines,dev]"
```

#### 3. Download models and voice assets

##### Vosk STT Model for OPAL Listen:
```bash
mkdir -p /gh/opal/models
cd /gh/opal/models
wget https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip
export OPAL_LISTENER_VOSK_MODEL_PATH=/gh/opal/models/vosk-model-small-en-us-0.15
```

##### Piper Voice for OPAL Talk:
```bash
mkdir -p /gh/opal/voices/piper
cd /gh/opal/voices/piper
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx.json
```

#### 4. Build the Web SPAs

```bash
# Build OPAL Event SPA
cd /gh/opal/opal-event/spa
npm install
npm run build

# Build OPAL Talk SPA
cd /gh/opal/opal-talk/spa
npm install
npm run build
```

---

### Running services for local development

Open separate terminal tabs or use a process manager (such as `tmux` or `foreman`):

```bash
# Terminal 1: Start OPAL Event (Port 8080, ZMQ XSUB 5555, XPUB 5556)
cd /gh/opal/opal-event
opal-event-svc --config config/opal_event.example.yaml

# Terminal 2: Start OPAL Talk (Port 8001)
cd /gh/opal/opal-talk
opal-talk-svc

# Terminal 3: Start OPAL Listen (Port 8000)
cd /gh/opal/opal-listen
python -m opal_listener
```

---

### Production deployment with systemd

For persistent background execution on a Raspberry Pi or Linux server, create systemd unit files:

#### `/etc/systemd/system/opal-event.service`
```ini
[Unit]
Description=OPAL Event Service
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/gh/opal/opal-event
Environment=PATH=/gh/opal/.venv/bin:/usr/bin:/bin
ExecStart=/gh/opal/.venv/bin/opal-event-svc --config /gh/opal/opal-event/config/opal_event.yaml
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

#### `/etc/systemd/system/opal-talk.service`
```ini
[Unit]
Description=OPAL Talk TTS Service
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/gh/opal/opal-talk
Environment=PATH=/gh/opal/.venv/bin:/usr/bin:/bin
Environment=OPAL_TALK_HOST=127.0.0.1
Environment=OPAL_TALK_PORT=8001
ExecStart=/gh/opal/.venv/bin/opal-talk-svc
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

#### `/etc/systemd/system/opal-listen.service`
```ini
[Unit]
Description=OPAL Listen Service
After=network.target sound.target opal-event.service

[Service]
Type=simple
User=pi
WorkingDirectory=/gh/opal/opal-listen
Environment=PATH=/gh/opal/.venv/bin:/usr/bin:/bin
Environment=OPAL_LISTENER_VOSK_MODEL_PATH=/gh/opal/models/vosk-model-small-en-us-0.15
Environment=OPAL_EVENT_XPUB_URL=tcp://127.0.0.1:5555
ExecStart=/gh/opal/.venv/bin/python -m opal_listener
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start the units:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opal-event opal-talk opal-listen
```

---

### Docker Compose layout

To run OPAL in containers, configure `docker-compose.yml`:

```yaml
version: "3.8"

services:
  opal-event:
    build:
      context: ./opal-event
    container_name: opal-event
    network_mode: host
    volumes:
      - ./opal-event/config:/app/config
      - opal-event-data:/app/var
    restart: unless-stopped

  opal-talk:
    build:
      context: ./opal-talk
    container_name: opal-talk
    network_mode: host
    volumes:
      - ./voices:/voices:ro
      - opal-talk-data:/app/var
    restart: unless-stopped

  opal-listen:
    build:
      context: ./opal-listen
    container_name: opal-listen
    network_mode: host
    devices:
      - /dev/snd:/dev/snd
    volumes:
      - ./models:/models:ro
      - opal-listen-data:/app/var
    environment:
      - OPAL_LISTENER_VOSK_MODEL_PATH=/models/vosk-model-small-en-us-0.15
      - OPAL_EVENT_XPUB_URL=tcp://127.0.0.1:5555
    restart: unless-stopped

volumes:
  opal-event-data:
  opal-talk-data:
  opal-listen-data:
```

---

## 3. Architecture

### System architecture diagram

The diagram below illustrates how external sound and visual inputs flow through the OPAL pipeline:

```
  ┌────────────────┐         ┌────────────────┐         ┌────────────────┐
  │ Microphones /  │         │ Cameras / Video│         │ Web Clients /  │
  │ Audio Inputs   │         │ Feeds / RTSP   │         │ Third-Party API│
  └───────┬────────┘         └───────┬────────┘         └───────┬────────┘
          │                          │                          │
          ▼                          ▼                          ▼
   ┌──────────────┐           ┌──────────────┐           ┌──────────────┐
   │ OPAL Listen  │           │  OPAL Look   │           │ User Scripts │
   │ (Port 8000)  │           │ (Port 8002*) │           │    & Apps    │
   └──────┬───────┘           └──────┬───────┘           └──────┬───────┘
          │ (Intent Event)           │ (Detection Event)        │ (HTTP Ingress)
          │                          │                          │
          └──────────────────┐       │       ┌──────────────────┘
                             ▼       ▼       ▼
                     ┌───────────────────────────────┐
                     │          OPAL Event           │
                     │  (Port 8080 / ZMQ 5555, 5556) │
                     │  - In-process XSUB/XPUB broker │
                     │  - Rule Engine (AND filters)  │
                     │  - Bounded Action Executor    │
                     └───────────────┬───────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
   ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
   │  OPAL Talk   │          │  OPAL Think  │          │ External     │
   │ (Port 8001)  │          │ (Port 8003*) │          │ Webhooks /   │
   │ - TTS Piper/ │          │ - Agent Runs │          │ Shell Scripts│
   │   Kokoro     │          │ - Tool Policy│          └──────────────┘
   │ - Audio Play │          └──────────────┘
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │ Local ALSA / │
   │ Speakers     │
   └──────────────┘

* Planned service contract ports
```

### Port and network protocol map

| Service | Protocol | Default Port / Address | Responsibility |
|---|---|---|---|
| **OPAL Listen** | HTTP / REST | `http://127.0.0.1:8000` | REST API, SSE live results, Web GUI |
| | ZeroMQ PUB | `tcp://127.0.0.1:5559` | Egress intent events (`opal.listen.intent.match`) |
| **OPAL Talk** | HTTP / REST | `http://127.0.0.1:8001` | REST API, `/ui` SPA, progressive PCM streaming |
| **OPAL Event** | HTTP / REST | `http://127.0.0.1:8080` | REST API, event ingress, management |
| | ZeroMQ XSUB | `tcp://127.0.0.1:5555` | Inbound event publishers connect here |
| | ZeroMQ XPUB | `tcp://127.0.0.1:5556` | Outbound event subscribers connect here |
| **Vite Dev SPAs**| HTTP | `http://127.0.0.1:5173` | React frontend development servers |

### Persistence and data storage

All persistence in OPAL is local and file-backed:

- **SQLite WAL Mode:** Each service maintains its own isolated SQLite database in `./var/` with Write-Ahead Logging (WAL) enabled:
  - `opal-listen`: `var/opal_listener.db` (intents, phrases, slots, profiles, recognition logs, failed utterances).
  - `opal-talk`: `var/opal_talk.db` (engine configs, voices, profiles, service settings).
  - `opal-event`: `var/opal_event.db` (observed events, rules, actions, execution records).
- **Atomic Configuration Files:** Service YAML files represent authoritative configuration. Live changes apply atomically in memory and write out safely to disk.

---

## 4. Configuration reference

OPAL separates configuration into two distinct categories:

1. **Portable service settings:** High-level behaviors (e.g. cooldown times, thresholds, default speeds, max text lengths) stored in SQLite and adjustable live via `PATCH /api/v1/settings/`.
2. **Infrastructure settings:** Hardware, network bindings, and directory paths stored in YAML configuration files and environment variables.

### Configuration file layouts

#### OPAL Listen: `opal_listener/config/`
- `settings.yaml`: Cooldown intervals, deduplication windows, fuzzy matching thresholds, audio watchdog timeouts.
- `stt.yaml`: Speech engine provider (`vosk`), model paths, input sample rate, VAD energy threshold.
- `intents.yaml`: Pre-seeded intent definitions and example training phrases.
- `slots.yaml`: Slot types, canonical values, and synonyms/aliases.
- `wakewords.yaml`: Wakeword phrases, detection thresholds, and acoustic distractor lists.

#### OPAL Talk: `config/service.yaml`
```yaml
host: "127.0.0.1"
port: 8001
data_dir: "./var"
config_dir: "./config"
cors_origins: ["*"]
engines:
  piper:
    voices_dir: "/gh/opal/voices/piper"
  kokoro:
    model_path: "/gh/opal/voices/kokoro/kokoro-v1.0.onnx"
    voices_path: "/gh/opal/voices/kokoro/voices-v1.0.bin"
```

#### OPAL Event: `config/opal_event.yaml`
```yaml
server:
  host: "127.0.0.1"
  port: 8080
broker:
  publishers: "tcp://127.0.0.1:5555"
  subscribers: "tcp://127.0.0.1:5556"
security:
  http_allowlist:
    - "127.0.0.1"
    - "localhost"
executor:
  workers: 4
  queue_size: 100
persistence:
  database_path: "var/opal_event.db"
  event_mode: "matched"
```

### Live hot-reloading vs restart-required settings

| Change Target | Requires Restart? | Mechanism |
|---|---|---|
| Rules & actions (`opal-event`) | **No** | In-memory atomic rule update; live filesystem watcher |
| Portable settings (`/settings/`) | **No** | Atomic `PATCH /api/v1/settings/` applied to running state |
| Profiles / voices / intents / slots | **No** | Transactional `config_change()` guard and snapshot swap |
| Network bind host & port | **Yes** | Server process must restart to rebind TCP sockets |
| ZeroMQ endpoints & CURVE keys | **Yes** | Sockets must be recreated and re-bound |
| SQLite database file paths | **Yes** | Database connections must be reopened |

---

## 5. REST API reference

All OPAL services expose standard REST endpoints under the `/api/v1` namespace.

### Standard request context headers

Every inbound HTTP request accepts these optional headers, which are validated, logged, and echoed in the response:

```http
X-Opal-Session-Id: sess-9a2f-41b8
X-Opal-Transaction-Id: tx-5b7c-11e2
X-Opal-Request-Id: req-88c1-00a4
X-Opal-Causation-Id: evt-12f8-99e3
```

### Standard response envelope and error format

Successful responses return JSON or raw media streams. When an error occurs, the response body adheres to the `OpalError` schema:

```json
{
  "error_code": "precondition_failed",
  "message": "Resource ETag does not match supplied If-Match header.",
  "details": {
    "current_etag": "\"2-8fa41b9c\"",
    "provided_etag": "\"1-0e73d22a\""
  }
}
```

### Common HTTP status codes

- `200 OK`: Request succeeded; body contains resource data.
- `201 Created`: Resource created synchronously; `Location` header points to new URI.
- `202 Accepted`: Asynchronous operation queued or accepted (e.g. event published to broker).
- `204 No Content`: Resource deleted successfully; empty body.
- `412 Precondition Failed`: ETag validation failed on mutable update.
- `422 Unprocessable Entity`: Schema validation error or invalid parameter types.
- `428 Precondition Required`: Mutation attempted without required `If-Match` header.

### Endpoints overview

Interactive OpenAPI documentation and live Swagger UI are available on every running service:
- OPAL Listen: `http://localhost:8000/docs`
- OPAL Talk: `http://localhost:8001/docs`
- OPAL Event: `http://localhost:8080/docs`

---

## 6. Transports, streaming, and event bus

### The ZeroMQ event bus

OPAL uses ZeroMQ for high-throughput, low-latency inter-service communication.

#### The 2-Frame wire protocol
All messages on the OPAL ZeroMQ bus use exactly two frames:

```
┌─────────────────────────────────┐
│ Frame 1: Topic (UTF-8 string)   │
│ e.g. "opal.listen.intent.match" │
├─────────────────────────────────┤
│ Frame 2: JSON Event Envelope    │
│ e.g. {"id": "...", "data": ...} │
└─────────────────────────────────┘
```

#### JSON event envelope format

```json
{
  "id": "c1f7a4d8-4903-4e38-9226-6a56b6b7979f",
  "topic": "opal.listen.intent.match",
  "source": "opal-listener",
  "timestamp": "2026-10-12T19:42:01.120Z",
  "metadata": {
    "session_id": "sess-user-1",
    "transaction_id": "tx-voice-turn-4",
    "schema_version": "1.2"
  },
  "data": {
    "wakeword": "computer",
    "transcript": "turn on the kitchen light",
    "intent_name": "light_turn_on",
    "slot_values": {
      "room_name": "kitchen"
    },
    "intent_confidence": 1.0,
    "matched": true
  }
}
```

### Server-Sent Events (SSE)

Long-running streams use standard `text/event-stream` MIME types:

- **OPAL Listen (`GET /api/v1/recognition/stream`):** Streams partial transcripts, VAD speech-start/end markers, and finalized intent matches.
- **OPAL Look (`GET /api/v1/sessions/{id}/events`):** Streams bounding boxes, object labels, and tracking IDs for active camera sessions.
- **OPAL Think (`GET /api/v1/runs/{id}/events`):** Streams LLM generation tokens, tool invocation steps, and execution statuses.

### Progressive raw PCM audio streaming

`OPAL Talk` implements true progressive audio streaming via `POST /api/v1/generate/stream`:

```
Client                                  OPAL Talk
  │                                         │
  ├─── POST /api/v1/generate/stream ───────▶│ (Text chunking)
  │    {"text": "Sentence 1. Sentence 2."}  │
  │                                         │ (Synthesize chunk 1)
  │◀── HTTP 200 OK (Transfer-Encoding: chunked)
  │    X-Opal-Audio-Format: pcm             │
  │    X-Opal-Sample-Rate: 22050            │
  │                                         │
  │◀── [Raw PCM bytes: Chunk 1] ────────────┤ (Playback begins immediately)
  │                                         │ (Synthesize chunk 2 in background)
  │◀── [Raw PCM bytes: Chunk 2] ────────────┤
  │                                         │
  └─── Stream closed ───────────────────────┘
```

The progressive audio format is signed 16-bit little-endian (`s16le`), mono.

---

## 7. CLI reference

The unified `opal` CLI provides complete management over all services.

```bash
# General syntax:
opal <service> [global-options] <command-group> <command> [args]
```

### Key commands quick lookup

```bash
# Check status across all services
opal listen system doctor
opal talk health
opal event health

# Manage voice recognition
opal listen wakewords list
opal listen intents list
opal listen test text "turn off the desk lamp"

# Manage speech synthesis
opal talk voices list
opal talk generate "Speech output is operational." -o output.wav

# Manage event bus and rules
opal event topics
opal event rules list
opal event publish system.alert --data-json '{"level": "warning"}'
```

*(For the complete command syntax, argument flags, and scripting options, consult the authoritative [docs/CLI.md](CLI.md).)*

---

## 8. Web GUI guide

OPAL includes dedicated browser interfaces for visual administration, testing, and real-time monitoring.

### 1. OPAL Listen Web GUI
Served by the `opal_listener` process at `http://localhost:8000/ui/opal-listen.html`:
- **Microphone tester:** Live audio visualizer showing sound levels, energy VAD triggers, and real-time transcription.
- **Intent & slot manager:** Form-based editor to define training phrases and add slot aliases.
- **Failure review console:** View unmatched voice utterances with one-click promotion into training phrases.

### 2. OPAL Talk React SPA
Served at `http://localhost:8001/ui/` (or via Vite dev server at `http://localhost:5173`):
- **Voice catalog:** Audition host voices and view synthesis capabilities.
- **Profile configurator:** Adjust global speed and volume presets.
- **Interactive synthesis playground:** Type test text, preview generated WAV waveforms, and test streaming audio.

### 3. OPAL Event React SPA
Located in `opal-event/spa/`:
- **Live event monitor:** Real-time feed of events flowing through the ZeroMQ broker.
- **Visual rule editor:** Construct topic patterns and boolean conditions with instant validation.
- **Action execution auditor:** Inspect recent shell and HTTP executions, exit codes, and stdout/stderr output.

---

## 9. Recipes and workflows

### Recipe 1: Voice-activated home automation

**Objective:** Configure OPAL Listen to hear a voice command, route the intent through OPAL Event, and speak a confirmation using OPAL Talk.

#### Step 1: Configure the intent in OPAL Listen

```bash
opal listen intents add light_turn_on --description "Turn on a room light"
opal listen intents add-phrase light_turn_on "turn on the {room} light"
opal listen intents add-phrase light_turn_on "switch on the {room} light"

opal listen slots add room
opal listen slots add-value room kitchen
opal listen slots add-value room living_room
```

#### Step 2: Create a shell action in OPAL Event

```bash
opal event actions create --json '{
  "id": "activate_relay",
  "handler": "shell",
  "config": {
    "command": ["/usr/local/bin/toggle_relay.sh", "{{data.slot_values.room_name}}", "on"],
    "timeout_sec": 5
  }
}'
```

#### Step 3: Create a speech response action

```bash
opal event actions create --json '{
  "id": "say_confirmation",
  "handler": "shell",
  "config": {
    "command": ["opal", "talk", "generate", "I turned on the {{data.slot_values.room_name}} light.", "--stream", "-o", "-"],
    "timeout_sec": 10
  }
}'
```

#### Step 4: Link the intent to the actions via a rule

```bash
opal event rules create --json '{
  "id": "rule_light_on",
  "topic": "opal.listen.intent.match",
  "enabled": true,
  "conditions": [
    {"field": "data.intent_name", "operator": "eq", "value": "light_turn_on"}
  ],
  "actions": ["activate_relay", "say_confirmation"]
}'
```

---

### Recipe 2: Iterative training with failed utterances

**Objective:** Inspect utterances that failed recognition in daily use and promote them into active phrases without service downtime.

```bash
# 1. Check for failed utterances
opal listen failed list

# 2. Inspect a failed item
opal listen failed get 7
# Shows: transcript = "illuminate the kitchen"

# 3. Promote it to the light_turn_on intent
opal listen failed promote 7 --intent light_turn_on

# 4. Verify that the phrase works
opal listen test text "illuminate the kitchen"

# 5. Run the regression test suite to ensure no other intents were degraded
opal listen test regression
```

---

## 10. Integrating with your application

### Python application integration

Use standard libraries (`httpx` and `pyzmq`) to interact with OPAL:

```python
import httpx
import zmq
import json

# 1. Call OPAL Talk to generate audio
def speak_phrase(text: str, output_path: str):
    url = "http://127.0.0.1:8001/api/v1/generate"
    response = httpx.post(url, json={"text": text})
    response.raise_for_status()
    with open(output_path, "wb") as f:
        f.write(response.content)

# 2. Subscribe to OPAL Event ZeroMQ bus
def listen_for_events():
    context = zmq.Context()
    sub_socket = context.socket(zmq.SUB)
    sub_socket.connect("tcp://127.0.0.1:5556")
    sub_socket.setsockopt_string(zmq.SUBSCRIBE, "opal.listen.intent")

    print("Listening for intent events...")
    while True:
        topic = sub_socket.recv_string()
        payload = sub_socket.recv_json()
        print(f"[{topic}] Recognized intent: {payload['data']['intent_name']}")

if __name__ == "__main__":
    speak_phrase("Python client connected.", "ready.wav")
    listen_for_events()
```

### TypeScript / Node.js integration

```typescript
import axios from 'axios';

async function generateSpeech(text: string): Promise<Buffer> {
  const response = await axios.post(
    'http://127.0.0.1:8001/api/v1/generate',
    { text, output_format: 'wav' },
    { responseType: 'arraybuffer' }
  );
  return Buffer.from(response.data);
}
```

---

## 11. Testing and verification

OPAL includes comprehensive test suites across all packages:

```bash
# 1. Test OPAL Listen (unit tests, NLU compiler, audio pipelines)
cd /gh/opal/opal-listen
pytest

# 2. Test OPAL Talk (contract parity, synthesis adapters, streaming)
cd /gh/opal/opal-talk
make test-fast

# 3. Test OPAL Event (ZeroMQ broker, rule evaluation, action runners)
cd /gh/opal/opal-event
pytest
```

---

## 12. Troubleshooting and FAQ

### 1. ZeroMQ "Slow Joiner" syndrome
**Issue:** Events published immediately after connecting a new publisher or subscriber are occasionally missed.  
**Explanation:** ZeroMQ connections are asynchronous. The TCP handshake and subscription exchange take several milliseconds to complete.  
**Solution:** Connect sockets during service startup rather than on demand. The CLI provides a brief connection settle delay (`0.2s`) on one-shot commands.

### 2. Audio input device errors
**Issue:** `opal listen system doctor` reports microphone failure or ALSA reports `Device or resource busy`.  
**Solution:**
- Check for other processes using the microphone: `fuser -v /dev/snd/pcm*`
- List detected sound cards: `arecord -l`
- Test capture with ALSA utilities: `arecord -d 3 -f S16_LE -r 16000 test.wav`

### 3. Concurrency precondition failures (`HTTP 412`)
**Issue:** Modifying an intent, rule, or setting returns `HTTP 412 Precondition Failed`.  
**Solution:** Another client updated the configuration concurrently. When using the CLI, omit the `--if-match` flag to allow the CLI to automatically fetch the latest ETag before submitting the update.

---

## 13. Appendix: operation ↔ CLI ↔ GUI map

The following master cross-reference connects every REST operation in the OPAL suite to its corresponding CLI command and Web GUI interface.

| Service | REST Operation | Method & Path | CLI Command | Web GUI Location |
|---|---|---|---|---|
| **Listen** | `list_wakewords` | `GET /api/v1/wakewords/` | `opal listen wakewords list` | Wakewords Tab |
| | `create_wakeword` | `POST /api/v1/wakewords/` | `opal listen wakewords add` | Wakewords Tab > Add |
| | `list_intents` | `GET /api/v1/intents/` | `opal listen intents list` | Intents Tab |
| | `create_intent` | `POST /api/v1/intents/` | `opal listen intents add` | Intents Tab > New Intent |
| | `add_phrase` | `POST /api/v1/intents/{id}/phrases` | `opal listen intents add-phrase` | Intent Details > Add Phrase |
| | `list_slots` | `GET /api/v1/slots/` | `opal listen slots list` | Slots Tab |
| | `create_slot` | `POST /api/v1/slots/` | `opal listen slots add` | Slots Tab > Add Slot |
| | `add_slot_value` | `POST /api/v1/slots/{id}/values` | `opal listen slots add-value` | Slot Values Editor |
| | `list_profiles` | `GET /api/v1/profiles/` | `opal listen profiles list` | Profiles Tab |
| | `activate_profile`| `POST /api/v1/profiles/{id}/activate` | `opal listen profiles activate`| Profiles > Activate |
| | `test_text` | `POST /api/v1/recognition/test_text` | `opal listen test text` | Playground > Text Test |
| | `test_audio` | `POST /api/v1/recognition/test_audio` | `opal listen test audio` | Playground > File Upload |
| | `start_pipeline` | `POST /api/v1/recognition/start` | `opal listen pipeline start` | Live Control > Start |
| | `stop_pipeline` | `POST /api/v1/recognition/stop` | `opal listen pipeline stop` | Live Control > Stop |
| | `list_failed` | `GET /api/v1/failed_utterances/` | `opal listen failed list` | Failed Utterances Tab |
| | `promote_failed` | `POST /api/v1/failed_utterances/{id}/promote` | `opal listen failed promote` | Failed Utterance > Promote |
| | `run_regression` | `POST /api/v1/regression/run` | `opal listen test regression` | Tests > Run Regression |
| | `health_check` | `GET /api/v1/health/` | `opal listen system doctor` | System Status Panel |
| **Talk** | `list_engines` | `GET /api/v1/engines/` | `opal talk engines list` | Engines Panel |
| | `list_voices` | `GET /api/v1/voices/` | `opal talk voices list` | Voices Tab |
| | `create_voice` | `POST /api/v1/voices/` | `opal talk voices add` | Voices Tab > Register Voice |
| | `test_voice` | `POST /api/v1/voices/{id}/test` | `opal talk voices test` | Voice Card > Test Audio |
| | `list_profiles` | `GET /api/v1/profiles/` | `opal talk profiles list` | Profiles Tab |
| | `activate_profile`| `POST /api/v1/profiles/{id}/activate` | `opal talk profiles activate` | Profile Card > Set Active |
| | `generate_audio`| `POST /api/v1/generate` | `opal talk generate` | Playground > Synthesize |
| | `stream_audio` | `POST /api/v1/generate/stream` | `opal talk generate --stream` | Playground > Stream Preview |
| | `get_settings` | `GET /api/v1/settings/` | `opal talk settings list` | Settings Tab |
| | `export_config` | `GET /api/v1/backup/export` | `opal talk backup export` | Backup & Restore > Export |
| | `import_config` | `POST /api/v1/backup/import` | `opal talk backup import` | Backup & Restore > Import |
| **Event** | `publish_event` | `POST /api/v1/events/` | `opal event publish` | Events > Publish Modal |
| | `list_events` | `GET /api/v1/events/` | `opal event list` | Events Explorer Tab |
| | `list_topics` | `GET /api/v1/topics/` | `opal event topics` | Topics Overview Tab |
| | `list_rules` | `GET /api/v1/rules/` | `opal event rules list` | Rules Management Tab |
| | `create_rule` | `POST /api/v1/rules/` | `opal event rules create` | Rules > Create Rule Form |
| | `test_rule` | `POST /api/v1/rules/{id}/test` | `opal event rules test` | Rule Details > Test Rule |
| | `list_actions` | `GET /api/v1/actions/` | `opal event actions list` | Actions Management Tab |
| | `create_action` | `POST /api/v1/actions/` | `opal event actions create` | Actions > Create Action |
| | `execute_action`| `POST /api/v1/actions/{id}/execute` | `opal event actions execute` | Action Details > Run Now |
| | `list_executions`| `GET /api/v1/executions/` | `opal event executions list` | Execution Audit History |
| | `get_stats` | `GET /api/v1/stats/` | `opal event stats` | Dashboard Metrics Panel |
| | `get_health` | `GET /api/v1/health/` | `opal event health` | Health & Status Panel |
