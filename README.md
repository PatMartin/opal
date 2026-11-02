# OPAL — Give Your AI Ears, Eyes, a Voice, and a Brain

![OPAL](opal.png)

**Build an AI assistant that lives on your hardware, uses tools you choose, and grows one useful skill at a time.**

Imagine telling your computer, “Turn on the kitchen lights,” and watching it happen. Or getting a spoken alert when a camera spots a visitor. Or asking an AI to investigate a computer problem, show the evidence, and request permission *before* changing anything.

That is the idea behind **OPAL**: **Open, modular, local-first AI**.

Instead of installing one enormous assistant and hoping it does everything, you choose the parts you need. Like building with LEGO, except some of the bricks talk, some watch cameras, and one might eventually explain why your printer hates you.

> **Our promise:** Listen. Talk. Look. Think. Act. You stay in charge.
>
> **Reality check (November 2026):** Listen, Talk, and Event have Python implementations. Look has a detailed API design but is **not yet a running service**. Think is a planned service. Examples below say **WORKS TODAY**, **NEEDS GLUE**, or **FUTURE DESIGN** so you know what you can actually run.

[What is OPAL?](#meet-the-five-pieces) · [Try it](#five-minute-tour-of-the-working-pieces) · [Install](#getting-started) · [Adventures](#build-something-fun) · [Command tree](#the-big-command-tree) · [Under the hood](#how-opal-works) · [Roadmap](#whats-next)

---

## The big idea, without the jargon

Most AI assistants arrive as one big package. You get the tools, models, and rules their maker picked.

OPAL takes another route:

- **Pick your hardware.** A Raspberry Pi can listen in one room while a GPU-equipped computer handles the heavy thinking.
- **Pick your engines.** Use Vosk for speech recognition and Piper or Kokoro for speech. Switch engines without rewriting every app.
- **Keep things local.** Your microphone and camera do not have to send everything to a cloud provider.
- **Start small.** A voice command does not need a giant language model if a simple rule will do.
- **See what happened.** Inspect events, test rules, review failures, and trace operations.
- **Keep control.** A future AI agent should ask before making risky changes. A clever guess is not permission.

**One important term: API.** An API is a way one program asks another program to do something. OPAL uses **REST**, a common web API style. That means your terminal, web app, Python script, and future AI agent can all ask a service to do the same job.

## Meet the five pieces

| Piece | Plain-English job | Example | Status |
| --- | --- | --- | --- |
| 👂 **[Listen](https://github.com/PatMartin/opal-listen)** | Hear words and understand commands | “Computer, turn on the kitchen lights” | **Working** |
| 🗣️ **[Talk](https://github.com/PatMartin/opal-talk)** | Turn written words into speech audio | “The garage door is still open” | **Working** |
| ⚡ **[Event](https://github.com/PatMartin/opal-event)** | Pass messages around and run approved actions | When a sensor reports heat, notify someone | **Working** |
| 👁️ **[Look](https://github.com/PatMartin/opal-look)** | Watch images and video and describe what models find | Spot a person in a camera feed | **API designed; implementation to come** |
| 🧠 **[Think](https://github.com/PatMartin/opal-think)** | Reason, plan, ask for approval, and use tools | Investigate a failing service without blindly changing it | **Planned** |

Think of Listen as the ears, Talk as the voice, Look as the eyes, Think as the brain, and Event as the nervous system.

You do **not** need all five. A Pi that hears simple commands and triggers events is already useful.

### What could the whole system do?

```text
You: "Computer, is someone at the front door?"
                  |
             OPAL Listen
             understands you
                  |
             OPAL Think
             decides what to ask
                  |
             OPAL Look
             checks the camera
                  |
             OPAL Event
             can notify other apps
                  |
             OPAL Talk
             creates a spoken reply
                  |
Computer: "I can see a person near the porch."
```

**This full conversation is a vision, not a working demo today.** Look and Think need implementations, and a playback app must send Talk's generated audio to a speaker. OPAL is honest about the gap between a good design and running code.

---

## Five-minute tour of the working pieces

Already running one of the services? These are real CLI patterns from the service documentation. **Each service has its own command-line installation today**; see [Getting started](#getting-started) before you try the combined-looking examples.

### 1. Make your computer say something — WORKS TODAY

```bash
# See which voices are registered.
opal-talk voices list

# Generate spoken audio with your active voice.
opal-talk generate "Greetings, human. The machines are being polite today." -o greeting.wav

# Play it on a Linux computer with an audio player.
ffplay -autoexit -nodisp greeting.wav
```

Talk makes the audio file. **Your player** makes the sound. This separation is useful: the service can run on a server while a different device handles the speaker.

Want a particular voice?

```bash
opal-talk engines list
opal-talk voices list --engine kokoro
opal-talk voices test kokoro-af-heart -o audition.wav
```

The voice ID must be registered, the model files must be installed, and the engine must be available. Voice IDs above are examples, not a promise that they are preinstalled.

### 2. Teach your assistant a phrase — WORKS TODAY

```bash
# Run these using Listen's own virtual environment.
opal listen system doctor
opal listen wakewords list
opal listen intents list

# Try a phrase without turning on any real device.
opal listen test text "computer, turn on the kitchen light"
```

A recognition result can contain something like:

```json
{
  "wakeword": "computer",
  "intent_name": "light_turn_on",
  "slot_values": {"room_name": "kitchen"}
}
```

An **intent** is what you want to do. A **slot** is a detail, such as which room. This makes a small local speech recognizer surprisingly capable.

**Good to know:** A text test checks understanding; it does **not** turn on a real light or publish a live command.

### 3. Send a message through the system — WORKS TODAY

```bash
opal-event health

opal-event publish sensor.temperature.reading \
  --data-json '{"room":"workshop","celsius":31.5}'

opal-event list --topic sensor.temperature.reading --limit 10
```

A **topic** is the label on a message. Here, `sensor.temperature.reading` tells listeners what kind of message it is.

The publish command reports when Event accepts the message. It does **not** prove every subscriber saw it. OPAL's event bus is **best effort**, not a guaranteed-delivery system.

---

## Getting started

You can use these projects separately, or run them on different computers on your network.

### What you need

A modern Linux machine, Python 3.11 or newer, Git, and the hardware or model files for the service you want to use. On Ubuntu, this is a reasonable starting point:

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-dev \
  build-essential libasound2-dev portaudio19-dev \
  libzmq3-dev ffmpeg sox
```

**Put the projects side by side:**

```bash
mkdir -p ~/opal-suite
cd ~/opal-suite

git clone https://github.com/PatMartin/opal-event.git
git clone https://github.com/PatMartin/opal-listen.git
git clone https://github.com/PatMartin/opal-talk.git
git clone https://github.com/PatMartin/opal-api.git
```

You can also clone [this umbrella repository](https://github.com/PatMartin/opal) for the guide itself. It does not currently install an all-in-one application.

### Start Event: the message switchboard

In terminal 1:

```bash
cd ~/opal-suite/opal-event
python3 -m venv .venv
.venv/bin/pip install -e '.[dev]'

cp config/opal_event.example.yaml config/opal_event.yaml
mkdir -p var

.venv/bin/opal-event-svc --config config/opal_event.yaml --log-level info
```

In terminal 2:

```bash
~/opal-suite/opal-event/.venv/bin/opal-event health
~/opal-suite/opal-event/.venv/bin/opal-event topics
```

### Start Listen: the ears

In a separate terminal:

```bash
cd ~/opal-suite/opal-listen
python3 -m venv .venv
.venv/bin/pip install -e '.[dev]'

# Download a compatible Vosk model separately.
# Replace this path with the real extracted model directory.
export OPAL_LISTENER_VOSK_MODEL_PATH=/absolute/path/to/vosk-model-small-en-us-0.15

.venv/bin/python -m opal_listener
```

Check it from another terminal:

```bash
~/opal-suite/opal-listen/.venv/bin/opal listen system doctor
~/opal-suite/opal-listen/.venv/bin/opal listen test text \
  "computer, turn on the kitchen light"
```

For microphones, models, Docker, and sound-device setup, use the [Listen quick start](https://github.com/PatMartin/opal-listen/blob/main/docs/QUICKSTART.md).

### Start Talk: the voice

In a separate terminal:

```bash
cd ~/opal-suite/opal-talk
python3 -m venv .venv
.venv/bin/pip install -e '.[engines,dev]'

# Install Piper and/or Kokoro voice model files separately.
# Then configure the correct paths in config/service.yaml.
.venv/bin/opal-talk-svc
```

Check it from another terminal:

```bash
~/opal-suite/opal-talk/.venv/bin/opal-talk health
~/opal-suite/opal-talk/.venv/bin/opal-talk engines list
~/opal-suite/opal-talk/.venv/bin/opal-talk voices list
```

See the [Talk guide](https://github.com/PatMartin/opal-talk/blob/main/docs/USER_GUIDE.md) for engine files, voice setup, and streaming.

### One little packaging trap

Each working service publishes its own `opal` console command. If you install them in **one Python environment**, one entry point can replace another. There is **not yet a single suite-wide `opal` dispatcher**.

For now:

- Use separate virtual environments, as shown above.
- Use `opal-event` for Event and `opal-talk` for Talk.
- Use Listen's environment-specific `opal listen ...`.
- When this guide shows `opal event ...` or `opal talk ...`, those are the native forms documented by their respective service CLIs, **not** proof that one globally installed command supports all services.

In other words: the command *language* is shared; the installation experience is still catching up.

---

## Build something fun

The best way to learn OPAL is to give it a mission.

### Adventure 1: Give your Raspberry Pi a secret phrase — WORKS TODAY

Give Listen a new wakeword and practice recognizing commands.

```bash
opal listen wakewords add "hey opal" --threshold 0.85
opal listen wakewords list

opal listen test text "computer, turn on the kitchen light"

# Live microphone testing (Ctrl-C to stop).
opal listen test listen-debug
```

You can build your own vocabulary too:

```bash
# Add a place to a room-name slot if that slot already exists.
opal listen slots add-value room_name garage

# Expand an existing intent with a natural way to say the same thing.
opal listen intents add-phrase light_turn_on \
  'switch on [the] {room_name} (light | lights)'

# Check the result before hooking anything up.
opal listen test text "switch on the garage lights" \
  --expect-intent light_turn_on
```

The square brackets mean “optional words.” Parentheses offer alternatives. The braces hold a slot. It is like teaching a tiny robot grammar instead of asking a giant LLM to understand everything.

**What you need:** A suitable model and microphone for live listening; the sample seeded intents and slots may differ if you have changed your configuration.

### Adventure 2: Build a talking announcement machine — WORKS TODAY

Want a funny welcome, a timer announcement, or an audio story generator?

```bash
opal-talk generate \
  "Attention everyone. The kitchen has been declared a cookie-friendly zone." \
  -o announcement.wav

ffplay -autoexit -nodisp announcement.wav
```

Need to speak while the rest of the sentence is still being generated? Stream raw audio to a speaker:

```bash
opal-talk generate \
  "This sentence can begin playing before it is fully generated." \
  --stream --sample-rate 22050 -o - \
  | aplay -t raw -f S16_LE -c 1 -r 22050
```

The sample rate on both sides must match. On a server without speakers, send the audio to a separate playback process or device.

### Adventure 3: Make your first event rule — WORKS TODAY

Pretend your workshop is getting too warm. We will create a rule but leave it **disabled**. The test will check logic without executing any action.

First create an event action that only prints a message:

```bash
opal-event actions create --json '{
  "id": "announce_hot_workshop",
  "type": "shell",
  "enabled": true,
  "mode": "exec",
  "executable": "/bin/echo",
  "args": ["Workshop temperature is {{data.celsius}} C"],
  "stdin": "none",
  "timeout_seconds": 10
}'
```

Now create the rule:

```bash
opal-event rules create --json '{
  "id": "hot_workshop",
  "enabled": false,
  "topic": "sensor.temperature.reading",
  "conditions": [
    {"path": "data.room", "operator": "eq", "value": "workshop"},
    {"path": "data.celsius", "operator": "gt", "value": 30}
  ],
  "actions": ["announce_hot_workshop"]
}'
```

Dry-run the rule:

```bash
opal-event rules test hot_workshop --event-json '{
  "topic": "sensor.temperature.reading",
  "data": {"room": "workshop", "celsius": 31.5}
}'
```

If the checks match, the result should show `matched: true`. **Nothing is executed by a rule test.** You can inspect the action and decide whether to enable the rule later.

To test a specific registered action yourself (this **does execute** it):

```bash
opal-event actions execute announce_hot_workshop --event-json '{
  "topic": "sensor.temperature.reading",
  "data": {"celsius": 31.5}
}'

opal-event executions list
```

Notice the pattern: **create → inspect → test → allow action → inspect outcome**. That is a good habit for automations, and an essential habit for future AI agents.

### Adventure 4: Turn “computer, lights on” into a real automation — NEEDS GLUE

The pieces already exist:

```bash
# Confirm Listen understands the phrase.
opal listen test text \
  "computer, turn on the kitchen light" \
  --expect-intent light_turn_on

# Test Event separately with a synthetic intent message.
opal-event publish opal.listen.intent.match --data-json '{
  "intent_name": "light_turn_on",
  "slot_values": {"room_name": "kitchen"}
}'
```

For the actual automation, connect Listen's *live* recognition events to Event, register a trusted device-control action, and create a rule matching the real event payload. Live publication is configurable; diagnostic speech tests do **not** automatically publish an actionable intent.

Then a spoken phrase can become a light-control request, without sending the phrase to an LLM.

**Needs glue:** A real light integration, an approved Event action, and verified event routing. OPAL does not come with a magic “all your lights are now connected” switch.

### Adventure 5: “Who's on my porch?” — FUTURE DESIGN

This is where Look gets exciting. The [current central Look 1.4.0 contract](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_look.yaml) describes camera sources, processing pipelines, raw and annotated video, motion filters, live sessions, finite jobs, observations, snapshots, captures, rules, and monitors.

The **intended** CLI is generated from that contract, with names such as `sources`, `pipelines`, `sessions`, and `monitors`. This is a **design sketch, not runnable code**:

```bash
# FUTURE DESIGN — commands/options must match the implemented Look CLI.
opal look sources list
opal look sources show front_door
opal look sources probe front_door

# A processing pipeline could detect objects and add annotations.
opal look pipelines list
opal look pipelines show porch_detection

# A live session would produce ongoing observations.
opal look sessions list
opal look observations list
```

The design also supports a simple viewer: show a camera feed, optionally apply a motion filter, and turn annotations on or off. The REST contract already includes a direct camera stream endpoint:

```http
GET /api/v1/sources/{source_id}/stream
```

That is a **specification**, not a claim that an OPAL Look server is available today.

A future end-to-end adventure could look like this: camera sees person → Look generates an observation → Event routes a message → Think decides what to say → Talk generates “Someone's at the door.” Nothing in the design needs the camera to be owned by a language model.

### Adventure 6: Give your assistant a cautious brain — FUTURE DESIGN

Someday you might ask:

> “OPAL, my graphics driver stopped working. Figure out why, but do not change anything without asking.”

The planned Think service should follow a clear, inspectable process:

```text
Observe the problem
     ↓
Collect evidence
     ↓
Suggest likely causes
     ↓
Test or simulate possible fixes
     ↓
Explain risks and ask permission
     ↓
Make the approved change
     ↓
Check whether it worked
     ↓
Revert or revise if needed
```

That is **guarded autonomy**: useful AI that can investigate freely within read-only bounds, but cannot turn a guess into an unauthorized system change. There is no runnable Think CLI yet.

---

## The big command tree

Here is the OPAL command language in one picture. The **working** sections reflect the current service CLI guides. The **planned** sections are the intended surface, not executable software. This is a learning map; see the linked per-service command references for every option.

```text
opal
├── listen                                      WORKING (in Listen's CLI)
│   ├── wakewords    list | get | add | update | remove
│   ├── intents      list | get | add | update | remove
│   │                phrases | add-phrase | remove-phrase
│   ├── slots        list | get | add | update | remove
│   │                values | add-value | remove-value
│   │                aliases | add-alias | remove-alias
│   ├── profiles     list | get | add | update | activate | remove
│   ├── settings     list | set
│   ├── test         text | audio | listen | listen-debug | regression
│   ├── pipeline     start | stop | status
│   ├── failed       list | get | add | update | remove | promote
│   ├── regression   list | get | add | update | remove
│   ├── logs         list | get | count
│   ├── perf         last | summary | history
│   ├── config       validate | backup | backups | restore | export | import
│   ├── system       doctor | devices | live | ready | info | ping
│   └── event        publish
│
├── talk                                        WORKING (or opal-talk)
│   ├── engines      list | get | update | validate
│   ├── voices       list | get | add | update | remove | validate | test
│   ├── profiles     list | get | add | update | remove | activate
│   ├── generate     TEXT -o FILE [--stream]
│   ├── settings     list | get | set | delete
│   ├── backup       list | create | download | restore | delete
│   │                export | import | validate
│   ├── health
│   ├── info
│   └── ping
│
├── event                                       WORKING (or opal-event)
│   ├── publish      TOPIC --data-json JSON
│   ├── list | get | delete | clear
│   ├── topics
│   ├── rules        list | get | create | update | delete | test
│   ├── actions      list | get | create | update | delete | execute
│   ├── executions   list | get
│   ├── settings     get | update
│   ├── backup       export | validate | import
│   └── health
│
├── look                                        DESIGNED — NOT RUNNABLE
│   ├── capabilities
│   ├── engines      list | show | add | update | remove
│   ├── models       list | show | add | update | remove
│   ├── sources      list | show | add | update | remove | probe | stream
│   ├── pipelines    list | show | add | update | remove | validate
│   ├── profiles     list | show | add | update | remove | activate
│   ├── analyze      image | source
│   ├── sessions     list | show | start | stop | stream | preview
│   ├── jobs         list | show | start | cancel
│   ├── observations list | show
│   ├── media        list | show | upload | remove
│   ├── captures     list | show | capture | download
│   ├── artifacts    list | show | download
│   ├── zones        list | show | add | update | remove
│   ├── rules        list | show | add | update | remove
│   ├── monitors     list | show | add | update | remove
│   ├── recordings   list | show | start | stop
│   ├── settings     get | set | reset
│   └── health
│
└── think                                       FUTURE — SUBJECT TO DESIGN
    ├── providers    list | show | configure
    ├── models       list | show
    ├── generate     text | stream
    ├── sessions     start | show | list | close
    ├── agents       list | show | add | update
    ├── tools        list | show | test
    ├── runs         start | show | watch | cancel
    ├── approvals    list | show | approve | reject
    ├── memory       search | list
    └── health
```

The Look tree above is a **human-friendly overview** of the central contract's resources, not a checked generated command inventory. Think verbs are **illustrative only**.

For exact commands, use the living references:

- [Listen CLI](https://github.com/PatMartin/opal-listen/blob/main/docs/CLI.md)
- [Talk CLI](https://github.com/PatMartin/opal-talk/blob/main/docs/CLI.md)
- [Event CLI](https://github.com/PatMartin/opal-event/blob/main/docs/CLI.md)
- [Central OpenAPI contracts](https://github.com/PatMartin/opal-api/tree/main/openapi)

---

## How OPAL works

### Every service has three doors

```text
        Your terminal               Browser UI               Your app
           (CLI)                       (SPA)                  (script)
             \                           |                       /
              \                          |                      /
               +------------------- REST API ------------------+
                                      |
                               OPAL service
                                      |
                           engines, models, local data
```

**CLI** means command-line interface: type a command. **SPA** means single-page web app: click around in a browser. Both should use the same REST API as any other client.

This is **API-first development**. Define the contract, then build the service and its clients against that contract. No “special magic” hidden in a CLI that other programs cannot use.

### The messages are the glue

OPAL Event uses **ZeroMQ**, a fast way for programs to exchange messages.

```text
Listen ───┐
Sensors ──┼──▶ Event broker ──▶ Subscribers / matching rules
Look* ────┘                           |
                                    Actions
                                    HTTP / shell / Python

* Look events are planned, not currently produced by a live Look service.
```

Event has two sides:

- **REST** for setting up rules, checking history, and publishing controlled events.
- **ZeroMQ** for sending messages to subscribers with little coupling between programs.

A message may be lost if nobody is listening, or if resources run out. If you need guaranteed delivery for an important workflow, add a durable queue or another system that provides those guarantees. An event receipt is not a promise that the door actually locked or the alert actually arrived.

### Learn these six useful words

| Word | Easy meaning | Example |
| --- | --- | --- |
| **Engine** | The software doing the hard work | Piper, Kokoro, Vosk |
| **Model** | The trained data an engine uses | A speech or vision model |
| **Profile** | Saved settings for one kind of job | “Quiet room” microphone profile |
| **Intent** | What a spoken command means | `light_turn_on` |
| **Event** | A message that something happened | `sensor.temperature.reading` |
| **Rule** | A condition that may trigger an approved action | “If temperature > 30°C…” |

Three other words are useful when Look arrives: a **source** supplies images (camera or file), a **pipeline** does a series of image-processing steps, and an **observation** is what a model found. A **session** handles ongoing work; a **job** handles work with an end. A **monitor** represents a long-running desired state in the design.

### Deploy the heavy parts where they belong

```text
Raspberry Pi or small computer             GPU computer (when needed)
+----------------------------+             +--------------------------+
| Microphone → Listen        |             | Look (future)            |
| Talk + speaker/player      |◀── LAN ────▶| Think (future)           |
+----------------------------+             | Larger inference engines |
             |                             +--------------------------+
             └──────────▶ Event ◀──────────────────────┘
```

Use a Pi for jobs a Pi does well. Let a larger machine handle expensive image or language models. You can move a service without teaching your applications a new vocabulary.

### The contract is the map

The source of truth for the public interface is the OpenAPI description:

- [Event 1.4.0](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_event.yaml)
- [Listen 1.4.0 central contract](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_listen.yaml)
- [Talk 1.4.0 central contract](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_talk.yaml)
- [Look 1.4.0 central contract](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_look.yaml)

**Important:** a published contract version is not always the version running inside a service. At this writing, Listen's service-local active YAML reports **1.3.0**, Talk's reports **1.2.0**, and Event's repo identifies its **1.4.0** file as authoritative. Always check the service you have actually started before assuming new contract features work. Look has a central 1.4.0 contract and also a separate [service-repository 1.4.0 design](https://github.com/PatMartin/opal-look/blob/main/opal_look_v1.4.0.yaml); use the central contract for the current suite-level design and reconcile any differences before implementation.

A nice feature of this approach: you can review the promises of a service *before* writing the code.

---

## A closer look at each working service

### Listen: a good listener does not need to be a mind reader

Listen handles wakewords, speech-to-text, intent matching, and named details (slots). It can use local Vosk models, with optional refinement, and offers text, recorded-audio, and live-microphone tests.

```bash
opal listen system devices
opal listen profiles list
opal listen intents phrases light_turn_on
opal listen test listen-debug
opal listen failed list
opal listen perf summary
```

If Listen gets a sentence wrong, inspect failed utterances and recognition logs. Add examples or aliases, then run regression tests. That is more dependable than hoping it gets smarter on its own.

### Talk: a voice with no strings attached

Talk works with installed Piper and Kokoro engines. It manages voice registrations and saved profiles, and produces WAV or raw PCM. You can ask for a completed file or a progressive stream.

```bash
opal-talk engines validate piper
opal-talk voices list
opal-talk profiles list
opal-talk generate "Systems online." -o ready.wav
opal-talk health
```

Talk does not manage speakers or upload model files through its public API. Your playback process handles sound output; the service host needs any referenced voice files.

### Event: the helpful switchboard operator

Event forwards messages, keeps configurable history, matches rules, and runs registered shell, HTTP, or Python actions. It tracks execution outcomes, including failures and dropped jobs.

```bash
opal-event topics
opal-event rules list
opal-event actions list
opal-event executions list --status failed
opal-event health --ready
```

Rules can be tested without action. Actions should be narrowly scoped. Event is the right place to say, “When this trusted message arrives, request that action,” rather than hard-coding every device interaction into your microphone or camera service.

---

## Make it reliable, not merely clever

### Check before you trust

```bash
opal-event health --ready
opal-talk health
opal listen system doctor
```

These commands must be run from the appropriate service environments. **Live** means a service answers; **ready** means it can do the work you need. A process can be alive while its model, audio device, or broker is broken.

### Keep changes safe

Use this order when editing rules, voices, profiles, and service settings:

1. Check what is running, and export or back up important settings.
2. Change one thing at a time.
3. Validate or dry-run where supported.
4. Confirm the actual behavior.
5. Check logs, history, and results; roll back if needed.

OPAL uses revision checking in parts of its API so that two tools do not silently overwrite each other's changes. You may see **ETag** and **If-Match** in the deeper docs. Think of them as “only save this if nobody changed it since I last looked.”

### Follow a problem across services

OPAL's contract design includes session, transaction, request, and causation IDs. They help answer four different questions: Which conversation? Which overall task? Which individual network request? Which earlier event caused this one?

Those IDs are for *tracing*, not for granting permission. Verify actual propagation when joining services with different contract versions.

### Respect the network boundary

Keep development APIs and event sockets on a trusted machine or LAN. Do not expose unauthenticated services directly to the internet. Limit the destinations actions may call, restrict file access, protect secrets, and be careful with cameras and microphones. If a remote vision or LLM provider is enabled, content may leave the local network.

### Tests are part of the product

The projects include Python tests and contract checks. Typical development commands include:

```bash
# Event
cd ~/opal-suite/opal-event
.venv/bin/python -m pytest

# Listen
cd ~/opal-suite/opal-listen
.venv/bin/python -m pytest -m 'not vosk' -q

# Talk
cd ~/opal-suite/opal-talk
make test

# Validate the shared REST contracts (install Node dependencies first).
cd ~/opal-suite/opal-api
npm ci
npm run check
```

The goal is not just “the code runs.” The goal is “the CLI and web app do what the contract promises, errors are clear, and nothing surprises you when something fails.”

---

## What's next?

The road ahead has two tracks. One makes today's services easier to use together. The other adds capabilities that make OPAL a more complete assistant.

**Build the foundation:** align service implementations with published 1.4.0 contracts; create a real shared `opal` command dispatcher; improve cross-service tracing, examples, deployment, and regression tests.

**Give it sight:** implement Look in small, testable slices: register cameras and models, inspect single images, watch live streams, apply motion and annotation filters, save specific evidence by choice, then publish well-defined events.

**Give it judgment:** design and implement Think with model-neutral adapters, bounded tool use, read-only investigation by default, explicit approval before changes, and an audit trail of what happened.

The best future feature is not “AI can do anything.” It is **“AI can help with more things while you can still understand and control it.”**

---

## Where to go deeper

| Want to… | Read this |
| --- | --- |
| Learn the whole speech-recognition CLI | [Listen command reference](https://github.com/PatMartin/opal-listen/blob/main/docs/CLI.md) |
| Set up microphones and Vosk | [Listen quick start](https://github.com/PatMartin/opal-listen/blob/main/docs/QUICKSTART.md) |
| Configure voices, profiles, or streaming | [Talk user guide](https://github.com/PatMartin/opal-talk/blob/main/docs/USER_GUIDE.md) |
| Build Event rules or actions | [Event command reference](https://github.com/PatMartin/opal-event/blob/main/docs/CLI.md) |
| Understand the Event message protocol | [ZeroMQ protocol](https://github.com/PatMartin/opal-event/blob/main/specs/zeromq_protocol.md) |
| Study the current Look design | [Look OpenAPI](https://github.com/PatMartin/opal-api/blob/main/openapi/opal_look.yaml) |
| Check the official REST contracts | [opal-api](https://github.com/PatMartin/opal-api) |
| Explore the repositories | [PatMartin on GitHub](https://github.com/PatMartin?tab=repositories) |

## The short version

You do not have to build a giant assistant on day one.

**Start with a voice. Add ears. Wire up one useful event. Test it. Then add eyes and a cautious brain when they are ready.**

OPAL is about owning the pieces, understanding the behavior, and having fun building something that actually helps.

**Your hardware. Your models. Your rules. Your AI.**

---

*This guide describes the repository state checked on November 1, 2026. Status labels matter: a CLI example marked FUTURE DESIGN is an illustration of the planned interface, not a promise of working software. For exact behavior, the current service code and the matching contract take precedence.*
