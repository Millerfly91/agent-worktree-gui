# Plan — per-project default agent / model (#33)

Design and decisions: [README.md](README.md). Each phase is shippable on its own and leaves the
app-wide default untouched.

## Phase 1 — resolve and store (server only, no UI)

- [ ] `lib/project-defaults.js`: pure helpers — `normaliseDefault(raw)` (trim, cap 200, drop empties),
      `projectKeyFor(path)` (the `lib/paths.js` key rule), `resolveModel({ explicit, threadModel,
      projectDefault, appDefault })` returning `{ model, source }` where source is
      `run | thread | project | app`.
- [ ] `config.json`: `projectDefaults` map, read at boot, written via `patchPanelConfig`.
- [ ] `GET/POST /api/settings/project-defaults` per the README; POST behind `authorised()`;
      400 on an unknown project; `clear: true` deletes the key.
- [ ] Wire `resolveModel` into `jobs.create` (`lib/jobs.js:1695`) and the reply path
      (`server.js:2699`), and into `lib/features/spawn.js` so assistant `SPAWN IN` spawns use the
      project default instead of always the gateway default.
- [ ] Record `modelSource` on the job so the card can say where the model came from.
- [ ] Add `defaults` to `spawn.projects[]` in the snapshot.

**Tests** (`test/project-defaults.test.js`, node:test like its neighbours):
precedence table (all 16 present/absent combinations collapse to the four rules); no default → app
default untouched; cleared default → falls back; stale model still used; path in a sub-folder and
a path in a worktree resolve to the owning project; path outside all projects → app default;
POST without token → 403; oversize / non-string input rejected.

## Phase 2 — UI

- [ ] Settings › Projects & Folders: per-project picker (reuse `core/model-menu.js`), inherited
      value shown, Clear button.
- [ ] Spawn card: project change pre-selects the default, `(project default)` / `(app default)`
      labels; `resetSpawnDefaults()` resets to the project default.
- [ ] Card tooltip shows `modelSource`.
- [ ] Stale-model marker on both surfaces.
- [ ] Update `docs/using-the-board.md` for the new control and the precedence order.

**Verification:** `tools/verify-project-defaults.mjs` (one claim per harness, per repo convention):
boots the panel against a throwaway repo, sets a default, spawns with no explicit model and asserts
the run body carried it; spawns with an explicit model and asserts that wins; clears and asserts
the body carries no model. A Playwright pass over the settings control and the spawn card.

## Phase 3 — provider and agent (gated on the open questions)

- [ ] Confirm with the live gateway whether a provider can be sent; if so pass it, otherwise encode
      it in the model id or keep it picker-only and document that.
- [ ] Agent (`hermes` / `claude`) as part of the default, with the model validated against that
      agent's list.

## Risks

- **Form vs server drift:** pre-filling the form while the server also resolves can disagree. The
  form shows the server's `defaults`; the server resolves. Never the reverse.
- **Key churn:** renaming or moving a project folder orphans its default. Acceptable; the settings
  list should show orphaned keys so they can be cleared.
- **Gateway default changes:** projects without a default follow it by design.

## Definition of done

Setting, using, overriding, and clearing a project default all work from the UI; unset projects are
byte-for-byte unchanged in behaviour; `node --test` and the new verify harness pass.
