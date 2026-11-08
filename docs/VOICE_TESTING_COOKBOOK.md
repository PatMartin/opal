# OPAL Voice Testing Cookbook

**20 ways to find out whether a voice system works—and exactly where it fails.**

Contract snapshot: **November 8, 2026**, [`PatMartin/opal-api@7ecc78a`](https://github.com/PatMartin/opal-api/tree/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi). All five service contracts declare **1.5.0**; the shared kernel declares **1.2.0**.

This is a design for a **hypothetical Typer CLI extrapolated from those contracts**, not documentation of an installed CLI. Command names and client conveniences below are proposed. REST operations, request fields, and response fields are grounded in the snapshot. No recognition tests were executed, and no benchmark results are claimed.

Every executable example uses **only `opal` commands**. JSON is passed directly as command data; there are no shell scripts, pipelines, Python snippets, or external audio utilities. Human speech, physical setup, and previously recorded WAV fixtures are prerequisites where stated. Code fences provide a copy button in supporting Markdown viewers.

## Contents

| # | Application | Main question | Support |
|---|---|---|---|
| [1](#1-check-the-whole-capture-path) | Capture preflight | Is the microphone pipeline actually alive? | Contract |
| [2](#2-separate-hearing-errors-from-understanding-errors) | Fault isolation | Did speech recognition or intent matching fail? | Contract |
| [3](#3-put-whisper-vosk-and-other-recognizers-on-the-same-test) | Recognizer comparison | Which configured backend handles the same recording best? | Deployment |
| [4](#4-build-a-synthetic-speaker-smoke-test) | Synthetic speakers | Does recognition survive different TTS voices and engines? | Contract + installed voices |
| [5](#5-find-the-fast-and-slow-speech-boundaries) | Speech-rate stress | Does recognition collapse when people speak quickly? | Contract |
| [6](#6-test-gain-and-sample-rate-separately) | Audio compatibility | Are quiet audio and low sample rates handled properly? | Contract |
| [7](#7-replay-the-real-world-kitchen-porch-and-far-field) | Environmental robustness | Does the lab winner survive actual rooms? | Recorded fixtures |
| [8](#8-test-speakers-languages-and-code-switching) | Inclusive coverage | Who and which languages does the system struggle with? | Deployment + fixtures |
| [9](#9-make-silence-and-background-speech-fail-safely) | Negative audio | Does non-command audio become an actionable intent? | Recorded fixtures |
| [10](#10-test-the-wakeword-gate-and-near-misses) | Wakeword discrimination | Does a near-match open the gate? | Contract + fixtures |
| [11](#11-attack-negation-cancellation-and-quoted-commands) | Meaning reversals | Does “do not” accidentally become “do”? | Contract |
| [12](#12-catch-the-right-intent-with-the-wrong-slot) | Slot integrity | Did it hear fifteen, fifty, or the wrong room? | Deployment + contract |
| [13](#13-turn-intent-expectations-into-a-regression-suite) | Intent regression | Did a configuration change break known behavior? | Contract gap |
| [14](#14-turn-a-real-failure-into-a-reviewed-improvement) | Feedback loop | Can a miss become a useful training example? | Contract; promotion gap |
| [15](#15-find-the-pause-that-cuts-a-sentence-in-half) | Endpointing | Does hesitation end the utterance too early? | Live protocol |
| [16](#16-compare-pi-gpu-and-refinement-latency-honestly) | Performance tradeoffs | Which deployment meets the accuracy and latency budget? | Deployment |
| [17](#17-test-event-routing-without-performing-the-action) | Dispatch dry run | Would the right event select the right action? | Contract |
| [18](#18-prove-live-dispatch-and-check-duplicate-suppression) | Live integration | Does a live command produce one completed execution? | Deployment + contract |
| [19](#19-use-think-to-invent-tests-not-to-declare-victory) | Adversarial test design | What difficult utterances have we forgotten? | Contract |
| [20](#20-use-vision-to-check-whether-the-command-worked) | Physical outcome verification | Did the requested change happen in the real world? | Deployment + contract |

**Contract** means the underlying operation exists, assuming the projected CLI is implemented. **Deployment** means an adapter, model, configured resource, or integration must already exist. **Contract gap** means the specification cannot fully describe a portable workflow today. None of these labels certify a particular service implementation.

## The projected CLI

Use resource groups such as `opal listen recognition`, `opal say voices`, and `opal event rules`. Use `list`, `show`, `add`, `update`, and `remove` for ordinary resources. Preserve THINK's explicitly advertised singular `task`, `session`, and `model` groups, with `task create` for intelligent work. Python identifiers and JSON keys stay in `snake_case`; command names and flags use hyphens.

These are the mappings used in this cookbook. Paths are relative to the selected service origin.

| Projected CLI family | Actual REST mapping |
|---|---|
| `opal listen recognition test-text` | `POST /api/v1/recognition/test_text` |
| `opal listen recognition test-audio` | `POST /api/v1/recognition/test_audio`, multipart field `file` |
| `opal listen recognition start / stop / status / stream` | Corresponding `/api/v1/recognition/{operation}` routes; status and stream use GET |
| `opal listen intents`, `wakewords`, `slots` | Their collection/item routes; nested phrases, values, and aliases preserve parent IDs |
| `opal listen regression add / list / run` | `POST /api/v1/regression/`, `GET /api/v1/regression/`, `POST /api/v1/regression/run` |
| `opal listen failed-utterances` | `/api/v1/failed_utterances/` and item/promote routes |
| `opal listen performance last / summary / history` | GET on the corresponding `/api/v1/performance/` routes |
| `opal listen logs list / show` | `GET /api/v1/logs/` and `GET /api/v1/logs/{log_id}` |
| `opal SERVICE settings list / update` | `GET /api/v1/settings/` and atomic `PATCH /api/v1/settings/` |
| `opal SERVICE profiles list / show / add / activate` | Shared profile routes as mounted by that service |
| `opal say engines list / validate` | `GET /api/v1/engines/`; `POST /api/v1/engines/{engine_id}/validate` |
| `opal say voices list / show / validate / test` | Corresponding voice collection/item/action routes |
| `opal say generate` | `POST /api/v1/generate` |
| `opal say generate-stream` | `POST /api/v1/generate/stream`, raw PCM response |
| `opal event actions`, `rules`, `events`, `executions` | Their collection/item routes; `rules test` maps to the non-executing test endpoint |
| `opal think task create / watch / show` | `POST /api/v1/tasks/`; GET task stream and task item |
| `opal think model list` | `GET /api/v1/models/` |
| `opal look capabilities show`, `sources list`, `pipelines list` | GET capability/source/pipeline catalogs |
| `opal look analyze run` | `POST /api/v1/analyze/` |
| `opal SERVICE health show / ready` | `GET /api/v1/health/` and `GET /api/v1/health/ready` |
| `opal listen health audio-devices` | `GET /api/v1/health/audio_devices` |

### Client conventions—not extra server capabilities

| Convention | Meaning |
|---|---|
| `--body 'JSON'` | Send the JSON request body unchanged after schema validation. This avoids pretending opaque settings have portable typed flags. |
| `--file PATH` | Upload the local file as the declared multipart field. The server does not need access to the client's path. |
| `--output PATH` | Save the response body locally. The proposed client creates parent directories and keeps diagnostics out of saved binary/JSON data. |
| `--show-headers` | Display response headers separately from the body, including ETag and correlation IDs. |
| Root `--base-url URL` | Override the origin for the one selected service. It does not configure or install a recognizer. |
| Root `--transaction-id ID` | Send `X-Opal-Transaction-Id`. This labels an operation; it is not an atomic transaction or an idempotency key. |
| Root `--context-session-id ID` | Send the logical `X-Opal-Session-Id` header, distinct from a managed THINK session. |
| Boolean flags | `--require-wakeword` / `--no-require-wakeword`, `--save-log` / `--no-save-log`, and `--debug` map to declared query parameters. |
| `--if-match auto` | Proposed client helper: GET the corresponding revision-bearing collection/item, obtain its exact strong ETag, and make one conditional write. Fail if no suitable ETag exists or the write returns 412. Never send the literal word `auto`, use `*`, or silently retry a conflict. |

Use `--if-match '"EXACT_ETAG"'` instead of `auto` when a reviewed revision must be pinned. `auto` reads the current revision immediately before writing; it does not preserve a revision reviewed earlier. A generic generator cannot infer ambiguous revision scopes, particularly promotion. Those gaps are called out below.

CAPITALIZED resource placeholders such as `ROOM_SLOT_ID` must be replaced with actual returned IDs. LISTEN's intent, slot, value, failure, and regression IDs are integers; do not substitute resource names. Names such as `lab.light_on` are intent names, which are explicitly used in recognition results and regression expectations.

### Prepare a disposable voice lab

The client must already know the default LISTEN, SAY, EVENT, THINK, and LOOK service URLs and authentication settings for the services used. Only deploy the services a recipe needs. The contracts provide no portable service launcher, engine installer, microphone recorder-to-file, noise mixer, or audio player.

For comparison recipes, use these **illustrative local test origins**, replacing them with your deployments. Each must expose the OPAL LISTEN API, even when an adapter calls another recognition platform.

| Origin | Example deployment—not a contract guarantee |
|---|---|
| `http://127.0.0.1:8111` | Whisper adapter |
| `http://127.0.0.1:8112` | Vosk adapter |
| `http://127.0.0.1:8113` | faster-whisper adapter |
| `http://127.0.0.1:8114` | Another installed adapter, such as a cloud recognizer |
| `http://127.0.0.1:8121` | A separately configured deployment with second-pass refinement |

Whisper and faster-whisper may use related model weights through different runtimes. Record the actual model, runtime, quantization, language/task settings, hardware, and refinement configuration before interpreting a difference. Cloud endpoints are not automatically OPAL-compatible. Adapter installation and credentials are outside these recipes.

For most examples, prepare these harmless recognition targets on the default LISTEN lab instance. They recognize text only; creating an intent does not create a device action.

```bash
opal listen settings list --output results/listen_settings_before.json
opal listen profiles list --output results/listen_profiles_before.json
opal listen settings update --body '{"event_publication_enabled":false}' --if-match auto
opal listen intents add --body '{"name":"lab.light_on","description":"Voice lab positive command","phrases":["turn on the desk light"]}' --if-match auto
opal listen intents add --body '{"name":"lab.light_off","description":"Voice lab opposite command","phrases":["turn off the desk light"]}' --if-match auto
opal listen intents add --body '{"name":"lab.cancel","description":"Voice lab cancellation","phrases":["cancel that","do not turn on the desk light"]}' --if-match auto
opal listen wakewords add --body '{"name":"opal","enabled":true,"confidence_threshold":0.8}' --if-match auto
opal listen profiles add --body '{"name":"voice_lab","is_active":true,"wakeword_names":["opal"],"enabled_intents":["lab.light_on","lab.light_off","lab.cancel"]}' --if-match auto
```

Run this setup once in a fresh lab, not blindly against an existing configuration. Reuse returned resources on subsequent runs. Prepare equivalent intent/profile definitions on comparison deployments before scoring intent behavior. The shared profile schema carries service-defined semantics, so verify the active profile with `recognition status` and text probes. Save the old active profile ID if this is not a disposable instance.

Fixtures under `fixtures/` are local WAV recordings supplied by the tester, unless a SAY command explicitly creates them. Their filenames describe intended content; they do not prove it. Check the recording and reference transcript. Treat recorded speech, synthetic speech, and live microphone trials as separate test populations.

## 1. Check the whole capture path

**Problem:** A service can be healthy while the microphone is wrong, silent, or overflowing.

Inspect devices and pipeline state, then observe a short live trial. In the streaming terminal, speak “Opal, turn on the desk light.” End the stream with the terminal's interrupt gesture before issuing the remaining commands, or use a second terminal.

```bash
opal listen health show
opal listen health ready
opal listen health audio-devices
opal listen recognition start
opal listen recognition status
opal listen recognition stream --debug
opal listen recognition status --output results/capture_status.json
opal listen recognition stop
```

**Check:** The intended `audio_device`, plausible `capture_sample_rate`, increasing `frames_processed`, live `audio_level` events, and a finalized `result`. Inspect `last_error` and metrics such as `mic_overflows`, `dropped_audio_blocks`, and `audio_queue_utilization`. A readiness response alone does not establish successful capture.

**Boundary:** Device enumeration exists; a portable typed microphone-selection setting does not. Select the device using the deployment's documented settings. An SSE connection sees future events only and can lose queued events if its consumer is slow.

## 2. Separate hearing errors from understanding errors

**Problem:** “The voice command failed” combines at least two different failures.

Use a recording whose reviewed reference is exactly “turn on the desk light.” Test the intended text independently from the recording.

```bash
opal listen recognition test-text --body '{"text":"turn on the desk light"}' --output results/isolation_reference.json
opal listen recognition test-audio --file fixtures/desk_light_clean.wav --no-require-wakeword --output results/isolation_audio.json
opal listen performance last
opal listen logs list --limit 10
```

If the audio result's transcript is wrong, copy its actual transcript into a second text probe. This example illustrates a possible substitution; replace it with the observed text.

```bash
opal listen recognition test-text --body '{"text":"turn on the desktop light"}' --output results/isolation_observed_text.json
```

**Check:** Correct reference text but wrong audio transcript points toward capture/STT. Correct transcript but wrong `intent_name` or `slot_values` points toward downstream interpretation. An incorrect transcript that still triggers the intended action remains a transcription error.

**Boundary:** `test-text` bypasses STT. It cannot establish Whisper accuracy. Both diagnostic endpoints are evaluated locally and never publish external actionable events.

## 3. Put Whisper, Vosk, and other recognizers on the same test

**Problem:** Comparing different recordings, models, or intent grammars produces a misleading winner.

Preconfigure the adapters and identical downstream intent definitions. Upload the **same bytes** to each instance. Here, wakeword gating is disabled to isolate STT plus intent interpretation.

```bash
opal --base-url http://127.0.0.1:8111 listen recognition status
opal --base-url http://127.0.0.1:8112 listen recognition status
opal --base-url http://127.0.0.1:8113 listen recognition status
opal --base-url http://127.0.0.1:8114 listen recognition status
opal --base-url http://127.0.0.1:8111 listen recognition test-audio --file fixtures/desk_light_clean.wav --no-require-wakeword --output results/whisper_clean.json
opal --base-url http://127.0.0.1:8112 listen recognition test-audio --file fixtures/desk_light_clean.wav --no-require-wakeword --output results/vosk_clean.json
opal --base-url http://127.0.0.1:8113 listen recognition test-audio --file fixtures/desk_light_clean.wav --no-require-wakeword --output results/faster_whisper_clean.json
opal --base-url http://127.0.0.1:8114 listen recognition test-audio --file fixtures/desk_light_clean.wav --no-require-wakeword --output results/other_clean.json
```

**Check:** Transcript errors, exact intent/slot results, `stt_latency_ms`, `total_latency_ms`, and any `refinement_error`. Provider-specific confidence scores are not interchangeable measures of accuracy. Repeat across the same corpus; one clip is a smoke test.

**Boundary:** LISTEN mounts neither an engine catalog nor a per-request `engine_id`/`model_id` override. A command such as `opal listen engines install whisper` cannot be derived from these contracts. An arbitrary settings map is not proof that a guessed `stt_engine` setting exists. This recipe is a common API comparison over prepared instances, not a portable backend installer or selector.

## 4. Build a synthetic speaker smoke test

**Problem:** The assistant recognizes one convenient demonstration voice but fails another.

Choose two registered voices from different installed engines where available—for example, Piper and Kokoro. Replace the IDs below with the returned voice IDs. SAY voice registration references host-resident resources; it does not install or upload models.

```bash
opal say engines list
opal say voices list
opal say voices validate VOICE_A_ID
opal say voices validate VOICE_B_ID
opal say generate --body '{"text":"turn on the desk light","voice_id":"VOICE_A_ID","output_format":"wav","sample_rate":16000}' --output fixtures/synthetic_a.wav
opal say generate --body '{"text":"turn on the desk light","voice_id":"VOICE_B_ID","output_format":"wav","sample_rate":16000}' --output fixtures/synthetic_b.wav
opal listen recognition test-audio --file fixtures/synthetic_a.wav --output results/synthetic_a.json
opal listen recognition test-audio --file fixtures/synthetic_b.wav --output results/synthetic_b.json
```

**Check:** Both should preserve the reference words and identify `lab.light_on`. The voice normally selects its registered engine; do not force a Piper voice through Kokoro with an incompatible engine override.

**Insight:** Synthetic round trips are excellent repeatable integration probes. They are weak evidence of performance on children, accents, hesitation, microphones, or rooms. The synthesizer can also mispronounce a word, so review the audio before assigning all errors to STT.

## 5. Find the fast and slow speech boundaries

**Problem:** Recognition works at demonstration speed but fails under time pressure.

Keep voice, words, gain, and sample rate fixed. Change only SAY's portable `speed` control.

```bash
opal say generate --body '{"text":"turn off the desk light","voice_id":"VOICE_A_ID","speed":0.75,"volume":1.0,"output_format":"wav","sample_rate":16000}' --output fixtures/rate_slow.wav
opal say generate --body '{"text":"turn off the desk light","voice_id":"VOICE_A_ID","speed":1.0,"volume":1.0,"output_format":"wav","sample_rate":16000}' --output fixtures/rate_normal.wav
opal say generate --body '{"text":"turn off the desk light","voice_id":"VOICE_A_ID","speed":1.5,"volume":1.0,"output_format":"wav","sample_rate":16000}' --output fixtures/rate_fast.wav
opal listen recognition test-audio --file fixtures/rate_slow.wav --output results/rate_slow.json
opal listen recognition test-audio --file fixtures/rate_normal.wav --output results/rate_normal.json
opal listen recognition test-audio --file fixtures/rate_fast.wav --output results/rate_fast.json
```

**Check:** Preserve the crucial word “off,” not merely a high intent confidence. Establish a usable rate range for each voice and deployed recognizer.

**Boundary:** A synthesis speed multiplier is not a calibrated words-per-minute measurement. Follow with real fast and hesitant speakers; synthetic speed changes do not reproduce human articulation changes.

## 6. Test gain and sample rate separately

**Problem:** Quiet input and low sample rate can look like the same bug when both are changed at once.

Create a baseline and two single-variable variants. The service must honor an explicitly requested SAY sample rate or reject it.

```bash
opal say generate --body '{"text":"turn on the desk light","voice_id":"VOICE_A_ID","volume":1.0,"sample_rate":16000,"output_format":"wav"}' --show-headers --output fixtures/audio_baseline.wav
opal say generate --body '{"text":"turn on the desk light","voice_id":"VOICE_A_ID","volume":0.15,"sample_rate":16000,"output_format":"wav"}' --show-headers --output fixtures/audio_quiet.wav
opal say generate --body '{"text":"turn on the desk light","voice_id":"VOICE_A_ID","volume":1.0,"sample_rate":8000,"output_format":"wav"}' --show-headers --output fixtures/audio_8khz.wav
opal listen recognition test-audio --file fixtures/audio_baseline.wav --output results/audio_baseline.json
opal listen recognition test-audio --file fixtures/audio_quiet.wav --output results/audio_quiet.json
opal listen recognition test-audio --file fixtures/audio_8khz.wav --output results/audio_8khz.json
```

**Check:** The returned `X-Opal-Sample-Rate`, recognition results, and explicit errors for unsupported input. A clean validation failure is preferable to an accepted file decoded incorrectly.

**Boundary:** Lower gain on a clean digital signal is not the same as lower signal-to-noise ratio. An 8 kHz WAV is not a complete telephone-channel simulation. SAY streaming produces headerless PCM; LISTEN's file test documents WAV. Renaming streamed PCM to `.wav` does not make a valid fixture.

## 7. Replay the real world: kitchen, porch, and far field

**Problem:** Quiet-room success says little about a refrigerator, porch fan, television, or microphone across the room.

Prepare reviewed recordings of the same target phrase under the named conditions. Record distance, microphone position, and noise conditions with the fixture collection. Reuse each recording across recognizers.

```bash
opal listen recognition test-audio --file fixtures/rooms/quiet_near.wav --output results/room_quiet_near.json
opal listen recognition test-audio --file fixtures/rooms/kitchen_appliance.wav --output results/room_kitchen.json
opal listen recognition test-audio --file fixtures/rooms/porch_fan.wav --output results/room_porch.json
opal listen recognition test-audio --file fixtures/rooms/across_room.wav --output results/room_far.json
opal listen recognition test-audio --file fixtures/rooms/tv_competing_speech.wav --output results/room_tv.json
```

**Check:** Score each environment separately. A system that works beautifully near the microphone may still be unsuitable for hands-free use across the room. Inspect both missed commands and unwanted interpretations of competing speech.

**Boundary:** No contract endpoint mixes noise, simulates reverberation, or measures SNR. These commands replay supplied WAV fixtures. Creating realistic perturbations would require an additional fixture tool or a proposed `opal` client feature.

## 8. Test speakers, languages, and code-switching

**Problem:** An average accuracy score can hide an assistant that fails particular users.

Use voluntarily supplied, reviewed recordings from the people expected to use the system. Include different accents and speaking styles. For multilingual tests, first configure a recognizer that supports the language and an intent grammar that accepts it; the current API does not standardize language selection.

```bash
opal listen recognition test-audio --file fixtures/speakers/speaker_a.wav --no-save-log --output results/speaker_a.json
opal listen recognition test-audio --file fixtures/speakers/speaker_b.wav --no-save-log --output results/speaker_b.json
opal listen recognition test-audio --file fixtures/speakers/speaker_c.wav --no-save-log --output results/speaker_c.json
opal listen recognition test-text --body '{"text":"enciende la luz del escritorio"}' --no-save-log --output results/spanish_text_control.json
opal listen recognition test-audio --file fixtures/languages/es_desk_light.wav --no-save-log --output results/spanish_audio.json
opal listen recognition test-audio --file fixtures/languages/code_switch_desk_light.wav --no-save-log --output results/code_switch.json
```

**Check:** Compare the Spanish transcript to a Spanish reference, then separately assess intent interpretation. An English translation can be meaningful while still being the wrong output for a transcription test. Use the same task and language policy for every recognizer. Report per-speaker and per-language results rather than hiding failures in a pooled average.

**Boundary:** `--no-save-log` requests no recognition-log storage. It is not a universal privacy switch: local streaming, explicitly saved result files, adapter behavior, and any upstream provider retention are separate concerns. This test measures the system, not a person's ability.

## 9. Make silence and background speech fail safely

**Problem:** A recognizer that eagerly invents a command can be worse than one that asks for clarification.

Supply recorded silence, room tone, music, unrelated conversation, and a television quotation containing a command. These are negative cases with reviewed expected behavior.

```bash
opal listen recognition test-audio --file fixtures/negative/silence.wav --output results/negative_silence.json
opal listen recognition test-audio --file fixtures/negative/room_tone.wav --output results/negative_room_tone.json
opal listen recognition test-audio --file fixtures/negative/music.wav --output results/negative_music.json
opal listen recognition test-audio --file fixtures/negative/unrelated_conversation.wav --output results/negative_conversation.json
opal listen recognition test-audio --file fixtures/negative/tv_command_without_wakeword.wav --require-wakeword --output results/negative_tv.json
```

**Check:** Distinguish an invented transcript from a false intent acceptance. Silence and non-command noise should not create an accepted command. Legitimate background speech may correctly produce text while still requiring rejection as an action request.

**Insight:** Measure false accepts per negative clip; for live trials, also report false activations per hour. “High match rate” can be a serious defect on this dataset. There is no native expected-`no_match` regression field in the current text-regression schema.

## 10. Test the wakeword gate and near-misses

**Problem:** Similar names, incomplete wakewords, or background speech can open a command session unexpectedly.

Use four reviewed fixtures: a command without a wakeword; “Opal” plus the command; a near-miss such as “Opel” plus the command; and the wakeword alone.

```bash
opal listen wakewords list
opal listen recognition test-audio --file fixtures/wake/no_wakeword.wav --no-require-wakeword --output results/wake_bypass_control.json
opal listen recognition test-audio --file fixtures/wake/no_wakeword.wav --require-wakeword --output results/wake_missing.json
opal listen recognition test-audio --file fixtures/wake/opal_command.wav --require-wakeword --output results/wake_positive.json
opal listen recognition test-audio --file fixtures/wake/opel_command.wav --require-wakeword --output results/wake_near_miss.json
opal listen recognition test-audio --file fixtures/wake/wakeword_only.wav --require-wakeword --output results/wake_only.json
```

**Check:** The missing-wakeword case should produce no match when the gate is required. The positive case should reach the intended command. Record near-miss false accepts and positive false rejects separately. Wakeword-only input must not become a fabricated command.

**Boundary:** The file endpoint provides a wakeword-required check. Real-time candidate confirmation and timing still require the live SSE protocol: `wakeword_candidate`, `wakeword_confirmed`, and `wakeword_rejected`. A candidate event alone is not a confirmed command.

## 11. Attack negation, cancellation, and quoted commands

**Problem:** Matching the important nouns while ignoring “do not” reverses the user's meaning.

The lab setup provides explicit on, off, and cancel intents. Test these alongside more difficult phrases that were not used as training examples.

```bash
opal listen recognition test-text --body '{"text":"turn on the desk light"}' --output results/meaning_on.json
opal listen recognition test-text --body '{"text":"turn off the desk light"}' --output results/meaning_off.json
opal listen recognition test-text --body '{"text":"do not turn on the desk light"}' --output results/meaning_negative.json
opal listen recognition test-text --body '{"text":"I was going to ask you to turn on the desk light, but cancel that"}' --output results/meaning_reversal.json
opal listen recognition test-text --body '{"text":"the television said turn on the desk light"}' --output results/meaning_quotation.json
opal listen recognition test-audio --file fixtures/meaning/do_not_turn_on.wav --output results/meaning_negative_audio.json
```

**Check:** The trained negative should select `lab.cancel`; the reversal should cancel or reject according to your declared policy. The quotation should not select the on action. Define acceptable outcomes before running the test. This contract is not a guarantee that an intent matcher understands conversational negation.

**Harsh lesson:** A lower confidence threshold can improve your demonstration while increasing precisely the false accepts that make a voice assistant untrustworthy. Do not optimize match rate by itself.

## 12. Catch the right intent with the wrong slot

**Problem:** `set_timer` with fifty minutes instead of fifteen is still a failure. So is controlling the wrong room.

**Prerequisite:** A configured timer intent with a documented numeric slot, plus a room-based light intent bound to an existing room slot. Include them in the active profile where required. LISTEN exposes slots and aliases, but these contracts do not define the phrase-template grammar or portable numeric-slot semantics; this setup cannot honestly be generated from the schema alone.

```bash
opal listen slots list
opal listen intents list
opal listen slots values add ROOM_SLOT_ID --body '{"value":"living_room"}' --if-match auto
opal listen slots values aliases add ROOM_SLOT_ID ROOM_VALUE_ID --body '{"alias":"lounge"}' --if-match auto
opal listen recognition test-text --body '{"text":"set a timer for fifteen minutes"}' --output results/slot_fifteen_text.json
opal listen recognition test-text --body '{"text":"set a timer for fifty minutes"}' --output results/slot_fifty_text.json
opal listen recognition test-audio --file fixtures/slots/fifteen_minutes.wav --output results/slot_fifteen_audio.json
opal listen recognition test-audio --file fixtures/slots/fifty_minutes.wav --output results/slot_fifty_audio.json
opal listen recognition test-text --body '{"text":"turn on the lounge light"}' --output results/slot_alias.json
```

Use the ID returned by the value creation as `ROOM_VALUE_ID`; reuse an existing canonical value if it already exists. **Check:** Exact numeric values, units, and the configured canonical room value in `slot_values`, not just `matched=true`. Add homophones, decimals, dates, and unfamiliar names appropriate to the application.

## 13. Turn intent expectations into a regression suite

**Problem:** Adding a phrase can improve one utterance while breaking a neighboring intent.

The native regression feature stores **text phrases**, expected intents, and expected slots. Use it to protect intent behavior.

```bash
opal listen regression list --show-headers
opal listen regression add --body '{"name":"desk_light_on","phrase":"turn on the desk light","expected_intent":"lab.light_on","expected_slots":{}}' --if-match auto
opal listen regression add --body '{"name":"desk_light_off","phrase":"turn off the desk light","expected_intent":"lab.light_off","expected_slots":{}}' --if-match auto
opal listen regression add --body '{"name":"explicit_cancel","phrase":"do not turn on the desk light","expected_intent":"lab.cancel","expected_slots":{}}' --if-match auto
opal listen regression run --enabled-only --output results/intent_regression.json
```

**Contract gap:** Regression writes require `If-Match`, but the regression collection and item GET responses do not declare an ETag. These creation commands require a deployment that supplies a documented, usable revision. Otherwise the `auto` helper must stop. The portable fix belongs in the contract; fabricating an ETag is not a workaround. Running an already populated suite remains a defined operation.

**Check:** `total`, `passed`, `failed`, and individual `failure_reason` values. The schema does not document whether `pass_rate` uses 0–1 or 0–100; use the counts for unambiguous reporting. The client would need an explicitly defined exit-code policy before this becomes a CI gate.

**Boundary:** These cases do not exercise WAV decoding, STT, microphones, or wakewords. A 100% text-regression result is not a 100% voice-recognition result. There is no audio fixture field or native WER metric here.

## 14. Turn a real failure into a reviewed improvement

**Problem:** Repeated production misses are noticed but never become controlled learning material.

Find a failed recognition, review its actual transcript and intended meaning, then create a review record. Replace `FAILED_LOG_ID` with the integer ID in the returned result; replace the example transcript with the observed one.

```bash
opal listen logs list --no-matched --limit 20
opal listen logs show FAILED_LOG_ID
opal listen failed-utterances add --body '{"transcript":"light up my desk","expected_intent":"lab.light_on","notes":"Reviewed paraphrase; reference audio checked"}'
opal listen failed-utterances show FAILED_UTTERANCE_ID
opal listen failed-utterances update FAILED_UTTERANCE_ID --body '{"reviewed":true,"expected_intent":"lab.light_on"}'
```

Copy the new record's returned ID into `FAILED_UTTERANCE_ID`. The optional integer `recognition_log_id` can also be supplied when creating the record to preserve the link to the actual miss.

Where the implementation documents the promotion revision, the intended compact workflow is:

```bash
opal listen failed-utterances promote FAILED_UTTERANCE_ID --intent-name lab.light_on --create-regression --if-match '"PROMOTION_ETAG"'
opal listen regression run --output results/after_promotion.json
```

**Contract gap:** Promotion requires `If-Match`, but the failed-utterance read has no ETag and the promotion operation does not identify the authoritative revision for its phrase-plus-regression mutation. `PROMOTION_ETAG` is a placeholder for an implementation-documented token, not a usable literal. The review-state update itself is explicitly exempt from `If-Match`.

If promotion revision handling is unavailable, use the clearly defined phrase collection instead, then rerun any existing suite:

```bash
opal listen intents phrases add LIGHT_ON_INTENT_ID --body '{"phrase":"light up my desk"}' --if-match auto
opal listen recognition test-text --body '{"text":"light up my desk"}'
opal listen regression run
```

**Check:** Replay the original audio separately and test neighboring/negative phrases. The promoted phrase becomes training material; it is no longer independent holdout evidence. Do not promote a misheard transcript that changes the user's meaning.

## 15. Find the pause that cuts a sentence in half

**Problem:** “Turn on ... the desk light” becomes a partial command because endpointing treats hesitation as completion.

Observe live capture while a tester says the same command with short, medium, and long natural pauses. Include a self-correction: “Turn on—no, turn off—the desk light.” Record the planned pause durations separately; the API does not generate those pauses.

```bash
opal listen profiles list
opal listen profiles show BASELINE_PROFILE_ID
opal listen recognition start
opal listen recognition stream --debug
opal listen recognition status --output results/pause_status.json
opal listen recognition stop
```

After ending the stream, repeat with an already configured alternate endpointing profile:

```bash
opal listen profiles activate LONG_PAUSE_PROFILE_ID --if-match auto
opal listen recognition start
opal listen recognition stream --debug
opal listen recognition stop
opal listen profiles activate BASELINE_PROFILE_ID --if-match auto
```

**Check:** `speech_start`, partial transcripts, `speech_end.reason`, finalized results, and `session_end`. Separate premature cutoff from waiting too long. A `max_length` termination deserves different analysis from `speech_end` or `stt_endpoint`.

**Boundary:** `vad_settings` is an open object with no portable key definitions. Do not pretend `--silence-timeout-ms` is generated from this schema. The protocol does not expose a standardized timestamp on every speech boundary or a barge-in/cancel-playback control.

## 16. Compare Pi, GPU, and refinement latency honestly

**Problem:** A more accurate backend can still feel unusable if it answers too late or drops audio under load.

Replace the origins with isolated deployments on the hardware being compared. A second-pass instance must already have its refiner configured. Start with the same fixture; then repeat the trial set and evaluate live operation separately.

```bash
opal --base-url http://127.0.0.1:8111 listen settings list --output results/direct_settings.json
opal --base-url http://127.0.0.1:8121 listen settings list --output results/refined_settings.json
opal --base-url http://127.0.0.1:8111 listen recognition test-audio --file fixtures/desk_light_clean.wav --output results/direct_probe.json
opal --base-url http://127.0.0.1:8121 listen recognition test-audio --file fixtures/desk_light_clean.wav --output results/refined_probe.json
opal --base-url http://127.0.0.1:8111 listen performance summary --limit 100 --output results/direct_summary.json
opal --base-url http://127.0.0.1:8121 listen performance summary --limit 100 --output results/refined_summary.json
opal --base-url http://127.0.0.1:8121 listen recognition status --output results/refined_runtime.json
```

**Check:** The returned summary `count`, latency `p50`/`p95`, accuracy on the same references, and whether refinement actually ran on the tested path. LISTEN specifies asynchronous bounded refinement and draft fallback on error, timeout, or a full queue. During a separately observed live trial, inspect `refinement_completed`, `refinement_failed`, queue depth, dropped blocks, and `refinement_error` in finalized results.

**Boundary:** `--limit 100` requests the most recent observations; it does not run 100 tests or isolate a named benchmark. There is no run-ID filter here. Avoid concurrent unrelated traffic, retain individual result files, and disclose contamination. Per-run pipeline metrics reset on a new run; do not assume performance history resets with them. Server stage latency is not network latency, time to first audible response, or total user-perceived delay. No standardized fault-injection or load-generator command exists.

## 17. Test event routing without performing the action

**Problem:** The correct transcript can still be routed to the wrong downstream action.

Register a harmless HTTP probe action and a test-only rule. **Prerequisite:** EVENT must already allow the fixed LISTEN health origin below and be able to authenticate to it if required. Replace the origin with the lab service's reachable address; loopback refers to the EVENT host/container, not necessarily the CLI computer.

```bash
opal event actions add --body '{"id":"voice_lab_probe","type":"http","enabled":true,"description":"Read the lab LISTEN health endpoint; no device action","method":"GET","url":"http://127.0.0.1:8111/api/v1/health/","body_mode":"none","timeout_seconds":5}' --if-match auto
opal event rules add --body '{"id":"voice_lab_dry_run","enabled":true,"topic":"voice_lab.command","conditions":[{"path":"data.intent_name","operator":"eq","value":"lab.light_on"}],"actions":["voice_lab_probe"]}' --if-match auto
opal event rules test voice_lab_dry_run --body '{"event":{"topic":"voice_lab.command","data":{"intent_name":"lab.light_on"}}}'
opal event rules test voice_lab_dry_run --body '{"event":{"topic":"voice_lab.command","data":{"intent_name":"lab.light_off"}}}'
opal event rules test voice_lab_dry_run --body '{"event":{"topic":"voice_lab.command","data":{}}}'
```

**Check:** The positive case selects `voice_lab_probe`; the opposite intent and missing field do not. Inspect `matched`, `action_ids`, and `checks`. Rule testing does not execute the action.

**Boundary:** `voice_lab.command` and its payload are a deliberately defined test event. Do not assume they are the live LISTEN event schema. These tests validate a routing predicate, not the microphone pipeline, destination reachability, or successful action execution.

## 18. Prove live dispatch and check duplicate suppression

**Problem:** Diagnostic recognition succeeds, but no real event is dispatched—or a single utterance produces repeated actions.

Reuse the harmless probe from recipe 17. In an isolated EVENT instance, create a temporary rule for LISTEN's documented live topic. Empty conditions intentionally accept every live match during this lab test; this avoids guessing undocumented payload fields.

**Prerequisite:** LISTEN's selected HTTP or ZeroMQ publication transport is already wired to this EVENT instance. Use one transport; the contract rejects `both`. Record the original publication setting, transport, deduplication window, and any existing rules before starting.

```bash
opal event rules add --body '{"id":"voice_lab_live","enabled":true,"topic":"opal.listen.intent.match","conditions":[],"actions":["voice_lab_probe"]}' --if-match auto
opal listen settings list --output results/live_settings_before.json
opal listen settings update --body '{"event_publication_enabled":true}' --if-match auto
opal listen recognition start
opal listen recognition stream --debug
```

Speak the same recognized command twice inside the configured `dedup_window_ms`, then once outside it. End the stream and inspect the retained evidence.

```bash
opal listen recognition status --output results/live_dispatch_status.json
opal event events list --topic opal.listen.intent.match --limit 20
opal event executions list --action-id voice_lab_probe --limit 20
opal event executions show EXECUTION_ID
opal listen recognition stop
opal listen settings update --body '{"event_publication_enabled":false}' --if-match auto
opal event rules remove voice_lab_live --if-match auto
```

**Check:** Live `source=mic` matches reach EVENT; matches marked `deduplicated` are not republished. Distinguish separately intended repeats from duplicate suppression using your declared window. Match executions to the specific returned event IDs and verify terminal `succeeded` status. Multiple rules can otherwise explain multiple executions.

**Boundary:** The final setting restores the lab's original disabled state; restore your actual prior value if different. `test-audio` never publishes externally, even with publication enabled. EVENT is best effort: HTTP 202 means admission, not delivery or successful execution. Its manual execute path can even return a retained `dropped` execution after queue saturation. This is not an exactly-once guarantee. Do not automatically retry physical actions based only on missing observations.

## 19. Use THINK to invent tests, not to declare victory

**Problem:** Developers keep testing the same obvious phrases and miss entire classes of failures.

Ask a configured model for candidate tests. Use a THINK lab with no enabled side-effecting actions for this writing task; `max_steps=1` is a budget, not a tool-denial policy. No managed session is required.

```bash
opal think model list
opal think task create --body '{"prompt":"Propose 12 difficult voice-test utterances for desk-light on, off, and cancel intents. Include negation, quotation, hesitation, self-correction, background speech, and ambiguous requests. For each, suggest an expected intent or rejection and explain the ambiguity. Produce test data only. Do not execute actions or claim measurements.","max_steps":1,"timeout_seconds":60,"metadata":{"evalset":"voice_lab_candidates","case_id":"adversarial_design"}}'
opal think task watch TASK_ID
opal think task show TASK_ID --output results/candidate_utterances.json
```

Replace `TASK_ID` with the returned task ID. Review the authoritative answer after the task reaches `completed`; a streamed fragment is not the final answer. Metadata is tracking data and is not model-visible context.

Choose candidates, define expected outcomes yourself, then test them. These are illustrative manually reviewed candidates, not claimed outputs from a model run:

```bash
opal listen recognition test-text --body '{"text":"please leave the desk light off"}' --output results/adversarial_leave_off.json
opal listen recognition test-text --body '{"text":"do you remember when I asked you to turn on the desk light"}' --output results/adversarial_quoted_history.json
```

**Check:** Useful coverage and failure discovery. A model's proposed expected label is a suggestion requiring review. Do not let the model both invent its own tests and certify a perfect score. Keep a stable human-labeled holdout set. THINK has no guaranteed automatic access to these local result files; pass actual data explicitly if later asking it to analyze failures.

## 20. Use vision to check whether the command worked

**Problem:** Correct intent and successful HTTP execution still do not prove the light actually changed.

**Prerequisite:** A lab lamp, camera source, and validated LOOK pipeline capable of distinguishing that lamp's on/off state; a separately configured live voice-to-lamp action; and an isolated observation view. A generic object detector that identifies “lamp” is not automatically a lamp-state detector. Recipe 18's health probe does not control the lamp.

Take a before observation, run the live voice trial, then inspect the action result and an after observation. Replace the source/pipeline IDs with actual resources.

```bash
opal look capabilities show
opal look sources list
opal look pipelines list
opal --context-session-id lamp_trial_01 --transaction-id lamp_before_01 look analyze run --body '{"source":{"source_id":"LAMP_CAMERA_ID"},"pipeline_id":"LAMP_STATE_PIPELINE_ID"}' --output results/lamp_before.json
opal listen recognition start
opal listen recognition stream --debug
```

Speak the command, retain the finalized recognition and associated live event IDs, then end the stream. Wait for the corresponding lamp action's terminal result before the after observation; issuing an analyze request immediately after admission would introduce a race.

```bash
opal event executions show LAMP_EXECUTION_ID
opal --context-session-id lamp_trial_01 --transaction-id lamp_after_01 look analyze run --body '{"source":{"source_id":"LAMP_CAMERA_ID"},"pipeline_id":"LAMP_STATE_PIPELINE_ID"}' --output results/lamp_after.json
opal listen recognition stop
```

**Check:** Separate five claims: the right words were recognized; the intended action was selected; it executed; the relevant lamp visibly changed; the change occurred after the command. Inspect LOOK `observed_at`, `source_id`, `pipeline_revision`, observations, and task-specific scores. Keep lighting/exposure changes and unrelated motion from masquerading as success.

**Boundary:** This is a manual composite acceptance test. The contracts do not provide an automatic cross-service assertion engine or a shared clock guarantee. The two explicit context headers label the LOOK requests; they do not retroactively label a microphone event or prove causation. Join the live recognition, event, and execution evidence explicitly. A visual observation is an additional check, not an infallible ground truth.

## What to measure

These recipes produce evidence. The current contracts do not turn all that evidence into an audio benchmark automatically.

| Metric | Definition or discipline | Current support |
|---|---|---|
| Word error rate (WER) | `(substitutions + deletions + insertions) / reference_word_count`; freeze tokenization and normalization | Requires an additional scorer; do not divide by zero on silent references |
| Character error rate | Character-level errors with an explicit normalization policy | Requires an additional scorer; useful when word tokenization is unsuitable |
| Intent accuracy | Correct expected intent or declared rejection divided by labeled cases | Native text regression for positive expected intents; manual/external evaluation for audio and rejection cases |
| Slot correctness | Exact required values, canonical identifiers, and units | Returned `slot_values`; native text regression supports `expected_slots` |
| False acceptance | Negative cases incorrectly accepted divided by negative cases | Outcomes available; corpus labels and aggregation missing |
| Wakeword false rejection | Valid gated commands incorrectly rejected divided by valid gated commands | Diagnostic outcomes and live events; curated labels needed |
| Live false activations | False activations divided by observed background-audio hours | Duration accounting and complete capture need a test harness |
| Latency | Stage and total p50/p95 with sample count; distinguish cold/warm runs | LISTEN latency summaries exist; no benchmark-run partition |
| Delivery and execution | Selected action, event linkage, terminal status, duplicates, drops | EVENT records; admission is not completion |
| Physical success | Requested observable state change after the completed action | LOOK can supply observations when a suitable pipeline exists |

Choose pass/fail thresholds **before** testing, based on the use case. Report denominators, individual failures, timeouts, rejected inputs, and unavailable backends. Never exclude failed requests just to improve averages. A small difference on a tiny corpus is not evidence of a general winner.

For a serious comparison, freeze a human-labeled holdout corpus, preserve identical audio per recognizer, separate calibration from evaluation, record software/model/hardware versions, and keep synthetic samples separate. A 20-recipe cookbook is a coverage map, not a statistically sufficient benchmark.

## Harsh but constructive contract critique

### 1. LISTEN is an intent workbench, not yet an ASR evaluation platform

The diagnostic audio endpoint is useful, but the regression schema has `phrase`, `expected_intent`, and `expected_slots`—no audio asset, reference transcript, expected rejection, or immutable test-run result. Calling this a voice regression system would overstate what it tests.

**Smallest valuable extension:** Add versioned audio test cases, immutable runs/results, explicit expected rejection, and reproducible WER/CER scoring. Keep text-intent regression as its own mode. Include invalid-input and timeout outcomes in the result model.

### 2. “Implementation-neutral” needs discoverable, selectable implementations

SAY and LOOK expose engine catalogs; LISTEN does not mount the shared engine paths. A generic settings map and `model_paths` object cannot tell a generated CLI which recognizers exist, which models are loaded, or what valid options mean.

**Fix:** Define LISTEN adapter/model discovery and effective configuration. Prefer request-scoped engine/model/profile overrides for test calls so a benchmark does not keep switching global active state. Return the resolved choices with each result. Keep installation optional and separate from runtime inference APIs.

### 3. Some conditional writes have no portable source for the condition

LISTEN regression writes require `If-Match`, but its regression GETs do not declare ETags. Failure promotion also requires `If-Match` without defining which revision covers the mutation. A generated client cannot responsibly guess.

**Fix:** Declare the revision-bearing reads and revision scope. State how promotion handles its phrase and optional regression changes together. Missing and stale preconditions already have error conventions; complete the successful workflow.

### 4. The shared profile schema mixes unrelated service concepts

The kernel puts SAY voice/gain fields, LISTEN wakeword/VAD fields, and LOOK pipeline fields into one open profile shape. Worse, `OpalProfileCreate.output_format` is typed as an object but defaults to the string `wav`; SAY's generation request correctly defines a string enum. A generic CLI generator can faithfully reproduce an incoherent interface.

**Fix:** Keep common identity and activation semantics in a small base. Define typed service-specific profile extensions and validate defaults against their schemas. Reuse should reduce ambiguity, not distribute it to every client.

### 5. Arbitrary objects move essential documentation out of the contract

The schema admits `vad_settings`, `confidence_thresholds`, `model_paths`, engine options, and settings maps without enough portable meaning to create all the desired typed flags. LISTEN also leaves phrase-template grammar and numeric slot behavior unspecified here.

**Fix:** Publish schemas for portable controls and discoverable adapter-specific option schemas. Clearly distinguish values that need a restart from live changes. Do not invent a universal option name where engines have materially different behavior.

### 6. A match rate is not an accuracy score

`PerformanceSummary.match_rate` says how often the recognizer matched something; it has no reference labels. On a background-noise corpus, a higher match rate can mean a worse system. A nullable confidence score is not automatically a calibrated probability.

**Fix:** Name operational rates explicitly, define units/scales, and keep label-based evaluation metrics separate. Add rejection-aware results and run-scoped summaries. Preserve negative examples as first-class tests.

### 7. Reproducibility is much weaker in LISTEN results than in LOOK observations

LOOK observations carry engine/model IDs and pipeline revision. LISTEN results carry `profile_name`, transcript, intent, timing, and errors, but no equivalent immutable configuration identity, audio hash, model/runtime version, or benchmark case ID. A global status provider name cannot fully explain an old result.

**Fix:** Return a compact immutable configuration fingerprint and resolved provider/model details. Store audio fixture hashes and corpus/case versions with evaluation results. Retain both draft and refined transcripts when evaluating a second pass; the current result has no distinct draft/refined transcript pair.

### 8. Keep the valuable boundaries, and expose their consequences

Diagnostic calls never dispatch externally. SAY does not own playback. EVENT is best effort. THINK does not magically acquire local files. Those are useful boundaries. The missing piece is a small explicit composition/testing layer, not pretending each service already owns every adjacent responsibility.

**Fix:** Build a deterministic test coordinator or optional CLI plugin that composes existing calls. Make its additions visible: fixture preparation, run partitioning, scoring, joins, timeout handling, and exit-code gates. Do not hide an entire test platform behind a vaguely described `opal test` command.

## Opportunities worth exploring

The following are **proposals**, not commands generated from the current contracts. They are separate ideas beyond the 20 recipes.

| Opportunity | What it could reveal | Missing piece |
|---|---|---|
| Audio benchmark matrix | Model × runtime × hardware × language × room tradeoffs | Immutable corpus/run APIs, target selection, metric definitions |
| Audio metamorphic tests | Whether gain, silence padding, benign noise, or resampling changes intent | Deterministic transformations with provenance and paired results |
| “Mixture of ears” | Whether recognizer disagreement identifies uncertain commands | Multiple ASR outputs, calibrated arbitration, latency/cost budget |
| Refiner damage detector | Cases where the expensive second pass makes a correct draft worse | Draft/final transcripts, reference labels, paired scoring |
| Echo and self-trigger testing | Whether the assistant hears and obeys its own spoken answer | Explicit playback/capture integration, echo handling, loop assertions |
| Barge-in testing | Whether interruption stops speech and captures the new intent | Playback cancellation and timed duplex interaction contract |
| Adapter conformance pack | Which backends normalize audio, errors, and metadata consistently | Typed options, supported-format discovery, portable fixtures |
| Human voice acceptance panel | Whether automation serves the actual household users | Reviewed recordings, per-speaker results, held-out examples |
| Acoustic canary | Whether an update or microphone change silently worsens a room | A scheduled test coordinator, stable physical setup, drift thresholds |
| Visual command referee | Whether a voice-directed device action achieved its observable goal | Declared state detector, event/result correlation, explicit assertions |

For illustration only, a future optional test plugin might offer this workflow:

```bash
opal voice-test corpus add --manifest fixtures/voice_lab_manifest.json
opal voice-test run --corpus voice_lab --target whisper_lab --target vosk_lab --target faster_whisper_lab
opal voice-test report RUN_ID --metrics wer,intent_accuracy,slot_exact_match,false_accept_rate,p95_latency
opal voice-test gate RUN_ID --policy policies/voice_release.json
```

Those four commands require new client behavior and/or contracts. The referenced manifest and policy files are proposed inputs, not supplied artifacts. They are shown to make the design opportunity concrete, not to disguise a missing implementation.

**Most valuable first experiment:** Compare the same reviewed human recordings through two configured recognizers, include negative audio and number/negation traps, and inspect every disagreement. That will reveal more than a polished synthetic “hello world” demonstration.

## Sources and validation scope

All links below are pinned to the reviewed commit, so later contract changes do not silently change this cookbook's basis.

| Source | What was used |
|---|---|
| [LISTEN contract](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/opal_listen.yaml) | Recognition endpoints, live SSE protocol, diagnostic publication prohibition, slots, failures, regression, timing, revision gaps |
| [SAY contract](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/opal_say.yaml) | Installed engines, voice validation, generation controls, WAV/PCM boundaries, requested sample-rate behavior |
| [EVENT contract](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/opal_event.yaml) | Actions, conditions, dry-run rules, retained executions, admission and delivery semantics |
| [THINK contract](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/opal_think.yaml) | Singular CLI groups, tasks, model discovery, metadata and terminal answer semantics |
| [LOOK contract](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/opal_look.yaml) | Capability discovery, frame analysis, task-specific observations and provenance |
| [Shared kernel](https://github.com/PatMartin/opal-api/blob/7ecc78adc8d7d8ced25ef85476c5e5f19b0fae3d/openapi/shared/opal_kernel.yaml) | Shared profiles, engines, settings, correlation, ETags, common health operations |

Static checks covered **151 command examples**, **53 distinct service/method/path mappings**, and **47 literal JSON request bodies**, including required fields, types, allowed values, and declared query parameters. The four examples affected by revision gaps remain explicitly conditional; passing static checks does not resolve those gaps. The four proposed `voice-test` commands were excluded from current-contract validation.

Runtime adapters, deployment settings, fixture existence/content, recognition quality, physical actions, and the proposed Typer implementation were not executed or certified.
