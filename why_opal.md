# Why OPAL?

**OPAL is a set of small, local-first services for building assistants that can listen, talk, see, think, and act on your own hardware.**

Most voice and vision assistants come in two forms. One is a closed cloud product you can't inspect or extend. The other is a pile of scripts that wire one speech engine, one model, and one camera library together and break when any of them changes. OPAL takes a different approach. It splits an assistant into five focused services, each behind a stable, documented REST contract, and connects them with a shared event bus.

| Service | Role | One-line pitch |
|---|---|---|
| [**OPAL Listen**](#opal-listen--hear-and-understand) | Ears | Wakeword, speech-to-text, intent, and slot extraction, tuned for a Raspberry Pi 5 |
| [**OPAL Talk**](#opal-talk--speak) | Voice | Engine-neutral text-to-speech (Piper, Kokoro, …) with WAV, PCM, and live streaming |
| [**OPAL Look**](#opal-look--see) | Eyes | Engine-neutral computer vision: detection, classification, segmentation, pose, OCR |
| [**OPAL Think**](#opal-think--reason-and-act-autonomously) | Brain | Model-neutral inference, conversations, tool-using agents, and memory |
| [**OPAL Event**](#opal-event--the-nervous-system) | Nervous system | ZeroMQ event bus, rule engine, and sandboxed action execution |

You can run one service or all five, on one machine or several.

---

## The big ideas

### 1. Your data stays on your hardware

Everything runs locally by default. Listen does speech recognition with Vosk (and optionally faster-whisper) on the device. Talk synthesizes speech with host-installed Piper or Kokoro voices. Look runs vision models you already have. Think can point at a local inference server just as easily as at a hosted provider. Services bind to `127.0.0.1` by default, and when they are exposed beyond loopback, OPAL Event requires bearer authentication and ZeroMQ CURVE encryption.

There is also no hidden data collection. Look never records video automatically, uploaded media expires after 24 hours, and configuration backups never include secrets, media, or history.

### 2. Engines are interchangeable

Every service puts an implementation-neutral contract in front of its engines:

- Talk speaks in terms of *voices, profiles, and output formats*, not Piper CLI flags or Kokoro's Python API.
- Look speaks in terms of *engines, models, sources, and tasks*. Ultralytics YOLO is listed as an example adapter, not a requirement.
- Think speaks in terms of *providers, models, agents, and tools*. LiteLLM and Google ADK are suggested adapters, not wire dependencies.
- Listen uses a pluggable `ListenerProvider`, with Vosk built in and third-party providers loaded via entry points.

When a better engine comes along, you write one adapter and your clients, automations, and UI stay the same. Each engine reports what it can do, and options it doesn't support are **rejected rather than silently ignored**, so you won't be left wondering why a setting had no effect.

### 3. The API is the product

Each service has an authoritative **OpenAPI 3.1 contract**, and every surface is built from it:

- a **REST API** (FastAPI),
- a **CLI** (`opal listen …`, `opal talk …`, `opal event …`), where every command is a REST call, and
- a **React SPA**, which talks only to the REST API.

In the implemented services, tests check this parity. Every contract operation has a matching route, CLI command, and SPA usage, and responses are validated against the contract. Anything you can click in the UI, you can script from the CLI or call from your own code.

### 4. One standard across the suite

All five services follow the **OPAL Suite Standard**, so learning one makes the rest familiar:

- **Shared correlation headers.** `X-Opal-Session-Id`, `X-Opal-Transaction-Id`, `X-Opal-Request-Id`, and `X-Opal-Causation-Id` flow through HTTP and into event metadata. You can trace a request like *"computer, is anyone at the front door?"* from the wakeword to the camera check to the spoken reply.
- **Shared operational routes.** Every service has `/api/v1/health/`, `/health/live`, and `/health/ready` for orchestrators; atomic `GET`/`PATCH /api/v1/settings/`; and portable `backup/export`, `import`, and `validate`.
- **Safe concurrent configuration.** Strong ETags with `If-Match` prevent lost updates (`428` if the header is missing, `412` if it's stale).
- **Predictable HTTP behavior.** `201` + `Location` for creates, `202` for async admission, `204` for deletes, a uniform `OpalError` body, and cursor pagination (`items`, `next_cursor`, `has_more`) for growing histories.

### 5. Event-driven by design

Services don't call each other in fragile chains. They publish events onto a shared ZeroMQ bus using a simple two-frame wire format (topic + JSON envelope), and OPAL Event routes them. Listen publishes `opal.listen.intent.match` whenever it recognizes a command. Look can publish session and result events. Think can hand tool side effects to Event. You can add a new consumer, such as a dashboard, a logger, or a home-automation bridge, by subscribing to the bus, with no changes to existing services.

### 6. Built to keep running

These services are meant to run unattended on small devices:

- Configuration changes are **validated before they're applied** and swapped in atomically. If one is invalid, the last good configuration stays active.
- Imports and restores take safety backups automatically and roll back on failure.
- Watchdogs restart audio capture with exponential backoff if a microphone disappears.
- Queues, uploads, log retention, and caches have explicit limits, so nothing grows without bound.
- Health and "doctor" checks report problems without crashing the API.

---

## OPAL Listen — hear and understand

> "computer, turn on the kitchen light" →
> `{"wakeword": "computer", "intent_name": "light_turn_on", "slot_values": {"room_name": "kitchen"}}`

OPAL Listen turns live microphone audio into **structured intents**. It runs fully offline and is tuned for the Raspberry Pi 5 and Linux.

**Why use it**

- **Accurate wakeword detection.** A two-stage detector opens on a fast partial match, so the start of your command isn't lost, then confirms against sound-alike distractors such as "commuter" and "compute" to cut false triggers.
- **Grammar-constrained recognition.** The recognizer is built from your enabled intents, slot values, and aliases. This fixes common open-dictation errors such as "kitchen lie it" or dropped short words.
- **Intents and slots you define.** You create intents, example phrases, slots, values, and aliases through the API, CLI, or UI, or import them from YAML. Profiles let you switch between full configuration sets.
- **A built-in improvement loop.** Unmatched commands are saved as *failed utterances*. You can review them, **promote** them to new phrases with one call, and protect against regressions with a **regression suite** you run on demand.
- **Visibility into what's happening.** Live results stream over SSE, and recognition logs and latency history are stored. `opal listen test listen-debug` shows mic levels, partial transcripts, and VAD events in real time.
- **The same code path for tests and production.** Live audio and uploaded WAVs go through the same pipeline, so golden-set, negative, noise (20/10 dB SNR), and wakeword false-positive/false-negative tests exercise the real code.
- **Easy to wire up.** Every matched intent is published to the event bus, ready for OPAL Event rules or OPAL Think agents to act on.

**Stack:** FastAPI · SQLite (WAL + Alembic) · Vosk · optional faster-whisper · Typer CLI · React UI · Docker Compose.

---

## OPAL Talk — speak

OPAL Talk is a **text-to-speech service with one stable API** over whichever TTS engines you install. Piper and Kokoro adapters ship today.

**Why use it**

- **No engine lock-in.** Clients refer to OPAL `voice_id`s and profiles. Engine-specific features go through an opaque `engine_options` object, so the common API stays clean while advanced options remain available.
- **Low-latency streaming.** `POST /api/v1/generate/stream` returns a progressive raw PCM stream (`s16le`, mono, with the sample rate in `X-Opal-Sample-Rate`), so playback can begin before synthesis finishes. Pipe it directly to `aplay`.
- **Long text is handled for you.** Talk splits long input into chunks internally, and the chunk boundaries don't show up in the audio you get back.
- **Simple, portable controls.** Voice, speed, volume, format (WAV or PCM), and sample rate are the same for every engine.
- **Safe with model files.** Voices are registrations that point at resources already on the host. The API never uploads, overwrites, or deletes model files.
- **Easy to extend.** Implement the `TtsEngine` interface (`availability`, `describe_voice`, `synthesize`, `option_specs`) and register it under the `opal_talk.engines` entry-point group. Chunking, resampling, gain, and encoding are handled by the core.
- **Focused scope.** Playback, device control, and synthesis history are left to other components, which keeps Talk small and predictable.

```bash
opal talk voices add lessac --name Lessac --engine piper --ref en_US-lessac-medium
opal talk profiles add main --voice lessac --active
opal talk generate "Hello from OPAL." --stream -o - | aplay -r 22050 -f S16_LE -c 1
```

**Stack:** FastAPI · SQLite · piper-tts · kokoro-onnx · Typer CLI · React + Vite SPA (served at `/ui`).

---

## OPAL Look — see

OPAL Look is an **implementation-neutral computer vision service** that works alongside Listen and Talk. It covers object detection, image classification, instance and semantic segmentation, pose estimation, and OCR, with optional tracking for detection.

**Why use it**

- **Works with many video sources.** USB and Pi cameras, RTSP streams, HTTP/MJPEG feeds, video files on the host, and uploaded media are all registered the same way and validated before use.
- **One-shot or continuous analysis.** `POST /api/v1/analyze` gives a synchronous answer for a single image. Sessions run continuous analysis, and several can run concurrently and share a camera.
- **Results you can use directly.** Structured results stream over **SSE**. You can view an **annotated MJPEG preview** in the browser and capture **exact-frame JPEG snapshots** of interesting moments. If the exact frame has been evicted, the request fails rather than returning a newer frame.
- **Consistent coordinates.** Boxes, keypoints, and masks are normalized to the original orientation-corrected frame, not the model's resized input, and masks use a documented row-major RLE format. Downstream code doesn't need to know which model produced a result.
- **Privacy defaults.** Look does no identity recognition and never records automatically. Results are kept for 7 days, uploaded media for 24 hours, and snapshots only when you choose to keep them. Tracking IDs are temporary and tied to a session, not a person.
- **Reusable profiles.** Profiles are presets that work with any source, and sessions save their resolved settings at creation so results are reproducible.
- **Clear boundaries.** Zones, line-crossing rules, and automations are handled by consumers such as OPAL Event, which keeps the vision service small. Look can optionally publish `opal.look.*` events to the bus.

> **Status:** Look is defined by its full v1.3.0 OpenAPI contract (`opal_look_v1.3.0.yaml`). It is ready to implement against, but no reference implementation exists in this repository yet.

---

## OPAL Think — reason and act autonomously

OPAL Think is a **framework-neutral API for inference, conversations, and autonomous agents**. It is the reasoning layer that turns "the user said X and the camera saw Y" into "do Z."

**Why use it**

- **Four levels of interaction:**
  - **Generations:** isolated model calls, with no hidden persistence and no tools.
  - **Conversations:** persistent message history with model-generated turns.
  - **Agents:** reusable definitions combining instructions, models, tools, delegation, and memory.
  - **Runs:** durable, asynchronous, cancellable workflows with step-by-step traces and usage tracking.
- **Any provider, any model.** You configure local or hosted providers and register models by their existing identifiers. Capabilities are checked at invocation time, and unsupported inputs fail with `422` instead of degrading silently.
- **Controlled autonomy.** Agents can only call tools that are explicitly allowed and enabled. Server policy is checked **before every tool invocation and every delegation**. Model instructions can never grant privileges, denied tools fail closed and appear in the run trace, and tool side effects normally go through OPAL Event, which applies its own allowlists.
- **Multi-agent delegation with limits.** Agents can delegate to explicitly allowed child agents. Delegation shares the parent's budget, has a configurable maximum depth, and is attributed to the parent run so you can inspect it.
- **Opt-in memory.** Namespaced records at the service, agent, or conversation level, with backend-neutral search. Memory is off unless you enable it, and trimming a model's context never deletes stored messages.
- **Streaming throughout.** SSE events for `delta`, `tool_call`, `tool_result`, `usage`, and more, with bounded `Last-Event-ID` replay for run events.
- **Separation of concerns.** Think doesn't record audio or capture images; Listen, Look, and Talk do that. Think focuses on reasoning and coordinates the other services through tools and the event bus.

> **Status:** Think's contract was recovered from a partial source and is **explicitly incomplete**. Treat it as a design direction, not a final specification. No reference implementation exists in this repository yet.

---

## OPAL Event — the nervous system

OPAL Event is the **event bus, rule engine, and action runner** that connects the other services. It works on its own too, for anything that can publish a ZeroMQ message or an HTTP POST.

**Why use it**

- **A simple, fast bus.** A ZeroMQ XSUB/XPUB broker that any producer or consumer, in any language, can connect to using two frames: a topic and a JSON envelope. Events published over REST are routed through the same path as ZeroMQ events, so both are handled identically.
- **Rules without code.** Match on an exact topic or a `prefix.*` pattern, add typed AND conditions, and attach one or more actions. `vision.*` matches `vision.person.detected` but not `visionary`, and there are no surprising type coercions.
- **Sandboxed actions:**
  - **Shell** actions run as argv lists and never build a shell command string from event data.
  - **HTTP** actions are checked against a scheme/host/port allowlist *after DNS resolution*, on every connection, with redirects disabled.
  - **Python** plugins run in a separate process from a registered module and function, using JSON over stdin/stdout.
  - Templating is limited to `{{data.path}}` and `{{event}}`: no expressions and no evaluation.
- **Bounded, observable execution.** A fixed worker pool with a bounded queue. Every execution ends as `succeeded`, `failed`, `timed_out`, or `dropped` and is recorded with truncated, redacted output. Timed-out subprocess groups are killed.
- **Dry-run rules.** `POST /api/v1/rules/{id}/test` evaluates a rule against a sample event without running anything.
- **Live configuration.** A single YAML file is the source of truth and can be edited by hand or through REST. Valid changes to rules and actions apply immediately, and invalid ones leave the last good configuration in place.
- **Secure when exposed.** Everything binds to loopback by default. On any other interface, REST bearer auth and ZeroMQ CURVE are required, and tokens are never returned by the API.

```bash
opal event publish vision.object.detected --data-json '{"object":"person","camera":"front_door"}'
opal event rules list
opal event actions execute announce_person --event-json '{"topic":"manual.test","data":{"camera":"front_door"}}'
```

**Stack:** FastAPI · PyZMQ (with CURVE) · SQLite · `watchfiles` hot reload · Typer CLI · React + TypeScript + Vite SPA.

---

## Putting it together

Here's what a doorbell assistant could look like with OPAL:

```
 🎤  "computer, who's at the front door?"
      │
      ▼
 OPAL Listen ── opal.listen.intent.match ──▶ OPAL Event (bus + rules)
                                               │
                         ┌─────────────────────┼──────────────────────┐
                         ▼                     ▼                      ▼
                    OPAL Think           OPAL Look              your own
                 (agent decides)     (front_door camera:       subscribers
                         │            person detected)         (logs, HA, …)
                         ▼
                    OPAL Talk  ──▶ 🔊 "There's one person at the front door."
```

Every hop carries the same transaction ID, so you can trace the whole interaction. Every service can be replaced or upgraded independently. Every step can be inspected through a REST API, a CLI, and a web UI.

## Who OPAL is for

- **Makers and home-lab users** who want a Raspberry Pi assistant that works offline and doesn't send audio or video to the cloud.
- **Developers** who need speech, vision, or agent features behind a clean, versioned API instead of building directly on one engine.
- **Teams building embedded or edge products** who need health probes, atomic configuration, backups, and tests from the start.
- **Anyone experimenting with AI agents** who wants autonomy with real guardrails: explicit tool allowlists, policy checks on every call, and sandboxed side effects.

## Project status at a glance

| Service | Contract | Reference implementation |
|---|---|---|
| OPAL Listen | v1.3.0 (`opal_listen_v1.3.0.yaml`) | ✅ `opal-listen/` (implements the 1.2.0 contract) |
| OPAL Talk | v1.3.0 (`opal_talk_v1.3.0.yaml`) | ✅ `opal-talk/` (implements the 1.2.0 contract) |
| OPAL Event | v1.3.0 (`opal_event_v1.3.0.yaml`) | ✅ `opal-event/` (implements the 1.2.0 contract) |
| OPAL Look | v1.3.0 (`opal_look_v1.3.0.yaml`) | ⏳ Contract only |
| OPAL Think | v1.3.0 (`opal_think_v1.3.0.yaml`), **incomplete** | ⏳ Contract only |

The v1.3.0 contracts are a version-label release. The shared event-emission schema (`opal_events_shared_v1.3.0.openapi.yaml`) is still a draft and has not yet been merged into every service contract. See the [README](../README.md) and [CHANGELOG](../CHANGELOG.md) for details.

## Get started

Each implemented service has its own quick start:

- [`opal-listen/docs/QUICKSTART.md`](../opal-listen/docs/QUICKSTART.md)
- [`opal-talk/README.md`](../opal-talk/README.md) and [`opal-talk/docs/USER_GUIDE.md`](../opal-talk/docs/USER_GUIDE.md)
- [`opal-event/README.md`](../opal-event/README.md) and [`opal-event/docs/USER_GUIDE.md`](../opal-event/docs/USER_GUIDE.md)

Start with the service you need, such as Listen for voice commands or Talk for spoken responses, then add OPAL Event to connect it to everything else.
