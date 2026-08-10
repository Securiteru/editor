---
name: pascal-cli-build
description: Build and verify Pascal architectural scenes through the managed local CLI and MCP service, or build/test the CLI package from this repository. Use when an agent must start Pascal, create or edit a scene, connect an MCP client, or validate a CLI/runtime change.
allowed-tools: Bash Read Grep Glob
---

# Build Pascal scenes with the CLI

Use the managed Pascal CLI as the lifecycle owner and its MCP service as the scene-construction API. The CLI is not a geometry compiler: `pascal editor` starts the editor and MCP processes; MCP semantic tools create and edit the scene graph.

## Source of truth

Before constructing a scene, read the live MCP guide resource:

```text
pascal://agent-guide
```

The repository copy is `packages/mcp/src/resources/agent-guide.ts`. Confirm tool names and schemas with MCP `tools/list`; do not invent REST routes or MCP arguments.

## Start the managed local runtime

Use Node.js 22.13+ and one package runner consistently. The supported end-user flow is:

```bash
npx @pascal-app/cli editor --no-open --json
# or, after installation:
pascal editor --no-open --json
```

Useful lifecycle commands:

```bash
pascal status --json
pascal mcp status --json
pascal projects --json
pascal resume --json
pascal logs --follow
pascal doctor --json
pascal stop --json
```

Use `--foreground --no-open` when a supervisor owns the process. Otherwise the CLI manages detached editor and MCP processes. Ports are loopback-only and selected dynamically by default; `--port <n>` is only a preference and may be replaced when occupied. Do not assume port 3000, 3002, or a fixed MCP port.

The managed MCP configuration is deliberately stable:

```json
{
  "mcpServers": {
    "pascal": {
      "command": "pascal",
      "args": ["mcp", "connect"]
    }
  }
}
```

Configure supported clients with `pascal mcp setup codex` or `pascal mcp setup claude`. `pascal mcp connect` starts Pascal if necessary, discovers the current MCP port, reads the private token from the Pascal runtime directory, and proxies MCP over stdio. Never copy the token into client configuration or logs.

For a temporary isolated run, set a temporary home rather than changing project data:

```bash
tmp_home="$(mktemp -d)"
PASCAL_HOME="$tmp_home" pascal editor --no-open --json
PASCAL_HOME="$tmp_home" pascal status --json
PASCAL_HOME="$tmp_home" pascal stop --json
rm -rf "$tmp_home"
```

Managed local data is under `~/.pascal/` by default:

```text
~/.pascal/runtime/       installed editor runtimes
~/.pascal/data/pascal.db projects and scenes
~/.pascal/run/editor.json process identity and ports
~/.pascal/run/mcp-token  private MCP credential
~/.pascal/logs/editor.log
```

Do not start a second standalone MCP server against a different data directory when the managed CLI is running; that creates two unrelated scene stores. Use `PASCAL_DATA_DIR` only when intentionally running both a development editor and a development MCP server against the same database.

## Canonical scene-construction workflow

Use MCP, not hand-written graph JSON, for normal construction:

1. **Bind persistence.** For a new project call `create_project` when the tool is available. For an existing project call `list_scenes`, then `load_scene`. A bridge without a store is memory-only and will not update the browser or survive the process.
2. **Inspect context.** Read `get_scene`, `list_levels`, `get_level_summary`, and/or `get_walls` before placing geometry. Reuse returned IDs; never guess level or wall IDs.
3. **Create structure semantically.** Prefer this order:
   - `create_story_shell` for a perimeter footprint, optional slab, and ceiling;
   - `create_room` for a room polygon, zone, slab, ceiling, and walls;
   - `add_door` / `add_window` for hosted openings;
   - `furnish_room` for collision-aware catalog furniture;
   - `create_roof` for roofs;
   - `place_item` for deliberate catalog-item placement.
4. **Save progress as a draft.** Call `save_scene` with `saveMode: "draft"` while iterating. Use `saveMode: "checkpoint"` only for a meaningful durable milestone; publish it only when that is explicitly desired.
5. **Verify before handoff.** Run `validate_scene`, `verify_scene`, `check_collisions`, and `get_project_status`. Fix errors and warnings, then repeat the checks. Require `verify_scene.hasIssues === false`, an empty `check_collisions.collisions` array, `get_project_status.nodeCount > 0` for a non-empty design, and a successful save.
6. **Return the authoritative URL.** Return the `editorUrl` from MCP output. If a tool only returns an ID, call `get_project_status` and use its `editorUrl`; do not infer a route from memory.

For a brief or quick starter, `create_house_from_brief` can select and save a built-in template. Treat its `limitations` and `validation` output as requirements to resolve, then refine with semantic tools and run the full checks above.

## Coordinate and placement rules

- Pascal uses right-handed coordinates: **X/Z are the floor plan and Y is up**. Units are metres; rotations are radians.
- `wall.start` and `wall.end` are level/building-local `[x, z]` points.
- Wall-hosted door/window `position[0]` is **wall-local metres along the wall**, not a world X coordinate and not automatically the midpoint. `add_door` and `add_window` accept `t` from `0` to `1` and return the resulting `localX`; prefer these tools.
- `cut_opening.position` is also normalized `0..1` along the wall and is stored as wall-local metres.
- Every hosted opening must have a wall parent, matching `wallId`, and membership in the wall's child list. Keep the opening entirely within the wall span and within the wall height.
- Leave practical space between openings and avoid placing an opening directly at a wall endpoint. For multiple openings on a wall, inspect `get_walls` after each placement and choose explicit, separated normalized positions.
- `furnish_room` intentionally skips or nudges furniture that blocks door clear zones, leaves the room bounds, or overlaps another item. Read its `skipped` output instead of silently accepting missing furniture.

## Collision and template limitations

There is no global `no-collision` setting. The editor's interactive door/window tools reject overlapping hosted openings by default; holding **Alt** is a UI-only force-placement escape hatch and leaves a warning preview. MCP/API graph writes are not a substitute for strict validation.

`check_collisions` currently reports overlapping item footprints. It is not a complete door/window-overlap validator. `verify_scene` checks scene integrity, hosted-opening linkage, wall-local bounds, vertical bounds, and practical layout issues. Agents must therefore space doors/windows deliberately and inspect the returned wall summaries.

Known repository caveat: the current `two-bedroom` seed template initializes its doors/windows at wall-local X `0`. Do not treat that template as final geometry without inspecting and correcting every opening. Prefer `create_room` followed by `add_door`/`add_window`; if a template is required, call `get_walls`, reposition openings to explicit separated positions, then run `validate_scene` and `verify_scene` before saving a checkpoint.

Do not use raw `apply_patch` or `save_scene.graph` for ordinary construction when a semantic tool exists. If a raw graph edit is unavoidable, batch it, preserve parent/child and host references, validate it before persistence, and run all final checks again.

## Recover common failures

- **Browser appears empty:** call `get_project_status`; compare `nodeCount`, `latestVersion`, `draftVersion`, `browserVisibleVersion`, and `graphHash`. If the graph is non-empty, re-bind with the status call, save a draft, and re-run `verify_scene`.
- **`live_sync_version_conflict` / `version_conflict`:** stop mutating, call `load_scene`, inspect the current graph/version, and retry using the current version. Do not overwrite a newer browser or agent save blindly.
- **MCP is unhealthy:** run `pascal status --json`, `pascal mcp status --json`, `pascal doctor --json`, and `pascal logs --follow`. Use `pascal restart`; use `pascal stop --force` only as guarded recovery when the recorded process identity is trusted.
- **A requested port is busy:** use the port in the CLI JSON result, not the requested port.
- **A scene looks rotated:** re-check the X/Z convention. External plan formats often treat Y as north; Pascal stores the second plan coordinate directly as world Z.
- **No project persists:** confirm the MCP session is using the managed connector and that a project was bound before `save_scene`; an in-memory bridge without a store cannot update the browser.

## Build and test the CLI package from this repository

This section is for changes under `packages/cli`, not for ordinary scene construction.

Install the workspace with the declared package manager:

```bash
bun install --frozen-lockfile
```

If Bun's dependency-age policy rejects the lockfile in this development environment, the known reproducible workaround is:

```bash
bun install --frozen-lockfile --minimum-release-age=0 --force --backend=copyfile
```

Build and test the CLI package before claiming it works:

```bash
bun run --cwd packages/cli check-types
bun run --cwd packages/cli build
bun run --cwd packages/cli test
```

A TypeScript build alone does not create the portable editor runtime. For release-like lifecycle verification, run the complete sequence:

```bash
bun run --cwd packages/cli build-runtime
bun run --cwd packages/cli stage-runtime
bun run --cwd packages/cli smoke-runtime
```

`build-runtime` builds the standalone Next editor; `stage-runtime` bundles MCP and writes `packages/cli/dist/runtime/runtime-manifest.json`; `smoke-runtime` packs and installs the CLI, occupies the default port with a foreign server, starts the packaged editor, verifies HTTP health, checks repeated-start reuse, connects through managed MCP, saves a project, resumes it, runs doctor, and stops the process. Do not claim the packed CLI is verified until `smoke-runtime` passes.

For a direct local dist smoke after the runtime has been staged:

```bash
PASCAL_HOME="$(mktemp -d)" \
  node packages/cli/dist/bin/pascal.js editor --no-open --json
```

Capture the returned JSON and verify the reported editor URL and MCP port with `pascal status --json`, `pascal mcp status --json`, and an actual HTTP request to the returned editor host/port. Always stop the temporary instance afterward. Tests should cover startup, health, reuse, shutdown, process identity, occupied-port fallback, stale-state recovery, MCP connectivity, storage isolation, update rollback, and crash cleanup.

## Completion contract

Never report a Pascal build as complete from compilation alone. At minimum, provide:

- the exact CLI command and runtime/home used;
- the actual editor URL and MCP status;
- the project ID and authoritative `editorUrl` for scene work;
- node/room counts;
- `validate_scene`, `verify_scene`, and `check_collisions` results;
- the save mode/version used; and
- any remaining limitations or template corrections.

Do not expose bearer tokens, private runtime paths containing credentials, cookies, or authorization headers in output.
