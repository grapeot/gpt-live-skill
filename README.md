# gpt-live-skill

A skill and (soon) runnable examples for building full-duplex voice agents on the OpenAI GPT-Live API family (`gpt-live-1`).

GPT-Live is a voice model that listens and speaks at the same time. It owns the conversational experience — interruptions, backchannels, turn-taking, background-noise handling — and delegates reasoning and tool calls to a backend you choose. The voice layer is billed by audio duration ($0.05/min at launch); the backend model and its tool usage are billed separately.

## What is here

- `skills/gpt-live/` — an agent-facing skill for developing against the GPT-Live API: architecture, delegation modes, session lifecycle, event handling, audio constraints, and known pitfalls.
- `docs/working.md` — changelog and lessons learned.
- Runnable examples are coming; the repo starts with the skill so agents can build against the API correctly from day one.

## Installing the skill

Hand this repository's URL to your coding agent (Codex, Claude Code, Cursor, OpenCode, or similar) and ask it to install the GPT-Live skill. The installing agent should start from the target workspace's `AGENTS.md` or `CLAUDE.md`, follow any routing file it references (for example a `WORKSPACE.md`), and link `skills/gpt-live` into the workspace's skill discovery chain — for example an index file like `rules/skills/INDEX.md`, or a global skills directory such as `~/.config/opencode/skills/` or `~/.claude/skills/`.

## Roadmap: runnable examples

Planned additions, each a small standalone program that an agent can read and adapt:

1. **WebSocket quickstart** — microphone in, speaker out, one session, Responses delegation with hosted `web_search`. Validates the API access and the event loop.
2. **Client-delegation relay** — a trusted server holding the API key; the backend is an external agent (the example targets an OpenCode session). Demonstrates transcript accumulation, delegation handling, and `session.commentary.append` result delivery.
3. **Device client (ESP32-S3)** — an M5StickS3-style client: ES8311 24 kHz PCM uplink/downlink to the relay, LCD captions from transcript events, button-driven session start/stop.

## Configuration

Copy `.env.example` to `.env` and set your OpenAI API key. Credentials are read from local environment variables and belong on a trusted server; clients and devices connect to that server rather than to OpenAI directly.