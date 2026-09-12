# gpt-live-examples

A skill and (soon) runnable examples for building full-duplex voice agents on the OpenAI GPT-Live API family (`gpt-live-1`).

GPT-Live is a voice model that listens and speaks at the same time. It owns the conversational experience — interruptions, backchannels, turn-taking, background-noise handling — and delegates reasoning and tool calls to a backend you choose. The voice layer is billed by audio duration ($0.05/min at launch); the backend model and its tool usage are billed separately.

## What is here

- `skills/gpt-live/` — an agent-facing skill for developing against the GPT-Live API: architecture, delegation modes, session lifecycle, event handling, audio constraints, and known pitfalls.
- `docs/working.md` — changelog and lessons learned.
- Runnable examples are coming; the repo starts with the skill so agents can build against the API correctly from day one.

## Installing the skill

Hand this repository's URL to your coding agent (Codex, Claude Code, Cursor, OpenCode, or similar) and ask it to install the GPT-Live skill. The installing agent should start from the target workspace's `AGENTS.md` or `CLAUDE.md`, follow any routing file it references (for example a `WORKSPACE.md`), and link `skills/gpt-live` into the workspace's skill discovery chain — for example an index file like `rules/skills/INDEX.md`, or a global skills directory such as `~/.config/opencode/skills/` or `~/.claude/skills/`.

## Configuration

Copy `.env.example` to `.env` and set your OpenAI API key. Credentials are read from local environment variables and belong on a trusted server; clients and devices connect to that server rather than to OpenAI directly.