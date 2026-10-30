# Per-project default agent / model

Issue: [#33 — Directory specific agent defaults](https://github.com/Millerfly91/agent-worktree-gui/issues/33)
Status: **planning** — nothing here is built yet. The work breakdown is in [plan.md](plan.md).

## The ask

Let each project carry its own default provider/model. The reader can still pick something else
when spawning and mid-thread. The existing app-wide default stays exactly as it is, and a project
with no default of its own (or one that was removed) keeps using it.

## How defaults work today (read from the tree)

| Concern | Where | Behaviour |
|---|---|---|
| App-wide default | `lib/hermes-models.js`, `server.js` `/api/models` | The gateway's `config.yaml` `model.default` / `model.provider`. The panel never sets it; the spawn form's `(default)` option means "send no model". |
| Provider | gateway | The panel does **not** choose a provider. `/api/models` lists models grouped by provider (`providers[]`), but a run only carries `model` (`lib/hermes-transport.js:173`). |
| Agent kind | `server.js:2555`, `jobs.transports` | `body.agent`, `"hermes"` unless the form says otherwise (`"claude"` is the CLI transport; `--model` at `lib/claude-transport.js:75`). |
| Per-run model | `public/app.js` `formValues()` → `jobs.create({ model })` (`lib/jobs.js:1695`) | `opts.model \|\| modelFor(thread) \|\| null`. |
| Per-thread model | `jobs.setThreadModel`, `POST /api/threads/model` | Mid-run override; replies use `body.model ?? modelFor(thread) ?? first.model` (`server.js:2699`). |
| Reset | `resetSpawnDefaults()` in `public/app.js` | Clears the model select to `""` after a spawn from a row. |
| Project config | `config.json` (`scanRoots`, `projects[]`), `patchPanelConfig` | Discovered projects usually have **no** `projects[]` entry, so an entry cannot be assumed to exist. |
| Assistant / `SPAWN IN` spawns | `lib/features/spawn.js:195` | Send no model field today, so they run on the gateway default. |

## Decisions

1. **Unit is the project, not an arbitrary directory.** The board is organised by project; worktrees
   inherit from the project that owns them. A path outside every project, or a sub-folder of one,
   resolves through the existing path→project attribution and uses that project's default.
   Per-subdirectory defaults are out of scope.
2. **A default is `{ agent?, provider?, model? }`, all optional.** Absent field = inherit. A project
   may set only a model, or only an agent.
3. **Precedence, highest first:**
   1. explicit choice in the spawn form / on the card for this run
   2. the thread's own choice (`threadModels`) — unchanged for replies
   3. **the project's default** (new)
   4. the app-wide default (gateway config) — unchanged
4. **Resolve on the server**, in one function, at spawn. The form pre-fills for visibility, but
   assistant `SPAWN IN` spawns, schedulers and API callers have no form, so the server is the only
   place that covers all of them. The form must not be the source of truth.
5. **Storage:** a `projectDefaults` map in `config.json`, keyed by the normalised project path
   (the `lib/paths.js` key rule, not the display name — names collide and paths are what the board
   already keys on). Written through `patchPanelConfig`. Removing a project's default deletes the
   key; there is no "null means something" state.
6. **Stale references are kept, not dropped.** If the saved model is no longer in the provider's
   catalogue, the run still uses it (same rule `hermes-models.js` applies to the gateway default)
   and the UI marks it "not in the current model list". The reader decides; the panel does not
   silently fall back to a different model.
7. **Provider is advisory until the gateway can take it.** The stock Runs API accepts `model` only
   (see `docs/runs-api-phase1.md`). Phase 1 stores the provider, uses it to scope the picker and
   to disambiguate identical model names across providers, and sends `model` as today. Sending a
   provider to the gateway is an open question below.

## UX

- **Settings › Projects & Folders:** each project row gets a "Default agent / model" control, the
  same model picker (`core/model-menu.js`) with a `(app default: <model>)` first entry and a
  Clear action. Showing the inherited value makes "unset" legible.
- **New agent card:** picking a project pre-selects its default in `job-model` and labels it
  `(project default)`; the `(default)` option reads `(app default)` when a project default exists,
  so the reader can see which one they are choosing. `resetSpawnDefaults()` resets to the
  project's default rather than blank.
- **Agent card:** the existing model chip gains the source of the model in its tooltip: run /
  thread / project / app.
- Changing the model on a running thread keeps working as it does now and never writes the project
  default.

## API

| Route | Auth | Purpose |
|---|---|---|
| `GET /api/settings/project-defaults` | open | `{ defaults: { [projectKey]: {agent,provider,model} } }` |
| `POST /api/settings/project-defaults` | `authorised()` | `{ project, agent?, provider?, model? }` sets; `{ project, clear: true }` removes. Validates the project exists; trims and caps strings like `setThreadModel` (200). |

The snapshot's `spawn.projects[]` entries gain a `defaults` field so the form needs no extra
round trip.

## Open questions

- **Provider on the wire.** Does the gateway's `/v1/runs` honour a provider override, or must the
  provider be encoded in the model id (e.g. `provider/model`)? Needs a check against the live
  gateway before Phase 2; until then provider is UI-only.
- **Agent default.** Should the project default also select `hermes` vs `claude`? The form has an
  agent choice, and a Claude model id is meaningless to Hermes (and vice versa). Proposed: yes,
  and the model is validated against the chosen agent's list.
- **Host-qualified keys.** `hosts.md` maps projects per host. If the same path can exist on
  several machines this config is per-install and local, which is expected; flag if not.

## Out of scope

Per-worktree or per-subdirectory defaults; per-project budget / max-turns defaults (natural
follow-up using the same map); changing the gateway's own default from the panel.
