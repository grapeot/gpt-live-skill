# AGENTS.md — gpt-live-examples

## Role

Public repository. Contains an agent-facing skill and (later) runnable examples for developing against the OpenAI GPT-Live API family (`gpt-live-1`). Working language: English.

## Structure

- `skills/gpt-live/SKILL.md` — the skill; the primary artifact of this repo
- `docs/working.md` — changelog and lessons learned
- `.env.example` — environment contract; fake values only

## Rules

- Update `docs/working.md` after meaningful changes (changelog entry per change; lessons learned only for real failures).
- This is a public repo: no real API keys, emails, phone numbers, internal paths, server addresses, or 1Password references in any tracked file. Use the fake placeholders from `.env.example`.
- Run a privacy scan before pushing and treat zero matches as the bar (pattern uses the `[U]sers`-style bracket trick so the scan does not match its own command text):
  `rg -n "o[p]://|/[U]sers/[a-z]|sk-[A-Za-z0-9_-]{16,}|BEGIN [A-Z ]*PRIVATE KEY|grapeot[@]" .`
- The README is user-facing: explain purpose, layout, and skill installation. Keep public-readiness and hygiene discussion in this file, not the README.
- Skill edits follow the meta-skill principles: result determinism over process scripts; known pitfalls only from real failures or explicit official-doc warnings; every section must raise the probability that an agent completes the task correctly.
- Version control: only commit when explicitly asked; small, independently reviewable commits.