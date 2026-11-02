# Import this conversation to AgentSky

## Summary schema (asteroids.session-summary v1)

Write a JSON file with these fields:

```json
{
  "format": "asteroids.session-summary",
  "format_version": 1,
  "source_harness": "claude_code",
  "source_harness_version": null,
  "source_session_id": "<the session id, e.g. from CLAUDE_CODE_SESSION_ID>",
  "created_at": "<ISO 8601 timestamp>",
  "summary": "<one paragraph: what this conversation accomplished>",
  "context": {
    "objective": "<what the person ultimately wants>",
    "constraints": ["<must not do X>"],
    "decisions": ["<decided to use Y approach>"],
    "completed": ["<finished task A>"],
    "pending": ["<next: do B>"],
    "verification": ["<how to confirm B is correct>"],
    "blockers": ["<needs access to X>"],
    "artifacts": [{"path": "path/to/file.ts", "description": "what it is"}]
  }
}
```

Only `summary` and `context` require judgment. All other fields are fixed values
or can be omitted (the CLI validates `format` and `format_version`).

## Gaps

The import transfers the recorded conversation only. These are NOT transferred:
- Local files not included in the transcript
- Runtime state (variables, shell state, running processes)
- Unrecorded tool outputs or reasoning
- Login credentials and secrets

List known gaps in `context.blockers`.

## Non-Claude/Codex harnesses

For harnesses that do not set `CLAUDE_CODE_SESSION_ID` or `CODEX_THREAD_ID`:
use `--session <id>` instead of `--current`, and set `source_harness` to the
harness name (e.g. `"hermes"`, `"openclaw"`).
