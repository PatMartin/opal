# `opal` Unified CLI Reference

The `opal` command is the unified terminal interface to the entire OPAL suite. It brings together
command-line control for all OPAL services:

- **`opal listen`**: Offline wakeword detection, speech-to-text, intent recognition, slot extraction, live pipeline inspection, failed utterance review, and regression testing.
- **`opal talk`**: Engine-neutral text-to-speech generation (Piper, Kokoro), voice catalog management, progressive raw-PCM streaming, and audio profiles.
- **`opal event`**: ZeroMQ event bus ingress/inspection, topic monitoring, declarative rule matching, sandboxed action execution (shell, HTTP, Python plugins), and execution auditing.
- *(Contract specifications)* **`opal look`** & **`opal think`**: Computer vision and autonomous reasoning/agent interfaces defined in the OPAL v1.3.0 suite contracts.

The CLI is a thin REST client. **Every command (with the single exception of `opal listen event publish`)
calls the respective service's REST API** through the operations declared in the authoritative OpenAPI 3.1
specifications. It never opens SQLite databases directly and never requires local file access to engines
or models. The target services can be running locally or on remote network hosts.

---

## Table of contents

1. [Installation and invocation](#1-installation-and-invocation)
2. [Connecting to the services](#2-connecting-to-the-services)
3. [Conventions](#3-conventions)
4. [Command tree](#4-command-tree)
5. [`opal listen`: wakewords, intents, and voice recognition](#5-opal-listen-wakewords-intents-and-voice-recognition)
6. [`opal talk`: text-to-speech and audio streaming](#6-opal-talk-text-to-speech-and-audio-streaming)
7. [`opal event`: event bus, rules, and action execution](#7-opal-event-event-bus-rules-and-action-execution)
8. [Contract specifications: `opal look` and `opal think`](#8-contract-specifications-opal-look-and-opal-think)
9. [Cross-service walkthroughs](#9-cross-service-walkthroughs)
10. [Scripting and automation](#10-scripting-and-automation)
11. [Troubleshooting](#11-troubleshooting)
12. [Appendix: Unified CLI ↔ REST operations mapping](#12-appendix-unified-cli--rest-operations-mapping)

---

## 1. Installation and invocation

Each OPAL service is packaged as an independent Python package with its own dependencies and entrypoints.
When installed into the same environment, the root `opal` command dynamically discovers its sibling
service subcommands.

### Unified installation

To install all implemented OPAL services into a single virtual environment:

```bash
cd /gh/opal
python3 -m venv .venv
source .venv/bin/activate

# Install the sub-projects in editable mode
pip install -e "./opal-event[dev]"
pip install -e "./opal-listen[dev]"
pip install -e "./opal-talk[engines,dev]"
```

Once installed, the `opal` entrypoint provides access to all subcommands:

```bash
opal --help
opal listen --help
opal talk --help
opal event --help
```

### Standalone console scripts

In addition to the unified `opal` entrypoint, dedicated console aliases are available for single-service deployments:

| Entrypoint | Subcommand group | Source package | Default REST Port |
|---|---|---|---|
| `opal listen ...` | `listen` | `opal_listener` | `8000` |
| `opal talk ...` | `talk` | `opal_talk` | `8001` |
| `opal-talk ...` | Shortcut for `opal talk` | `opal_talk` | `8001` |
| `opal event ...` | `event` | `opal_event` | `8080` |
| `opal-event ...` | Shortcut for `opal event` | `opal_event` | `8080` |

### Python module execution

Any CLI command can also be run directly as a Python module without relying on PATH resolution:

```bash
python3 -m opal_listener.cli listen --help
python3 -m opal_talk.cli talk --help
python3 -m opal_event.cli event --help
```

### Shell completion

Tab completion is powered by Typer/Click and can be generated for Bash, Zsh, or Fish:

```bash
# Bash
opal --install-completion bash
source ~/.bashrc

# Zsh
opal --install-completion zsh
source ~/.zshrc

# Fish
opal --install-completion fish
```

---

## 2. Connecting to the services

Each sub-CLI communicates over HTTP with its corresponding daemon. When options are omitted,
the CLI resolves service endpoints from environment variables, falling back to local defaults.

### Service endpoint resolution

| Service | CLI Subcommand | Default Base URL | Environment Variable Override |
|---|---|---|---|
| OPAL Listen | `opal listen` | `http://127.0.0.1:8000` | `OPAL_LISTEN_API_URL` (or `OPAL_LISTENER_API_URL`) |
| OPAL Talk | `opal talk` | `http://127.0.0.1:8001` | `OPAL_TALK_API_URL` |
| OPAL Event | `opal event` | `http://127.0.0.1:8080` | `OPAL_EVENT_API_URL` |
| ZeroMQ XPUB (Listen) | `opal listen event` | `tcp://127.0.0.1:5559` | `OPAL_EVENT_XPUB_URL` |
| ZeroMQ XPUB (Event Broker) | `opal-event-svc` | `tcp://127.0.0.1:5555` | Defined in `opal_event.yaml` |

### Global CLI options

Global options are placed directly after the service sub-group name:

```bash
# OPAL Listen options
opal listen [--api-url URL] [--xpub XPUB_URL] [--xsub XSUB_URL] [--json] [--timeout SECONDS] <command>

# OPAL Talk options
opal talk [--api-url URL] [--json] [--timeout SECONDS] <command>

# OPAL Event options
opal event [--api-url URL] [--token TOKEN] [--json] [--timeout SECONDS] \
           [--opal-session-id ID] [--opal-transaction-id ID] [--opal-causation-id ID] <command>
```

> **Important:** Global options must appear **before** the specific subcommand.
> 
> ```bash
> # Correct:
> opal event --api-url http://pi5.local:8080 rules list
> opal talk --json voices list
>
> # Incorrect (option treated as unknown argument to subcommand):
> opal event rules list --api-url http://pi5.local:8080
> ```

### Authentication

`opal event` supports REST Bearer token authentication (mandatory when binding to non-loopback interfaces):

```bash
# Via flag:
opal event --token "secret-token-value" rules list

# Via environment variable:
export OPAL_EVENT_API_TOKEN="secret-token-value"
opal event rules list
```

---

## 3. Conventions

Across the entire OPAL suite, all CLI subcommands conform to uniform conventions.

### Input payloads (`--json`, `--file`, stdin)

Commands that create, update, or import configuration accept structured JSON data. They resolve payloads in strict priority order:

1. `--json '<raw JSON text>'`: Inline JSON string passed on the command line.
2. `--file <path>`: Path to a local `.json` file.
3. **Standard Input (stdin)**: Piped JSON when stdin is not a terminal TTY.

```bash
# 1. Inline JSON
opal event rules create --json '{"id":"r1","topic":"vision.*","conditions":[],"actions":["act1"]}'

# 2. File payload
opal event rules create --file examples/sample_rule.json

# 3. Piped stdin
cat examples/sample_rule.json | opal event rules create
jq '.enabled = false' rule.json | opal event rules update r1
```

### Strong ETags and optimistic concurrency

In accordance with the OPAL Suite Standard, mutable configuration writes (`PATCH`, `DELETE`, and `import`)
require an `If-Match` precondition header carrying the current strong ETag revision.

The CLI automates this for you:
- When `--if-match` is omitted on update and delete commands, **the CLI automatically performs a `GET` request immediately before the mutation to fetch the resource's current ETag**.
- You can provide an explicit `--if-match` when writing scripts that require strict optimistic concurrency:

```bash
opal event rules update r1 --json '{"enabled": false}' --if-match '"1-a1b2c3d4"'
```

If the server configuration has changed concurrently, the CLI halts with:
- `HTTP 412 Precondition Failed`: The target resource changed since the ETag was read.
- `HTTP 428 Precondition Required`: The server rejected an unconditioned mutation.

### Correlation tracking headers

The OPAL v1.2/v1.3 standard defines four request-correlation headers accepted on all endpoints and echoed in all responses:

| Header | Meaning | CLI Option / Source |
|---|---|---|
| `X-Opal-Session-Id` | Logical conversational or client session ID | `--opal-session-id` / `$OPAL_SESSION_ID` |
| `X-Opal-Transaction-Id` | Distributed transaction or multi-service trace ID | `--opal-transaction-id` / `$OPAL_TRANSACTION_ID` |
| `X-Opal-Request-Id` | Single HTTP exchange ID (auto-generated if omitted) | Automatically assigned per request |
| `X-Opal-Causation-Id` | ID of the parent event or triggering request | `--opal-causation-id` / `$OPAL_CAUSATION_ID` |

These headers are propagated into ZeroMQ event metadata and audit logs, enabling end-to-end tracing across services.

### Output modes

- **Interactive mode (default):** Outputs clean, formatted UTF-8 tables, colorized status badges, and human-friendly key-value blocks.
- **Machine-readable mode (`--json`):** Outputs raw, unadorned JSON to standard output, suitable for piping into `jq`, `grep`, or downstream scripts. Operational messages and warnings are routed to `stderr`.

```bash
# Interactive table
opal talk voices list

# Machine-readable JSON
opal talk --json voices list | jq '.[].voice_id'
```

### Bounded catalogs vs cursor pagination

- **Configuration catalogs** (intents, slots, voices, profiles, rules, actions) are bounded collections. They return the full list of items without pagination.
- **High-volume event and execution histories** (`opal event list`, `opal event executions list`, `opal listen logs list`) use opaque cursor pagination (`items`, `next_cursor`, `has_more`). Pass `--cursor <token>` and `--limit <n>` to traverse pages.

### Exit codes

| Exit Code | Meaning | Cause |
|---|---|---|
| `0` | Success | Command completed successfully |
| `1` | Application / Client Error | Invalid CLI argument, local file not found, bad JSON syntax |
| `2` | Connection / Network Error | Service unreachable, DNS failure, connection refused, or timeout |
| `3` | Server Error (`5xx`) | Service crashed, internal database error, engine failure |
| `4` | Precondition / Validation Error | Schema validation failure (`422`), stale ETag (`412`), or missing ETag (`428`) |

---

## 4. Command tree

```
opal
├── listen [--api-url] [--xpub] [--xsub] [--json] [--timeout]
│   ├── wakewords   list | get | add | update | remove
│   ├── intents     list | get | add | update | remove | phrases | add-phrase | remove-phrase
│   ├── slots       list | get | add | update | remove
│   │               values | add-value | remove-value | aliases | add-alias | remove-alias
│   ├── profiles    list | get | add | update | activate | remove
│   ├── settings    list | set
│   ├── test        text | audio | listen | listen-debug | regression
│   ├── pipeline    start | stop | status
│   ├── failed      list | get | add | update | remove | promote
│   ├── regression  list | get | add | update | remove
│   ├── logs        list | get | count
│   ├── perf        last | summary | history
│   ├── config      validate | backup | backups | download | restore | delete-backup | export | import
│   ├── system      doctor | devices | live | ready | info | ping
│   └── event       publish
│
├── talk [--api-url] [--json] [--timeout]
│   ├── engines     list | get | update | validate
│   ├── voices      list | get | add | update | remove | validate | test
│   ├── profiles    list | get | add | update | remove | activate
│   ├── generate    TEXT -o OUT [--stream] [--format] [--sample-rate] [--voice] [-O key=val]
│   ├── settings    list | get | set | delete
│   ├── backup      list | create | download | restore | delete | export | import | validate
│   ├── health
│   ├── info
│   └── ping
│
├── event [--api-url] [--token] [--json] [--timeout] [--opal-session-id] [--opal-transaction-id]
│   ├── publish     TOPIC [--data-json] [--data-file] [--source] [--id]
│   ├── list        [--topic] [--source] [--limit] [--cursor]
│   ├── get         EVENT_ID
│   ├── topics
│   ├── rules       list | get | create | update | delete | test
│   ├── actions     list | get | create | update | delete | execute
│   ├── executions  list | get
│   ├── settings    get | update
│   ├── backup      export | validate | import
│   ├── health      
│   ├── health-live
│   ├── health-ready
│   └── stats
│
└── [contract specifications: look & think]
    ├── look        engines | models | sources | profiles | media | analyze | sessions | results | snapshots
    └── think       providers | models | generations | conversations | agents | runs | tools | memory
```

---

## 5. `opal listen`: wakewords, intents, and voice recognition

`opal listen` manages speech recognition, NLU catalogs, audio hardware, testing pipelines, and failure triage.

### Wakewords (`opal listen wakewords`)

```bash
# List all configured wakewords
opal listen wakewords list

# View details for a wakeword
opal listen wakewords get computer

# Add a wakeword with sound-alike distractors to prevent false positives
opal listen wakewords add jarvis --phrase "jarvis" --threshold 0.65 \
  --distractor "service" --distractor "harvest"

# Update threshold
opal listen wakewords update jarvis --threshold 0.70

# Remove wakeword
opal listen wakewords remove jarvis
```

### Intents & phrases (`opal listen intents`)

```bash
# List all intents
opal listen intents list

# Add an intent with slot template variables
opal listen intents add light_set_brightness --description "Adjust lighting level"

# Add training phrases with slot placeholders
opal listen intents add-phrase light_set_brightness "set the {room} light to {level} percent"
opal listen intents add-phrase light_set_brightness "dim the {room} light"

# List phrases for an intent
opal listen intents phrases light_set_brightness

# Remove a phrase by ID
opal listen intents remove-phrase light_set_brightness 4
```

### Slots, values & aliases (`opal listen slots`)

```bash
# List slots
opal listen slots list

# Create a slot
opal listen slots add room --description "Target room name"

# Add canonical values and spoken aliases
opal listen slots add-value room kitchen
opal listen slots add-alias room kitchen "galley"
opal listen slots add-alias room kitchen "cookhouse"

opal listen slots add-value room living_room
opal listen slots add-alias room living_room "front room"
opal listen slots add-alias room living_room "lounge"

# View slot values and aliases
opal listen slots values room
opal listen slots aliases room kitchen
```

### Profiles (`opal listen profiles`)

Profiles group model selections, wakewords, and audio settings into switchable presets:

```bash
opal listen profiles list
opal listen profiles add quiet_room --model-path /models/vosk-en-small --wakeword computer
opal listen profiles activate quiet_room
```

### Testing recognition (`opal listen test`)

```bash
# Test text normalization, intent matching, and slot extraction directly
opal listen test text "computer turn on the galley light"

# Test an audio file through the full recognizer pipeline
opal listen test audio fixtures/kitchen_light_on.wav

# Start live microphone testing in the terminal (Ctrl+C to stop)
opal listen test listen

# Live microphone diagnostic with visual audio levels, VAD events, and partial transcripts
opal listen test listen-debug
```

### Pipeline control (`opal listen pipeline`)

Control the background microphone capture thread:

```bash
opal listen pipeline status
opal listen pipeline stop
opal listen pipeline start
```

### Failed utterance review & promotion (`opal listen failed`)

Utterances that fail recognition are captured in SQLite for iterative tuning:

```bash
# List recent unrecognized utterances
opal listen failed list --limit 20

# Inspect a failed utterance
opal listen failed get 12

# Promote a failed utterance directly into a training phrase for an intent
opal listen failed promote 12 --intent light_turn_on
```

### Regression testing (`opal listen regression`)

Prevent regressions when updating models, intents, or phrases:

```bash
# List regression test cases
opal listen regression list

# Run the full regression suite against the current live recognizer
opal listen test regression
```

### Performance & logs (`opal listen logs`, `opal listen perf`)

```bash
# Query recent recognition logs
opal listen logs list --limit 50
opal listen logs count

# Check performance metrics and latencies (wakeword latency, STT duration, NLU parse time)
opal listen perf last
opal listen perf summary
opal listen perf history
```

### Diagnostics & hardware (`opal listen system`)

```bash
# Comprehensive system diagnostic (checks models, SQLite, microphone devices, disk)
opal listen system doctor

# List available host audio input devices
opal listen system devices

# Check liveness and readiness
opal listen system live
opal listen system ready
```

### ZeroMQ direct publishing (`opal listen event publish`)

Publishes an event frame directly to the configured ZeroMQ XPUB socket (`--xpub`):

```bash
opal listen event publish opal.listen.intent.match '{"intent":"light_turn_on","slots":{"room":"kitchen"}}'
```

---

## 6. `opal talk`: text-to-speech and audio streaming

`opal talk` manages TTS synthesis engines (Piper, Kokoro), voice catalogs, speech generation, and live streaming.

### Engine discovery & validation (`opal talk engines`)

```bash
# List installed TTS engines and their capabilities
opal talk engines list

# Inspect engine options and supported features
opal talk engines get piper

# Validate an engine against host dependencies
opal talk engines validate piper
```

### Voice registrations (`opal talk voices`)

Voices point to model assets already installed on the host. The API never uploads or deletes model files:

```bash
# List registered voices
opal talk voices list

# Register a Piper voice pointing to a local model
opal talk voices add en_US-lessac-medium \
  --name "Lessac" \
  --engine piper \
  --ref "en_US-lessac-medium"

# Register a Kokoro voice
opal talk voices add af_heart \
  --name "Heart" \
  --engine kokoro \
  --ref "af_heart"

# Validate that voice files are readable on the host
opal talk voices validate en_US-lessac-medium

# Test speech generation for a specific voice
opal talk voices test en_US-lessac-medium --text "Voice test successful."
```

### Profiles (`opal talk profiles`)

Profiles set default voices, speech rates, volumes, and engine options:

```bash
# List profiles
opal talk profiles list

# Create a profile
opal talk profiles add fast_announcer \
  --voice en_US-lessac-medium \
  --speed 1.2 \
  --volume 1.0

# Activate profile
opal talk profiles activate fast_announcer
```

### Speech generation & streaming (`opal talk generate`)

`opal talk generate` produces audio files or streams live raw PCM bytes.

```bash
# Generate a completed WAV file
opal talk generate "The kitchen light has been turned on." -o kitchen.wav

# Generate raw PCM (s16le, 16kHz, mono)
opal talk generate "System rebooting." --format pcm --sample-rate 16000 -o reboot.pcm

# Stream audio progressively to stdout and pipe directly into local playback
opal talk generate "Progressive low-latency audio stream." --stream -o - | aplay -r 22050 -f S16_LE -c 1

# Override voice and engine-specific parameters for a single request
opal talk generate "High-priority alert." \
  --voice af_heart \
  --speed 1.1 \
  -O noise_scale=0.667 \
  -o alert.wav
```

### Settings & backups (`opal talk settings`, `opal talk backup`)

```bash
# View all service settings
opal talk settings list

# Adjust maximum text length or chunk size
opal talk settings set max_text_chars 4000
opal talk settings set default_speed 1.0

# Create and download a configuration backup
opal talk backup create
opal talk backup list
opal talk backup download backup_20261012.zip -o backup.zip
```

---

## 7. `opal event`: event bus, rules, and action execution

`opal event` provides event ingress, topic monitoring, rule matching, and sandboxed action execution.

### Publishing events (`opal event publish`)

Publishes an event through the REST ingress endpoint (`POST /api/v1/events/`). The server normalizes the
event and forwards it to the ZeroMQ XSUB endpoint:

```bash
# Publish with inline JSON payload
opal event publish vision.object.detected \
  --source front_door_cam \
  --data-json '{"object":"person","camera":"front_door","confidence":0.95}'

# Publish with payload from a JSON file
opal event publish sensor.temperature --data-file temp_reading.json
```

### Event history & topics (`opal event list`, `opal event topics`)

```bash
# List observed topics
opal event topics

# List recent events (cursor-paginated)
opal event list --limit 20

# Filter events by topic prefix or source
opal event list --topic "vision.*" --limit 10
opal event list --source "front_door_cam"

# Get full details of a specific event
opal event get e4f3a120-8b1e-4c7b-99f2-2a781b0f56d9
```

### Rule management (`opal event rules`)

Rules match event topics and evaluate strictly typed AND conditions against event data:

```bash
# List all configured rules
opal event rules list

# Inspect a rule
opal event rules get announce_door_motion

# Create a rule from JSON file or inline string
opal event rules create --json '{
  "id": "person_at_door",
  "topic": "vision.object.detected",
  "description": "Trigger alert when a person is detected with high confidence",
  "enabled": true,
  "conditions": [
    {"field": "data.object", "operator": "eq", "value": "person"},
    {"field": "data.confidence", "operator": "gte", "value": 0.85}
  ],
  "actions": ["say_doorbell", "log_alert"]
}'

# Test a rule against a sample event without executing actions (dry-run)
opal event rules test person_at_door --event-json '{
  "topic": "vision.object.detected",
  "data": {"object": "person", "confidence": 0.92}
}'

# Disable a rule
opal event rules update person_at_door --json '{"enabled": false}'

# Delete a rule
opal event rules delete person_at_door
```

### Action management & execution (`opal event actions`)

Actions define what happens when a rule matches. Handlers include `shell`, `http`, and `python`:

```bash
# List actions
opal event actions list

# Create a shell action (argv array, template parameters, non-interpolated)
opal event actions create --json '{
  "id": "say_doorbell",
  "handler": "shell",
  "config": {
    "command": ["/usr/local/bin/notify.sh", "{{data.object}}", "{{data.camera}}"],
    "timeout_sec": 5
  }
}'

# Create an HTTP action (validated against host allowlist, redirects disabled)
opal event actions create --json '{
  "id": "post_webhook",
  "handler": "http",
  "config": {
    "url": "http://127.0.0.1:8001/api/v1/generate",
    "method": "POST",
    "headers": {"Content-Type": "application/json"},
    "body_template": "{\"text\": \"Person detected at {{data.camera}}\"}"
  }
}'

# Manually execute an action with a test event payload
opal event actions execute say_doorbell --event-json '{
  "topic": "manual.test",
  "data": {"object": "visitor", "camera": "front_gate"}
}'
```

### Execution auditing (`opal event executions`)

All action runs are recorded in SQLite with timing, exit codes, and truncated stdout/stderr:

```bash
# List execution history (filter by status: succeeded, failed, timed_out, dropped)
opal event executions list --status failed
opal event executions list --action-id say_doorbell --limit 10

# Inspect execution record
opal event executions get a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d
```

### Diagnostics & statistics (`opal event health`, `opal event stats`)

```bash
# Summary health status
opal event health

# Liveness probe (HTTP server and SQLite database responsive)
opal event health-live

# Readiness probe (ZeroMQ broker active and configuration loaded)
opal event health-ready

# Operational metrics (event counts, worker queue depth, dropped execution counters)
opal event stats
```

---

## 8. Contract specifications: `opal look` and `opal think`

The OPAL v1.3.0 suite contracts specify the future CLI surface for `opal look` and `opal think`.

### Planned `opal look` CLI commands

Based on `opal_look_v1.3.0.yaml`:

```
opal look
├── engines     list | get | update | validate
├── models      list | get | add | update | remove | validate
├── sources     list | get | add | update | remove | validate
├── profiles    list | get | add | update | remove | validate | activate
├── media       upload | list | get | download | remove
├── analyze     IMAGE_PATH [--model] [--task] [--confidence]
├── sessions    start | list | get | stop | remove | latest | events | preview
├── results     search | get | remove | snapshot
└── snapshots   list | get | download | remove
```

- `opal look analyze frame.jpg --task detection`: Synchronous one-shot computer vision inference.
- `opal look sessions start --source camera_front --profile person_detect`: Start a continuous video analysis session.
- `opal look sessions preview <session_id>`: Open or stream the annotated MJPEG preview.
- `opal look results search --session <session_id> --label person`: Search retained historical detection records.
- `opal look snapshots download <snapshot_id> -o frame.jpg`: Download exact-frame high-resolution snapshot.

### Planned `opal think` CLI commands

Based on `opal_think_v1.3.0.yaml`:

```
opal think
├── providers       list | get | add | update | remove
├── models          list | get | add | update | remove
├── generations     create PROMPT [--stream] [--model]
├── conversations   list | create | get | remove | messages | respond
├── agents          list | get | add | update | remove
├── runs            list | create | get | cancel | events
├── tools           list | get | add | update | remove
└── memory          list | get | set | search | remove
```

- `opal think generations create "Summarize recent activity"`: Stateless model inference.
- `opal think conversations respond <conv_id> "What did the front door camera see?"`: Multi-turn conversational turn.
- `opal think agents add triage_agent --instructions "Monitor events and alert"`: Reusable agent configuration.
- `opal think runs create triage_agent --input "Evaluate security alerts"`: Asynchronous, audited agent execution run.

---

## 9. Cross-service walkthroughs

These walkthroughs demonstrate how OPAL's CLI tools coordinate across services.

### Walkthrough 1: End-to-end voice intent execution

**Goal:** Speak a command, have OPAL Listen recognize it, trigger an action in OPAL Event, and hear the response spoken through OPAL Talk.

**Step 1: Check health across the suite**

```bash
opal listen system live
opal talk ping
opal event health-live
```

**Step 2: Register a voice in OPAL Talk and test playback**

```bash
# Register Piper Lessac voice
opal talk voices add en_US-lessac-medium --name Lessac --engine piper --ref en_US-lessac-medium

# Test generating audio
opal talk generate "Voice system ready." -o test.wav
aplay -q test.wav
```

**Step 3: Configure an automated rule and action in OPAL Event**

```bash
# 1. Create an action that calls OPAL Talk to speak a phrase
opal event actions create --json '{
  "id": "speak_light_confirmation",
  "handler": "shell",
  "config": {
    "command": ["opal", "talk", "generate", "Turning on the {{data.room_name}} light", "-o", "/tmp/reply.wav"],
    "timeout_sec": 5
  }
}'

# 2. Create a rule matching OPAL Listen's published intent topic
opal event rules create --json '{
  "id": "on_light_turn_on",
  "topic": "opal.listen.intent.match",
  "conditions": [
    {"field": "data.intent_name", "operator": "eq", "value": "light_turn_on"}
  ],
  "actions": ["speak_light_confirmation"]
}'
```

**Step 4: Simulate speech recognition**

```bash
# Simulate a voice command matching the intent
opal listen test text "computer turn on the kitchen light"

# Publish the intent event onto the bus
opal event publish opal.listen.intent.match --data-json '{
  "intent_name": "light_turn_on",
  "room_name": "kitchen",
  "confidence": 1.0
}'
```

**Step 5: Verify execution**

```bash
# Inspect the execution record
opal event executions list --limit 1
aplay -q /tmp/reply.wav
```

---

### Walkthrough 2: Reviewing failed utterances and updating training phrases

**Goal:** Turn unrecognized speech utterances into supported training phrases without restarting services.

```bash
# 1. View recent failed utterances
opal listen failed list --limit 5

# Sample output:
# ID  TRANSCRIPT               NO_MATCH_REASON       CREATED_AT
# 14  illuminate the scullery  no_intent_matched     2026-10-12T14:02:11Z

# 2. Add "scullery" as an alias for the kitchen slot value
opal listen slots add-alias room kitchen "scullery"

# 3. Promote the failed utterance into a phrase for 'light_turn_on'
opal listen failed promote 14 --intent light_turn_on

# 4. Verify by running recognition test on the exact phrase
opal listen test text "illuminate the scullery"

# 5. Run the regression test suite to ensure existing intents still pass
opal listen test regression
```

---

## 10. Scripting and automation

The CLI is designed for integration into shell scripts, crontabs, and systemd units.

### Extracting fields with `jq`

Always pass `--json` when scripting:

```bash
# Find the currently active OPAL Talk profile ID
ACTIVE_PROFILE=$(opal talk --json profiles list | jq -r '.[] | select(.is_active == true) | .profile_id')
echo "Active profile: $ACTIVE_PROFILE"

# Get the last execution error message
LAST_ERROR=$(opal event --json executions list --status failed --limit 1 | jq -r '.[0].stderr')
```

### Traversing cursor-paginated event history

Traverse pages using `has_more` and `next_cursor`:

```bash
#!/usr/bin/env bash
set -euo pipefail

CURSOR=""
HAS_MORE="true"

while [ "$HAS_MORE" = "true" ]; do
  if [ -z "$CURSOR" ]; then
    RESP=$(opal event --json list --limit 50)
  else
    RESP=$(opal event --json list --limit 50 --cursor "$CURSOR")
  fi

  # Process batch
  echo "$RESP" | jq -c '.items[] | {id: .id, topic: .topic, timestamp: .timestamp}'

  HAS_MORE=$(echo "$RESP" | jq -r '.has_more')
  CURSOR=$(echo "$RESP" | jq -r '.next_cursor // ""')
done
```

### Continuous low-latency streaming pipeline

Generate progressive speech and pipe directly into `aplay` without writing intermediate files to disk:

```bash
#!/usr/bin/env bash
# Stream live speech directly to ALSA
TEXT="${1:-System alert: check required.}"
opal talk generate "$TEXT" --stream -o - | aplay -q -r 22050 -f S16_LE -c 1
```

---

## 11. Troubleshooting

### Connection refused (`Errno 111`)

```
Error: Cannot connect to OPAL service at http://127.0.0.1:8000 (Connection refused)
```

- Verify that the target service is running:
  ```bash
  systemctl status opal-listener
  systemctl status opal-talk-svc
  systemctl status opal-event-svc
  ```
- Check that you are connecting to the correct port:
  - Listen: port `8000`
  - Talk: port `8001`
  - Event: port `8080`
- If running on a remote host, set the appropriate URL:
  ```bash
  export OPAL_LISTEN_API_URL="http://192.168.1.50:8000"
  ```

### Precondition failure (`HTTP 412`)

```
Error: HTTP 412 Precondition Failed: ETag mismatch.
```

- Someone or another process updated the rule, action, voice, or setting concurrently.
- If scripting with explicit `--if-match`, re-fetch the latest resource state and retry.
- Omit `--if-match` to let the CLI automatically fetch the current ETag prior to mutation.

### ZeroMQ publisher / broker port mismatch

If events published by `opal listen` are not being received by `opal event`:
- Note default configuration differences:
  - `opal-listener` default publisher connects to: `tcp://127.0.0.1:5559`
  - `opal-event-svc` default XSUB (inbound) broker binds to: `tcp://127.0.0.1:5555`
- To bridge them, configure `app_settings.xpub_url` in `opal-listener` to point to `tcp://127.0.0.1:5555`, or start an external proxy broker.

### Audio device capture issues (`opal listen`)

```bash
# Run doctor check
opal listen system doctor

# List detected ALSA audio devices
opal listen system devices

# Test raw microphone input levels
opal listen test listen-debug
```

If ALSA reports `Device or resource busy`:
- Check for other processes capturing the microphone: `fuser -v /dev/snd/pcm*`
- If using PulseAudio or PipeWire, ensure the service is configured to use the `default` or `pulse` ALSA plugin.

---

## 12. Appendix: Unified CLI ↔ REST operations mapping

The master table below maps every CLI command across `opal listen`, `opal talk`, and `opal event`
to its HTTP method, API endpoint, and OpenAPI operation ID.

### OPAL Listen operations

| CLI command (`opal listen ...`) | HTTP method | API endpoint | OpenAPI operationId |
|---|---|---|---|
| `wakewords list` | `GET` | `/api/v1/wakewords/` | `list_wakewords` |
| `wakewords add` | `POST` | `/api/v1/wakewords/` | `create_wakeword` |
| `wakewords get <id>` | `GET` | `/api/v1/wakewords/{id}` | `get_wakeword` |
| `wakewords update <id>` | `PATCH` | `/api/v1/wakewords/{id}` | `update_wakeword` |
| `wakewords remove <id>` | `DELETE` | `/api/v1/wakewords/{id}` | `delete_wakeword` |
| `intents list` | `GET` | `/api/v1/intents/` | `list_intents` |
| `intents add` | `POST` | `/api/v1/intents/` | `create_intent` |
| `intents get <id>` | `GET` | `/api/v1/intents/{id}` | `get_intent` |
| `intents update <id>` | `PATCH` | `/api/v1/intents/{id}` | `update_intent` |
| `intents remove <id>` | `DELETE` | `/api/v1/intents/{id}` | `delete_intent` |
| `intents phrases <id>` | `GET` | `/api/v1/intents/{id}/phrases` | `list_phrases` |
| `intents add-phrase <id>` | `POST` | `/api/v1/intents/{id}/phrases` | `add_phrase` |
| `intents remove-phrase <id> <pid>` | `DELETE` | `/api/v1/intents/{id}/phrases/{phrase_id}` | `delete_phrase` |
| `slots list` | `GET` | `/api/v1/slots/` | `list_slots` |
| `slots add` | `POST` | `/api/v1/slots/` | `create_slot` |
| `slots get <id>` | `GET` | `/api/v1/slots/{id}` | `get_slot` |
| `slots update <id>` | `PATCH` | `/api/v1/slots/{id}` | `update_slot` |
| `slots remove <id>` | `DELETE` | `/api/v1/slots/{id}` | `delete_slot` |
| `slots values <id>` | `GET` | `/api/v1/slots/{id}/values` | `list_slot_values` |
| `slots add-value <id> <val>` | `POST` | `/api/v1/slots/{id}/values` | `add_slot_value` |
| `slots remove-value <id> <vid>` | `DELETE` | `/api/v1/slots/{id}/values/{value_id}` | `delete_slot_value` |
| `slots aliases <id> <val>` | `GET` | `/api/v1/slots/{id}/values/{vid}/aliases` | `list_aliases` |
| `slots add-alias <id> <val>` | `POST` | `/api/v1/slots/{id}/values/{vid}/aliases` | `add_alias` |
| `slots remove-alias <id> <aid>` | `DELETE` | `/api/v1/slots/{id}/values/{vid}/aliases/{aid}`| `delete_alias` |
| `profiles list` | `GET` | `/api/v1/profiles/` | `list_profiles` |
| `profiles add` | `POST` | `/api/v1/profiles/` | `create_profile` |
| `profiles get <id>` | `GET` | `/api/v1/profiles/{id}` | `get_profile` |
| `profiles update <id>` | `PATCH` | `/api/v1/profiles/{id}` | `update_profile` |
| `profiles remove <id>` | `DELETE` | `/api/v1/profiles/{id}` | `delete_profile` |
| `profiles activate <id>` | `POST` | `/api/v1/profiles/{id}/activate` | `activate_profile` |
| `settings list` | `GET` | `/api/v1/settings/` | `get_settings` |
| `settings set <key> <val>` | `PUT` | `/api/v1/settings/{key}` | `put_setting` |
| `test text <txt>` | `POST` | `/api/v1/recognition/test_text` | `test_text` |
| `test audio <path>` | `POST` | `/api/v1/recognition/test_audio` | `test_audio` |
| `pipeline start` | `POST` | `/api/v1/recognition/start` | `start_pipeline` |
| `pipeline stop` | `POST` | `/api/v1/recognition/stop` | `stop_pipeline` |
| `pipeline status` | `GET` | `/api/v1/recognition/status` | `get_status` |
| `failed list` | `GET` | `/api/v1/failed_utterances/` | `list_failed_utterances` |
| `failed get <id>` | `GET` | `/api/v1/failed_utterances/{id}` | `get_failed_utterance` |
| `failed promote <id>` | `POST` | `/api/v1/failed_utterances/{id}/promote` | `promote_failed_utterance` |
| `failed remove <id>` | `DELETE` | `/api/v1/failed_utterances/{id}` | `delete_failed_utterance` |
| `regression list` | `GET` | `/api/v1/regression/` | `list_regression_tests` |
| `regression add` | `POST` | `/api/v1/regression/` | `create_regression_test` |
| `test regression` | `POST` | `/api/v1/regression/run` | `run_regression_tests` |
| `logs list` | `GET` | `/api/v1/logs/` | `list_logs` |
| `logs count` | `GET` | `/api/v1/logs/count` | `count_logs` |
| `perf last` | `GET` | `/api/v1/performance/last` | `get_last_performance` |
| `perf summary` | `GET` | `/api/v1/performance/summary` | `get_performance_summary` |
| `perf history` | `GET` | `/api/v1/performance/history` | `get_performance_history` |
| `system doctor` | `GET` | `/api/v1/health/` | `health_check` |
| `system devices` | `GET` | `/api/v1/health/audio_devices` | `get_audio_devices` |
| `system live` | `GET` | `/api/v1/health/live` | `liveness_probe` |
| `system ready` | `GET` | `/api/v1/health/ready` | `readiness_probe` |
| `config export` | `GET` | `/api/v1/backup/export` | `export_config` |
| `config import` | `POST` | `/api/v1/backup/import` | `import_config` |
| `config validate` | `GET` | `/api/v1/backup/validate` | `validate_config` |

---

### OPAL Talk operations

| CLI command (`opal talk ...`) | HTTP method | API endpoint | OpenAPI operationId |
|---|---|---|---|
| `engines list` | `GET` | `/api/v1/engines/` | `list_engines_api_v1_engines__get` |
| `engines get <id>` | `GET` | `/api/v1/engines/{engine_id}` | `get_engine_api_v1_engines__engine_id__get` |
| `engines update <id>` | `PATCH` | `/api/v1/engines/{engine_id}` | `update_engine_api_v1_engines__engine_id__patch` |
| `engines validate <id>` | `POST` | `/api/v1/engines/{engine_id}/validate` | `validate_engine_api_v1_engines__engine_id__validate_post` |
| `voices list` | `GET` | `/api/v1/voices/` | `list_voices_api_v1_voices__get` |
| `voices add` | `POST` | `/api/v1/voices/` | `create_voice_api_v1_voices__post` |
| `voices get <id>` | `GET` | `/api/v1/voices/{voice_id}` | `get_voice_api_v1_voices__voice_id__get` |
| `voices update <id>` | `PATCH` | `/api/v1/voices/{voice_id}` | `update_voice_api_v1_voices__voice_id__patch` |
| `voices remove <id>` | `DELETE` | `/api/v1/voices/{voice_id}` | `delete_voice_api_v1_voices__voice_id__delete` |
| `voices validate <id>` | `POST` | `/api/v1/voices/{voice_id}/validate` | `validate_voice_api_v1_voices__voice_id__validate_post` |
| `voices test <id>` | `POST` | `/api/v1/voices/{voice_id}/test` | `test_voice_api_v1_voices__voice_id__test_post` |
| `profiles list` | `GET` | `/api/v1/profiles/` | `list_profiles_api_v1_profiles__get` |
| `profiles add` | `POST` | `/api/v1/profiles/` | `create_profile_api_v1_profiles__post` |
| `profiles get <id>` | `GET` | `/api/v1/profiles/{profile_id}` | `get_profile_api_v1_profiles__profile_id__get` |
| `profiles update <id>` | `PATCH` | `/api/v1/profiles/{profile_id}` | `update_profile_api_v1_profiles__profile_id__patch` |
| `profiles remove <id>` | `DELETE` | `/api/v1/profiles/{profile_id}` | `delete_profile_api_v1_profiles__profile_id__delete` |
| `profiles activate <id>` | `POST` | `/api/v1/profiles/{profile_id}/activate` | `activate_profile_api_v1_profiles__profile_id__activate_post` |
| `generate` | `POST` | `/api/v1/generate` | `generate_audio_api_v1_generate_post` |
| `generate --stream` | `POST` | `/api/v1/generate/stream` | `stream_audio_api_v1_generate_stream_post` |
| `settings list` | `GET` | `/api/v1/settings/` | `get_all_settings_api_v1_settings__get` |
| `settings get <key>` | `GET` | `/api/v1/settings/{key}` | `get_setting_api_v1_settings__key__get` |
| `settings set <key>` | `PUT` | `/api/v1/settings/{key}` | `upsert_setting_api_v1_settings__key__put` |
| `settings delete <key>`| `DELETE` | `/api/v1/settings/{key}` | `delete_setting_api_v1_settings__key__delete` |
| `health` | `GET` | `/api/v1/health/` | `health_check_api_v1_health__get` |
| `info` | `GET` | `/` | `root__get` |
| `ping` | `GET` | `/healthz` | `healthz_healthz_get` |
| `backup list` | `GET` | `/api/v1/backup/list` | `list_backups_api_v1_backup_list_get` |
| `backup create` | `POST` | `/api/v1/backup/create` | `create_backup_api_v1_backup_create_post` |
| `backup download` | `GET` | `/api/v1/backup/download/{filename}` | `download_backup_api_v1_backup_download__filename__get` |
| `backup restore` | `POST` | `/api/v1/backup/restore/{filename}` | `restore_backup_api_v1_backup_restore__filename__post` |
| `backup export` | `GET` | `/api/v1/backup/export` | `export_config_api_v1_backup_export_get` |
| `backup import` | `POST` | `/api/v1/backup/import` | `import_config_api_v1_backup_import_post` |
| `backup validate` | `GET` | `/api/v1/backup/validate` | `validate_config_api_v1_backup_validate_get` |

---

### OPAL Event operations

| CLI command (`opal event ...`) | HTTP method | API endpoint | OpenAPI operationId |
|---|---|---|---|
| `publish` | `POST` | `/api/v1/events/` | `publish_event` |
| `list` | `GET` | `/api/v1/events/` | `list_events` |
| `get <id>` | `GET` | `/api/v1/events/{event_id}` | `get_event` |
| `topics` | `GET` | `/api/v1/topics/` | `list_topics` |
| `rules list` | `GET` | `/api/v1/rules/` | `list_rules` |
| `rules create` | `POST` | `/api/v1/rules/` | `create_rule` |
| `rules get <id>` | `GET` | `/api/v1/rules/{rule_id}` | `get_rule` |
| `rules update <id>` | `PATCH` | `/api/v1/rules/{rule_id}` | `patch_rule` |
| `rules delete <id>` | `DELETE` | `/api/v1/rules/{rule_id}` | `delete_rule` |
| `rules test <id>` | `POST` | `/api/v1/rules/{rule_id}/test` | `test_rule` |
| `actions list` | `GET` | `/api/v1/actions/` | `list_actions` |
| `actions create` | `POST` | `/api/v1/actions/` | `create_action` |
| `actions get <id>` | `GET` | `/api/v1/actions/{action_id}` | `get_action` |
| `actions update <id>` | `PATCH` | `/api/v1/actions/{action_id}` | `patch_action` |
| `actions delete <id>` | `DELETE` | `/api/v1/actions/{action_id}` | `delete_action` |
| `actions execute <id>` | `POST` | `/api/v1/actions/{action_id}/execute` | `execute_action` |
| `executions list` | `GET` | `/api/v1/executions/` | `list_executions` |
| `executions get <id>` | `GET` | `/api/v1/executions/{execution_id}` | `get_execution` |
| `settings get` | `GET` | `/api/v1/settings/` | `get_settings` |
| `settings update` | `PATCH` | `/api/v1/settings/` | `patch_settings` |
| `backup export` | `GET` | `/api/v1/backup/export` | `export_backup` |
| `backup validate` | `POST` | `/api/v1/backup/validate` | `validate_backup` |
| `backup import` | `POST` | `/api/v1/backup/import` | `import_backup` |
| `health` | `GET` | `/api/v1/health/` | `get_health` |
| `health-live` | `GET` | `/api/v1/health/live` | `get_health_live` |
| `health-ready` | `GET` | `/api/v1/health/ready` | `get_health_ready` |
| `stats` | `GET` | `/api/v1/stats/` | `get_stats` |
