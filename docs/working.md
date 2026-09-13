# Working Log

## Changelog

### 2026-09-13
- Added "Backend submission, timeouts, and late results" to the client-delegation guidance, from five live failures in a relay + agent-backend integration (four merged PRs, each verified with offline tests plus a live e2e):
  - synchronous backend submission blocked until generation finished and blew an HTTP timeout, so the relay declared failure mid-run → submit async and poll busy/idle
  - an expired wait budget dropped the correct answer that arrived later → announce the wait, keep listening, inject the late result with staleness guards
  - cancelling the relay's local await left the backend run consuming tools for 208 s after the user pressed stop → abort the backend explicitly on close
  - an agent interpreted a vague "src directory" as a whole-tree search (88 s `find`) and burned the entire budget → search-scope rules belong in the backend prompt
  - the relay fed the backend a word-by-word transcript with millisecond ranges → merge same-role spans into plain sentences
- Five matching rows in known pitfalls. All verified offline (unit tests over fake backends) and live (e2e with `gpt-live-1` + OpenCode).

### 2026-09-12
- Initial scaffold: README, GPT-Live skill (`skills/gpt-live/SKILL.md`), AGENTS.md, `.env.example`, CI privacy scan
- Skill covers: three-layer architecture, delegation modes (Responses vs client), session lifecycle, audio format rules, delegation cycle handling, live prompt guidance, security boundary, acceptance criteria, known pitfalls
- Repo renamed to `gpt-live-skill`; added README roadmap for planned runnable examples
- Added "Append acknowledgments and session lifetime" section to the skill, from a verified live failure: append acks are fed by frame progress (stalled input audio = ack pending forever), closing with pending appends strands the result (`context_injection_incomplete`), and the live model's backchannel is not the result-arrived signal. Three matching rows in known pitfalls. All three invariants reproduced against `gpt-live-1` with a client-delegation relay + OpenCode backend.

## Lessons Learned

- The failure mode that produced the new skill section: a headless e2e client (macOS `say` TTS → relay → GPT-Live) stopped sending input audio after the utterance. OpenCode computed the correct answer, the relay appended it, but the ack never arrived because the session's frame timeline had frozen; the harness then closed the session and the answer was lost to `context_injection_incomplete`. Fixed by draining pending appends before close and streaming silence frames as keepalive — then the same e2e delivered the spoken answer end to end with a clean close.
- `websockets` 15.x does not mount `websockets.exceptions` on plain `import websockets`; `except (websockets.exceptions.ConnectionClosed, ...)` only evaluates the attribute when an exception fires, masking the real error with `AttributeError`. Import `ConnectionClosed` from the submodule at the top. (Harness-side, noted here because it surfaced while verifying this skill's guidance.)