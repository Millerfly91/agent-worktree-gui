# Remaining phases — run as: `general-purpose` agent on Sonnet 5.5

This is the executable handoff for everything left in the Claude Code plan (milestones 2b–2f and the close-out).
Read `docs/claude-phase2-handoff.md` first for *why* and for what the spike proved; this file is *how*, in order.

## Why this agent

Every phase edits code, runs it, and uses git, so the agent needs the full toolset: `general-purpose` (or the
catch-all `claude`) — not `Explore` or `Plan`, which are read-only. Sonnet 5.5 is the efficient model: each phase
below is specified down to files and acceptance checks, so it does not need a larger model's open-ended planning.
Spend the extra only on 2b (a refactor of the 96 KB `lib/jobs.js`), and only if Sonnet stalls on it: re-run just that
phase on Opus 5.5. Do one phase per run, not all five in one context.

## Ground rules (apply to every phase)

- **Repo and PRs.** Work in `Millerfly91/agent-worktree-gui` (`origin`; `upstream` is `JLRoper/…`). One branch and one PR
  per phase, opened against the fork's `main`. Do not merge your own PR. Before branching, `git fetch origin` and
  merge `origin/main`; if the previous phase is unmerged, stack on its branch and say so in the PR.
- **Version.** Each PR bumps the *third* number of `package.json` (0.2.x) — see the table below — and
  `node tools/verify-version-bump.mjs --base origin/main` must pass before you push. The close-out PR is the only one
  that moves the minor number.
- **`config.json` is local.** In this clone it points `scanRoots` at `Documents/Projects`. Never commit it:
  `git stash push config.json` around branch switches and rebases, and `git add` files by name, never `-A`.
- **Checks before every push:** `node --check` on every file you touched, plus the existing `tools/verify-*.mjs` that
  cover what you changed (`verify-verdicts.mjs <project>` is the cheap general one).
- **Billed runs are small on purpose.** Any live Claude or Hermes run uses a tiny prompt, `--model haiku` for Claude,
  and `--max-budget-usd`. Test in a throwaway git repo, never in a real project: start a second panel with
  `PANEL_CONFIG=<file>` on a free port, `scanRoots` pointing at the throwaway directory (see `SETUP.md`, "A sandbox").
- **Do not break Hermes.** Hermes stays the default agent and its behaviour must be byte-for-byte unchanged until the
  picker in 2f. After 2b and after every later phase, run one real Hermes test agent (`PONG`, as in the earlier smoke
  test) and confirm it completes.
- **Handoff upkeep.** When a phase lands, tick it in the table below and note anything the next agent must know.

## Milestones and versions

| phase | what | branch | version | status |
|---|---|---|---|---|
| 2a | spike, handoff, version check | `claude-phase2-handoff` (PR #2) | 0.2.1 | done |
| — | this handoff | `remainingPhases` | 0.2.2 | this PR |
| 2b | transport split, no behaviour change | `claude-2b-transport-split` | 0.2.3 | todo |
| 2c | Claude spawn, stream, stop | `claude-2c-spawn` | 0.2.4 | todo |
| 2d | reply (resume) and steer | `claude-2d-reply-steer` | 0.2.5 | todo |
| 2e | approvals | `claude-2e-approvals` | 0.2.6 | todo |
| 2f | dedup, UI picker, docs, verify script | `claude-2f-ui-dedup` | 0.2.7 | todo |
| close | `Status: complete`, minor bump | `claude-complete` | 0.3.0 | todo |

If a phase splits, the extra PR takes the next patch number and the later ones shift; the rule is the contract, the
table is a forecast.

## Phase 2b — transport split (no behaviour change)

**Goal.** Put the Hermes HTTP calls behind an interface so a second transport can sit beside it. Nothing a user can see
changes.

**Seams in `lib/jobs.js`** (line numbers are from `main` at v0.2.0; re-find them by the quoted calls):
`probe` (`/v1/capabilities`), `create` (`POST /v1/runs`, worktree creation, the prompt-anchoring hack), `follow`
(SSE `/v1/runs/{id}/events`), `poll` and `controlContext` (`GET /v1/runs/{id}`), `steer`, `approve`,
`answerQuestion` (`/clarify`), `stop`. Shared and **not** to be touched: `consume`, `settle`, `push`, `emit`,
`linesForEvent`, the persistence and label stores, artifact scanning.

**Do.**
1. New `lib/transports/hermes.js` exporting a class with `probe()`, `start(job, body)`, `follow(job, onEvent)`,
   `status(job)`, `steer(job, text)`, `approve(job, choice, requestId)`, `answer(job, answer, requestId)`, `stop(job)`
   and `info()`. Move the code over verbatim — same requests, same error strings — taking the Hermes pieces of
   `create`/`follow`/… with it. `apiConfig`, `headers`, `previewRequest` and the key-from-`.env` reader move too.
2. `Jobs` holds a map of transports keyed by `agent` (`"hermes"` for now) and a `job.agent` field (default
   `"hermes"`, persisted; old `jobs.json` entries without it read as Hermes). `create/steer/approve/…` delegate by
   `job.agent`.
3. Keep `jobs.api.url`, `jobs.probe()` and `jobs.transportInfo()` working: `server.js` and `lib/assistant.js` call
   them (`server.js` ~15 sites, `assistant.js` only `create`/`stop`/`answerQuestion`). Prefer delegating getters over
   editing those callers.

**Done when.** `node --check` clean; `verify-verdicts` and `verify-assistant` pass as before; a real Hermes test agent
(`PONG`) spawns, streams and finishes, and a second one can be steered and stopped; the diff for `consume`, `settle`
and the UI is empty.

## Phase 2c — Claude spawn, stream, stop

**Goal.** A Claude job that runs, streams into the existing card, and can be stopped.

**Do.**
1. `lib/transports/claude.js`, implementing the same interface. `start` spawns
   `claude -p --input-format stream-json --output-format stream-json --verbose --session-id <uuid> --model <m>
   --max-budget-usd <n> --permission-mode <mode> --permission-prompt-tool stdio` with `cwd` = the job's worktree
   (no prompt anchoring — the process cwd is the agent's cwd). No shell, `windowsHide: true`, prompt on **stdin**.
   Binary from config `claudeBin` (default `claude`).
2. Wire protocol, all proven in the spike (`tools/spike-claude-stream.mjs`), one JSON object per line:
   - send first: `{"type":"control_request","request_id":"<uuid>","request":{"subtype":"initialize"}}`
   - send a turn: `{"type":"user","message":{"role":"user","content":"<text>"}}`
   - read: `system/init` → `assistant` (blocks `thinking` / `text` / `tool_use`) → `user` (`tool_result`) → `result`.
     Ignore other `system` subtypes (hooks) and `rate_limit_event`.
3. Translate to the panel's events so `consume()` works unchanged: `init` → an `init` line; `tool_use` →
   `tool.started` `{tool, preview}`; `tool_result` → `tool.completed` `{tool, preview, error, duration}`; `text` →
   `message.delta`; `result` → `run.completed` (or `run.failed` when `is_error`) carrying `usage` and
   `total_cost_usd`. Map Claude tool names onto the glyph table in `toolGlyph`.
4. **A job ends when the process exits, not at the first `result`** — a background task can wake the session for a
   second `init` + `result`. Keep stdin open for the process's life; close it on stop or when the card is dismissed.
5. Stop = write `{"type":"control_request","request_id":"<uuid>","request":{"subtype":"interrupt"}}`; the run ends
   with `result.subtype === "error_during_execution"` and exit code 1 — report that as **cancelled**, not failed.
   Fallback after ~5 s: `taskkill /PID <pid> /T /F`.
6. Persist `thread → session uuid` with the job store so 2d can resume. A panel restart marks in-flight Claude jobs
   `interrupted` (the child dies with the panel) exactly as Hermes jobs are.
7. Create the worktree with the existing `makeWorktree` (branch `wtgui/<slug>-<tag>`), unchanged.
8. Server: accept `agent: "claude"` and `model` on `POST /api/agents` (reject unknown agents with 400); probe =
   `claude --version`. Config: `claudeBin`, `claudeBudgetUsd` (default 0.50), `claudePermissionMode` (default
   `default`); `bypassPermissions` only when the existing `yolo` flag is set.

**Done when.** From `curl POST /api/agents {agent:"claude", prompt:"…"}` in the sandbox panel a haiku agent runs in its
own worktree, its tool calls and answer appear on the card, stop cancels a `Start-Sleep` run, and the Hermes `PONG`
test still passes. Add `tools/fixtures/claude-stream-*.jsonl` (sanitised: strip hook events and plugin output) and a
stub `claude` that replays one, so `tools/verify-claude-transport.mjs` can test translation with no API calls.

## Phase 2d — reply (resume) and steer

**Do.**
1. Reply = a new process with `--resume <session uuid>` in the job's worktree, same thread, next turn number. Resume
   was proven to work even from another cwd; still pass the worktree as cwd.
2. Steer = write a `user` message to the live process's stdin. It is read at the next tool boundary and lands in the
   same turn (proven). If the turn already ended the process may be idle — treat that as a normal reply.
3. Return the same shapes `steer()` returns for Hermes (`{ok, accepted, runId}` / `{ok:false, error, code}`) so the
   existing "Steer turn" button works unmodified.

**Done when.** A Claude card's reply box continues the conversation (the agent recalls earlier work), and a steer sent
during a `Start-Sleep` run changes the outcome before the sleep finishes. Extend the stub-based verify script.

## Phase 2e — approvals

**Do.**
1. A `control_request` with `request.subtype === "can_use_tool"` (`tool_name`, `input`, `tool_use_id`,
   `permission_suggestions`) becomes the existing `approval.request` event: `command` is a readable rendering of the
   tool call, `choices` is `["once","deny"]` plus `"session"` when `permission_suggestions` offers one.
2. `approve(job, choice)` answers with
   `{"type":"control_response","response":{"subtype":"success","request_id":"<id>","response":{...}}}`, where
   `once` → `{"behavior":"allow","updatedInput":<the original input>}` (proven).
3. **Verify first, then build** — these two shapes were not proven in the spike: the `deny` response
   (expected `{"behavior":"deny","message":"…"}`) and the "allow for the session" response (expected to carry
   `updatedPermissions` copied from the chosen suggestion). Add them as scenarios to `tools/spike-claude-stream.mjs`,
   run them live once on haiku, and record the real shapes in `docs/claude-phase2-handoff.md` before writing code.
4. Never auto-allow: a request the panel cannot render still waits for a person. Stopping a run answers pending
   requests with deny.

**Done when.** A haiku run that writes a file parks on the card with the allow/deny buttons; allow writes the file,
deny leaves it unwritten and the agent says so; the collapsed "✓ approved once" line persists across a reload.

## Phase 2f — dedup, UI picker, docs, verify script

**Do.**
1. **Dedup.** `claudeSessionRoots(known, { exclude })` in `lib/activity.js` skips session ids that belong to panel
   jobs, and `Scheduler.claudeRoots()` passes them (`job.agent === "claude"` → `job.sessionId`), mirroring how Hermes
   panel threads are excluded. Without this a panel Claude agent is drawn twice.
2. **UI.** An agent picker (Hermes / Claude) on the New-agent card, defaulting to Hermes and remembered per browser;
   a Claude model list (haiku / sonnet / opus aliases); a cost readout on the card from `total_cost_usd`; an
   `agent` badge on cards; a "claude found vX" chip beside the gateway chip. Keep `public/app.js`'s existing
   rendering paths — add, don't rewrite.
3. **Docs.** `README.md` and `SETUP.md`: Claude is optional, `claude` on PATH and logged in is the only requirement,
   monitoring needs nothing. Document `claudeBin`, `claudeBudgetUsd`, `claudePermissionMode`.
4. **Verify script.** Finish `tools/verify-claude-transport.mjs` (stub-driven: translation, resume, steer, approval,
   cancel) and document it next to the other `verify-*` tools.

**Done when.** One panel agent of each kind shows once (no duplicate chip), the picker spawns both, and
`verify-claude-transport.mjs` passes with no network.

## Close-out

Flip `Status: complete` in `docs/claude-phase2-handoff.md`, bump to `0.3.0`, update the table above, and sweep the
open decisions in the handoff into "decided". `verify-version-bump.mjs` then only requires the version to go up.

## Known traps

- **Windows:** no shell for `spawn`; kill trees with `taskkill /T`; the PowerShell tool refuses a bare
  `Start-Sleep` and re-runs it in the background, so use a short busy command or expect `run_in_background`.
- **Hooks and plugins run in headless sessions** and add `hook_*` noise (and cost, and latency). `--bare` skips them
  but changes auth and memory — decide it in 2c, do not default it silently.
- **Undocumented format:** pin the tested CLI (2.1.310), ignore unknown event and block types, and print unparsed
  lines to the card instead of dropping them.
- **`jobs.js` and `app.js` are huge and `upstream` edits them often.** Keep each PR small, merge `origin/main` before
  you start, and prefer adding files over reshaping them.
