# About this fork

Reality Adventures' copy of the UE LLM Toolkit, with the fixes we made while using it in our projects (UE 5.7 on macOS).

- **Upstream:** [ColtonWilley/ue-llm-toolkit](https://github.com/ColtonWilley/ue-llm-toolkit), which is itself forked from [Natfii/UnrealClaude](https://github.com/Natfii/UnrealClaude) by Natali Caggiano.
- **Base:** upstream `main` at `9f6a0f7` (2026-04-02, "restore UE 5.6 support"), the last upstream commit as of 2026-11-25. Upstream's own history is kept unchanged below our commits.
- **Licence:** MIT. Keep `LICENSE` and `Plugin/UELLMToolkit/LICENSE` with their copyright lines when copying the plugin anywhere.
- **Mac setup:** [docs/mac-setup.md](docs/mac-setup.md).

## Our changes

One commit each, on top of `9f6a0f7`.

| Change | Files | Tested |
|---|---|---|
| Plugin allows Mac; `EngineVersion` is 5.7.0 | `UELLMToolkit.uplugin` | Built and running in the Mac editor since April 2026 |
| Skip MCP server startup in commandlets (cook, resave), which failed on the editor's port and made UAT report a good cook as failed | `UnrealClaudeModule.cpp` | In our builds since August 2026 |
| `blueprint_modify` finds collapsed graphs by name (falls back to `GetAllGraphs`; refuses duplicate names) | `BlueprintGraphEditor.cpp` | Used on a collapsed graph. The duplicate-name error and macro graphs untried |
| `blueprint_modify` can add a Get node for a component added in the Blueprint | `BlueprintGraphEditor.cpp` | Used in a project |
| `ue-tool.sh save` reports a failed or timed-out request, so `close` stops instead of shutting down unsaved | `Scripts/ue-tool.sh` | Tested on a Mac against fake processes and a stub plugin |
| `ue-tool.sh` `status`, `close`, `restart`, `launch` work on macOS; Windows unchanged | `Scripts/ue-tool.sh` | Same. A real editor shutdown and relaunch through the script hasn't been run |
| MCP bridge: `package-lock.json` matches `package.json`, so `npm ci` works; the unused vitest dev deps are gone | `Resources/mcp-bridge/package*.json` | `npm ci` in an empty folder installs the same 90 packages as a working install |
| This file, the Mac setup guide and a pointer at the top of `README.md` | docs | |

Tried and dropped: a `Comment` node type for `blueprint_modify`. The tool stores each node's ID in that node's comment as `MCP_ID:<id>`, so a comment box would lose its text.

## Updating from upstream

```bash
git fetch upstream
git merge upstream/main
```

Expect conflicts in `UELLMToolkit.uplugin` (engine version and platform lists) and in the bridge's `package.json` / `package-lock.json`. After a merge, rebuild the plugin in a project and check `curl -s http://127.0.0.1:3000/mcp/status` before using it on a live project.

## Using it in a project

To use it in a project, copy `Plugin/UELLMToolkit/` into the project's `Plugins/` and `Scripts/ue-tool.sh` into its `Scripts/` (leaving `node_modules`, `Binaries` and `Intermediate` alone), then rebuild. To bring a change made in a project back here, copy the changed files over and commit them.
