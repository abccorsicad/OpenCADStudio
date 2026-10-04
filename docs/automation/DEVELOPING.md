# Developing the MCP bridge

Operator-facing setup lives in [README.md](README.md). This note is for
contributors iterating on the bridge itself (`src/mcp.rs`).

## Fast iteration: no version bumps needed

- Run the bridge straight from a local build: `target/debug/OpenCADStudio --mcp`
  for speed, `target/release/OpenCADStudio --mcp` when close to shipping.
  Both report the same handshake; only the `build_profile` field differs.
- Tool-surface changes need no version-number change. `serverInfo` carries a
  content `tool_schema` digest plus the `build_rev` git hash, so staleness is
  observable without a release.
- If you change any tool schema, the `tool_schema_digest_is_pinned` test
  fails: review the diff, then update `TOOL_SCHEMA_DIGEST` deliberately
  (never blindly).

## Sessions are per client connection

MCP clients bind the tool surface once when they connect, so after every
rebuild, restart the AI client session (or reconnect its MCP servers) to pick
up the new schema. The bridge helps: it watches its own executable and, once
it detects a rebuild, finishes the in-flight request and exits, so the next
call respawns a fresh bridge. No in-flight call ever fails because of this;
at worst the following call pays one respawn.

## Session discovery notes

`ocs_sessions` skips descriptors whose GUI process is gone (verified per OS:
Linux reads the process table directly and skips the check entirely where
`/proc` is unavailable, Windows and macOS take one process snapshot cached
for 10s per call) and deletes those stale files. Descriptors without process
information keep the legacy TCP-probe path, and every probe uses a bounded
connect timeout, so a dead session can delay discovery by at most its timeout
instead of hanging past client deadlines.

Concurrent launches are serialized with `automation/starting.lock`
(`{"pid":..,"started":unix_secs}`, 60s TTL): the first `launch_if_none: true`
caller claims it and spawns the GUI, later callers with a fresh live claim
wait for the same descriptor instead of opening another window. Stale claims
and claims from dead pids are reclaimed; an unusable directory fails open so
a broken lock can never wedge launching. Covered by
`startup_lock_serializes_concurrent_launches` (claim, wait, stale-reclaim,
dead-pid-reclaim, release).

## Rebuilding the release binary while the bridge runs (Windows)

The running `--mcp` bridge locks `target/release/OpenCADStudio.exe`, so the
link step fails with `Access is denied (os error 5)`. Rename the running exe
aside (`OpenCADStudio.prev.exe`) and rebuild — the live bridge keeps serving
from the renamed file, then its rebuild detector fires and it exits cleanly.
Restart the AI client afterwards so it respawns from the fresh binary, and
delete the `.prev` file once nothing runs from it.

## Backlog (agreed, not yet implemented)

- Dimension exporter: exploded dims / dangling-handle failure in
  `save_verified` output (needs codec work; workaround: dressed text + witness
  lines, see plan-drawing notes).
- Strict-client `outputSchema` validation of error shapes (worked around by
  omitting `structuredContent` on errors; opencode-side strictness remains).
- Wrapper regeneration for `annotate`/`tiles`/`diff` capture params (blocked:
  needs a live session + client `request_id` support on capture).
- End-to-end proof of the startup lock (two concurrent `launch_if_none: true`
  calls yield one GUI window) — needs a controlled window-spawning test.
- Subagent clean-interaction retest — DONE 2026-10-04: subagent announced
  `2026.39 / 64dc2c3c3c24` from the `bridge` object, capabilities round-trip
  succeeded, digest matched, no friction.
- SEP-2549 follow-up (optional): shorter TTL for `resources/*` (dynamic,
  e.g. 60s) while keeping 1h for the static `tools/list`.
