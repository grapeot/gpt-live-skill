# GPT-Live API Development

> Agent-facing skill for building voice applications on the OpenAI GPT-Live API family (`gpt-live-1` and successors).

## Metadata

- **Type**: API Guide
- **Scenario**: Designing and implementing full-duplex voice sessions — session lifecycle, audio streaming, delegation to a backend, returning results to the spoken conversation
- **Working language**: English
- **Created**: 2026-09-12
- **Authority**: The official OpenAI docs (`developers.openai.com/api/docs/guides/live*`) are the source of truth; verify against them before shipping. This skill encodes the architectural invariants and the pitfalls that are easy to miss.

## What this skill enables

An agent using this skill can, end to end: start a GPT-Live session, stream microphone audio in and spoken audio out, handle one or more delegation cycles (model decides → backend reasons and requests tools → the application executes → the result is spoken), and close the session with confirmed final usage.

## Out of scope

- The Realtime API (`gpt-realtime-*`). Different event contract (input-buffer commit, `response.create` turn loop, `response.done` for voice). Do not mix the two.
- WebRTC browser media negotiation details (it is a valid connection type; the primary WebSocket path is this skill's focus).
- Telephony/SIP carrier integration and partner integrations (LiveKit, Twilio, Telnyx, Daily).
- Custom voice creation (requires a separate signed agreement with OpenAI sales).

## Architecture: three layers, strict separation

The single most important thing to get right is the division of labor.

1. **Voice frontend (the GPT-Live model, hosted by OpenAI).** Listens and speaks simultaneously (full-duplex). Handles interruptions, pauses, backchannels ("mm-hmm"), and distinguishing background noise from speech. Decides *when* to involve the backend. It has a small context window and must not carry business rules.
2. **Backend (a hosted Responses model in Responses-delegation mode, or any service you operate in client-delegation mode).** Reasons over the conversation context supplied for it and emits structured tool calls (name + arguments). It does not execute anything either.
3. **Your application / relay (a trusted server holding the API key).** The only layer with filesystem, shell, credential, or business-system access. It executes tools, enforces permissions and confirmations, keeps task state, and decides what reaches the voice layer.

**Invariant across every mode and every model: the model emits intent; the relay executes.** A tool call is a structured request that only exists until your code runs it. This is also the guardrail point: parameter validation, irreversibility checks, and user confirmation all happen here, because neither the voice model nor the backend can enforce them.

The two real risk shapes this design addresses:

- **Noisy input.** The backend reasons over spoken-transcript context, which can mishear names, dates, and numbers. Tool-call arguments can be slightly off even when the backend itself is reliable.
- **No human brake.** A full-duplex loop runs fast; by the time a call reaches execution the user may not have had a chance to interject. If the tool is irreversible, there is no pause point. Hence: keep the early tool surface narrow and read-only/idempotent; require app-side confirmation for mutating operations.

## Delegation modes

The mode is chosen at session creation. Switching modes requires a new session (attempting it mid-session fails with `immutable_field_update`).

### Responses delegation (managed)

```
delegation: { type: "responses", responses: { model, instructions, tools, tool_choice, ... } }
```

- The backend model must be an OpenAI-hosted Responses model. There is no way to point this mode at your own endpoint or an OpenAI-compatible server.
- OpenAI manages the backend connection, prepares conversation context, and returns backend events to the live session inside `response.event` envelopes.
- Tools: `function` schemas (executed by your application) plus `web_search` (hosted by OpenAI, zero code).
- Conversation context flows through OpenAI's backend path — relevant when context sensitivity matters.

### Client delegation (self-operated)

```
delegation: { type: "client" }
```

- The backend is anything you run: another LLM (any provider), an agent harness, or a plain service.
- `session.delegation.created` carries metadata only: the delegation id and an offset timestamp. It does **not** contain the user's utterance or task text. Build the task from the transcript deltas you accumulate plus your application state.
- You own the full path: context assembly, backend request, execution, and deciding which result is spoken.
- Context stays in your process — the right choice when delegation involves private or personal data.

### Choosing

Prefer Responses delegation when the managed workflow fits (you want OpenAI to prepare backend requests and route results back). Prefer client delegation when you need your own model or agent, ownership of which context each backend call sees, filtering of results before they are spoken, or multiple different backends per task.

## Session lifecycle (primary WebSocket)

1. Connect to `wss://api.openai.com/v1/live/sessions` with `Authorization: Bearer $OPENAI_API_KEY`. No query parameters. The key stays on the trusted server.
2. Send `session.start` as the first message, containing the full session config: `model` (`gpt-live-1`), `instructions` (the short live prompt), `audio.format`, `audio.output.voice`, and `delegation`.
3. Wait for `session.started` (returns the resolved config and session id) before sending any audio.
4. Stream the session:
   - In: `session.input_audio.append` with base64 raw audio (no acknowledgment is sent for these).
   - Out: `session.output_audio.delta` (base64 audio), `session.input_transcript.delta` / `session.output_transcript.delta` (transcript fragments), delegation events, and — in Responses mode — `response.event` envelopes.
5. Send `session.close` and keep receiving until `session.closed` arrives; it carries the final usage. Release the socket only after that.

`session.update` adjusts supported settings within the existing delegation mode; `session.instructions.append` adds conversation instructions; `session.input_audio.mute`/`unmute` gate incoming audio. Muting input does not cancel backend work and does not stop generated speech.

## Audio format rules

- One format per session, applies to both directions, fixed at startup:
  - `{"type":"audio/pcm","rate":24000}` — mono signed 16-bit little-endian PCM (default)
  - `{"type":"audio/pcm","rate":16000}` — same, 16 kHz
  - `{"type":"audio/pcmu","rate":8000}` / `{"type":"audio/pcma","rate":8000}` — G.711, one byte per sample
- Base64 the raw bytes. No WAV or other container headers.
- PCM chunks must contain complete 16-bit samples: byte length must be even. Carry an odd trailing byte into the next chunk.
- Resample if your source rate differs; the API does not convert your bytes.
- There is no output-audio-done event and no timing fields on output audio. Track your own playback queue to know what has actually been heard. Transcript timestamps are intervals on the session timeline, not playback-completion markers.

## Handling a delegation cycle

### Responses mode

- Backend events arrive nested: a `response.event` envelope whose inner `event` is a Responses event. Dispatch on the inner event type; preserve the outer `delegation_id`.
- A completed function call is identified in nested `response.output_item.done` (item carries `call_id`, `name`, `arguments`). An arguments-done event alone is not sufficient.
- **`response.output: []` on a lifecycle snapshot — including `response.completed` — does not mean there are no pending function calls.** The forwarded snapshots deliberately carry empty output; collected `response.output_item.done` events are the only source of truth for what must be submitted.
- After executing the authorized operation:
  1. `response.item.create` with an item `{type: "function_call_output", call_id, output: "<json string>"}`.
  2. Explicitly `response.create` to continue the backend response.
- Submitting a result does not continue the response. Submit every pending result before continuing. `response.create` is a Live command that uses the session's configured backend — it does not take a Responses request body, model override, or `delegation_id`.

### Client mode

- On `session.delegation.created`, read `event.delegation.id`. Current ids look like `item_...`; treat the id as opaque and return it unchanged.
- Assemble the task from accumulated transcripts (which can contain mistakes, unfinished phrases, and corrections) plus application state.
- Run your backend, then return the result:
  - `session.commentary.append` — content the model should **speak** (it paraphrases the appended text).
  - `session.thinking.append` — quiet context for later responses; not spoken on append.
  - `session.instructions.append` — steer or redirect the live model's behavior (e.g., after a guardrail block).
- All three take a plain-string `content` (≤ 500 tokens per append) and **require `delegation_id`** — the delegation's id, or `null` for general session context.
- Appends confirm context injection, not speech. An acknowledgment means the text was accepted, not that the model consumed or said it.
- Repeated appends can continue the same delegation; stream coherent, verified chunks rather than one final blob.
- **A spoken interruption does not cancel backend work.** If the user changes the task, your application decides whether to cancel, re-task, or let the old work finish. Before announcing a cancellation or a success, verify the action actually happened. On a lost response, check whether the original action already occurred before retrying — a retry must not double-book.

### Append acknowledgments and session lifetime

The `*.appended` acknowledgments (`session.commentary.appended`, `session.thinking.appended`, `session.instructions.appended`) match your outgoing `event_id` via their `client_event_id`. Three invariants verified against a live session:

- **Frame progress feeds the ack.** The acknowledgment waits until the session's frame timeline reaches the estimated end of context injection. If no audio frames are flowing (e.g., a headless client stopped sending input after the user's utterance), the timeline freezes and the ack stays pending indefinitely. A real microphone's ambient noise is a natural keepalive; unattended clients and CI must keep sending silence frames while waiting for a backend result, or the result will never be acknowledged and never spoken.
- **Closing with pending appends loses the result.** `session.close` while an append ack is pending fails it with `context_injection_incomplete` — the text is stranded in context, never spoken. Before closing, drain in-flight appends: await their `*.appended` acks (matched by `event_id`) or fail them explicitly. A backend result computed but not acknowledged is not delivered.
- **The backchannel is not the answer.** While the backend works, the live model speaks waiting sounds ("mm, let me check") that arrive as ordinary `output_transcript` deltas. Treating the first assistant transcript as "the result arrived" and closing shortly after loses the race with the real result. The reliable completion signal in client mode is your own application state: the delegation is done when its append ack arrives (the relay forwards this as a state change), not when any assistant text appears.

- **The backchannel is not the answer.** While the backend works, the live model speaks waiting sounds ("mm, let me check") that arrive as ordinary `output_transcript` deltas. Treating the first assistant transcript as "the result arrived" and closing shortly after loses the race with the real result. The reliable completion signal in client mode is your own application state: the delegation is done when its append ack arrives (the relay forwards this as a state change), not when any assistant text appears.

### Backend submission, timeouts, and late results

Four behaviors verified with a client-delegation relay over an agent backend (each fixed a live failure):

- **A synchronous backend submission blocks until the whole generation finishes.** A plain task took ~4 s and a real lookup blew past the HTTP client's default 30 s timeout — the relay declared failure while the backend was still working. Submit backend work asynchronously (submit-then-poll: return immediately, then poll the backend's busy/idle status) so your `DELEGATION_WAIT_SECONDS` budget, not an HTTP client timeout, governs the wait. Warm-up submissions (e.g., a role prompt at session start) belong to the same session and must finish before the first delegation, or the two will race.
- **A timeout is not a failure.** When the budget expires, the live model should say it is still waiting — and the relay should keep waiting in the background for a late result, then inject it via a second append when it arrives. Guard the late delivery: drop it if the session is closing or a newer delegation has replaced this one, so an outdated answer is never spoken.
- **Cancelling the local wait does not stop the backend.** `task.cancel()` on your relay only unwinds the local await; the backend's server-side run keeps consuming tools after the user pressed stop. Issue an explicit abort to the backend on close, and do not let a failed abort block the rest of the shutdown.
- **The backend agent needs search-scope rules in its prompt.** Asked about a vague directory, an agent will happily run a whole-tree `find`/`grep -r` and burn the entire time budget. Put constraints in the backend prompt: no unbounded recursive searches, avoid dependency/cache directories, disambiguate vague targets first, keep single queries to seconds.
- **Feed the backend sentences, not transcripts.** A word-by-word delta list with millisecond ranges per token is token-expensive and hard for the backend to read. Merge adjacent same-role spans into plain sentences and describe the window as "the last 15 seconds", not as offsets.

## Live prompt (`session.instructions`)

The live model has a small context window. Keep the prompt to:

- Role, tone, and pace (a few sentences).
- Backchannel policy (start with moderate).
- Interruption policy (stop speaking when interrupted, listen).
- A **Delegation policy** section with three labels: `Backend tools:` (what the backend can actually do), `Delegate to the backend when:`, and `Do not delegate to the backend when:` — concrete conditions, not "when needed".

Everything else — business rules, procedures, tool schemas, result handling — belongs in the backend prompt. The live model must not promise outcomes or claim an action finished before the backend confirms it. Do not add a "never speak while the user is speaking" rule alongside a backchannel policy; it suppresses the listening sounds you want.

## Security boundary

- The API key exists only on the trusted server. Devices and browser clients connect to your relay; the relay connects to OpenAI. Never embed the key in device firmware or a client bundle.
- The application owns permissions, confirmations, business records, and durable task state. No delegation event, model instruction, or appended instruction approves an action.
- Appending instructions can redirect the conversation but does not block or cancel in-flight backend work — enforce blocks in application state and handle already-running work.
- Keep secrets out of `session.thinking.append`: it is quiet context, not a private channel, and it can influence later speech.

## Acceptance criteria (what "done" looks like)

A GPT-Live integration built with this skill is done when:

1. `session.start` is accepted and `session.started` arrives with a session id, and no audio was sent before it.
2. One full delegation cycle is verified end to end: user utterance → delegation event → exactly one local execution of the tool (verifiable in application state or logs) → result returned via the correct event → the spoken response plays.
3. `session.close` is answered by `session.closed` and the final usage is captured. A socket released without `session.closed` is a failed close.
4. No credential exists outside the trusted-server layer.

Verify both sides of any consequential interaction: the authoritative application state (did the tool run, exactly once?) and the audio actually played (the backend can finish while the spoken result is still being interrupted).

## Known pitfalls

All entries below come from the official documentation's explicit warnings — these are the failure modes that look plausible but are wrong.

| Pitfall | Symptom | Response |
|---|---|---|
| Treating the delegation event as carrying the task (client mode) | Backend receives an empty or generic task | Build the task from transcript deltas + application state |
| Stopping input audio while waiting for a backend result (headless client) | Append ack stays pending; `context_injection_incomplete` on close; result never spoken | Keep sending silence frames to keep frame progress advancing |
| Sending `session.close` while an append ack is pending | `context_injection_incomplete`; result stranded in context | Drain in-flight appends (await `*.appended` by `event_id`) before closing |
| Treating the first assistant transcript (backchannel) as the result | Session closed on a timer before the real result; result lost | Completion signal is the append ack / application state, not assistant text |
| Submitting backend work with a blocking synchronous request | HTTP client timeout fires while the backend still runs; relay declares failure mid-generation | Submit async and poll the backend's busy/idle status; let the app's time budget govern |
| Abandoning a backend result when the wait budget expires | User hears only "still checking"; the correct answer arrives later and is dropped | Announce the wait, keep listening in the background, inject the late result with staleness guards |
| Cancelling only the local wait on shutdown | Backend run keeps consuming tools after the user pressed stop | Abort the backend run explicitly on close; don't let a failed abort block shutdown |
| Letting the backend agent run unbounded whole-tree searches | One vague query burns the entire time budget (e.g., an 88 s `find`) | Put search-scope constraints in the backend prompt: bounded queries, skip dependency/cache dirs |
| Feeding the backend a word-by-word transcript with millisecond ranges | Token-expensive, hard to read | Merge same-role spans into sentences; describe the window in plain language |
| Assuming empty `response.output` means no pending calls | Tool result never submitted; backend response hangs | Collect completed calls from `response.output_item.done` only |
| Submitting a function output and expecting the response to continue | Backend response never completes | Always follow `response.item.create` with `response.create` |
| Attempting to switch delegation mode mid-session | `immutable_field_update` error | Start a new session |
| Sending an odd-length PCM chunk | Audio glitches or malformed append | Carry the trailing byte into the next chunk |
| Summing voice-usage snapshots | Cost double/triple-counted | Usage snapshots are cumulative; take the last value |
| Releasing the socket right after `session.close` | Final usage never confirmed | Keep receiving until `session.closed` (bounded wait, ~15 s) |
| Muting input to stop a task | Backend work keeps running; speech keeps playing | Mute ≠ cancel; cancel in application state and verify |
| Packing business rules into the live prompt | Degraded delegation decisions, conflicting instructions | Rules go to the backend prompt; the live prompt stays short |
| Treating transcript timestamps as playback completion | "It already said that" but the user never heard it | Track your playback queue; timestamps are session-timeline intervals |
| Letting a retry re-run an action after a lost response | Double-booking, duplicate side effects | Check whether the original action already happened before retrying |
| Embedding the API key in a device or client bundle | Credential exposure | Key stays on the relay; clients talk to the relay |
| Using Realtime API patterns (input-buffer commit, voice-turn `response.create` loop) | Session behaves unpredictably | GPT-Live manages listen/speak as streams; `response.create` starts or continues delegated backend work only |