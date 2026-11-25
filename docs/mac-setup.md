# Mac setup

How we run the toolkit on macOS with Claude Code. Tested with UE 5.7.4, node 22.14.0 and npm 10.9.2 (November 2026).

There are two routes to the same plugin tools, and both need the editor running:

- **MCP:** Claude Code talks over stdio to the Node bridge (`Resources/mcp-bridge/index.js`), which calls the plugin's REST API.
- **`Scripts/ue-tool.sh`:** calls the REST API directly, from any shell.

The plugin listens on `127.0.0.1:3000`, loopback only, so the bridge and the script must run on the same Mac as the editor.

## 1. Copy the plugin and script into the project

```
YourProject/
  Plugins/UELLMToolkit/   <- Plugin/UELLMToolkit/ from this repo
  Scripts/ue-tool.sh      <- Scripts/ue-tool.sh (also json-compact.py and build.sh)
  YourProject.uproject
```

`ue-tool.sh` can sit anywhere, as long as `json-compact.py` sits next to it. Only `launch` and `restart` need the `.uproject`: they look for it in the current directory and the folders above it, so run them from inside the project.

```bash
rsync -a --exclude node_modules --exclude Binaries --exclude Intermediate Plugin/UELLMToolkit/ "<project>/Plugins/UELLMToolkit/"
```

```bash
rsync -a Scripts/ "<project>/Scripts/"
```

## 2. Build the plugin

It's a C++ editor plugin, so it needs Xcode. Close the editor, then build the project's editor target:

```bash
"/Users/Shared/Epic Games/UE_5.7/Engine/Build/BatchFiles/Mac/Build.sh" <Project>Editor Mac Development -Project="<absolute path>/<Project>.uproject"
```

A Blueprint-only project has no editor target of its own; opening it should make the editor offer to build the plugin. `Scripts/build.sh` is Windows-only. With the editor open again, check that the plugin answers:

```bash
curl -s http://127.0.0.1:3000/mcp/status
```

## 3. Install the bridge's dependencies

```bash
cd "<project>/Plugins/UELLMToolkit/Resources/mcp-bridge" && npm ci
```

Needs Node 18 or newer. `node_modules` isn't in git. This repo's `.gitignore` covers it here, but not inside your project, so add this line to the project's `.gitignore` (or `.git/info/exclude`):

```
Plugins/UELLMToolkit/Resources/mcp-bridge/node_modules/
```

Upstream's lockfile didn't match `package.json`: `npm ci` refuses it, and `npm install` crashes with `Cannot read properties of null (reading 'edgesOut')`. This fork's lockfile is fixed. If an older copy is already in a project, copy `package.json` and `package-lock.json` over from this repo and run `npm ci`.

## 4. Register the MCP server with Claude Code

Run this in the project folder:

```bash
claude mcp add ue-llm-toolkit -s local -e UNREAL_MCP_URL=http://localhost:3000 -- "$(command -v node)" "$PWD/Plugins/UELLMToolkit/Resources/mcp-bridge/index.js"
```

- **Local scope** is stored in `~/.claude.json`, keyed to the folder you ran it in. Only sessions started in that folder get the tools.
- **Absolute path to node:** Claude Code started from the desktop app may not have the shell's nvm `PATH`, so a bare `node` might not be found. `$(command -v node)` writes the current node's full path into the registration. If you remove that node version later, register again.
- **New session:** a session that was already running won't load the server. Open a new one, then call `mcp__ue-llm-toolkit__unreal_status`. `claude mcp list` shows whether it connects.
- The plugin's own `.mcp.json` uses a relative path and a bare `node`, so it only works for a session started inside the plugin folder.
- **Undo:** `claude mcp remove ue-llm-toolkit -s local`.

## 5. `ue-tool.sh` on a Mac

```bash
Scripts/ue-tool.sh status
Scripts/ue-tool.sh list
Scripts/ue-tool.sh help blueprint_query
Scripts/ue-tool.sh call asset_search '{"search_term":"Character"}'
```

- `status` prints `Editor | Game | Plugin`. A standalone game ("Play in Standalone Game", or a `-game` / `-server` launch) runs the same `UnrealEditor` binary; the script counts those PIDs as the game.
- `close` and `restart` save every unsaved package first, then send the editor one SIGTERM and a SIGKILL after 30 s. Only editor PIDs are signalled. They refuse while a standalone game runs, because closing the editor may end it; `--force` overrides.
- `launch` uses `$UE_EDITOR` or the default install in `/Users/Shared/Epic Games/UE_5.7`, and starts it with `open -n`.
- Python 3 is optional; it compacts the JSON output.
- The Mac `close`, `restart` and `launch` paths were tested against fake processes, not a real editor shutdown and relaunch.

## Things to know

- **Editor only.** The tools see the editor's copy of the level, not what's happening inside a standalone game process.
- **`task_submit` is labelled read-only but isn't:** it runs any tool by name, including the modifying ones. Don't auto-approve tools by their read-only label.
- **Node IDs live in node comments.** Every node the tool adds gets `MCP_ID:<id>` as its node comment; that's how the tool finds it again. It doesn't affect the game.
- **Edits may save.** When the editor's "Save on Compile" is set to Always, every `blueprint_modify` edit saves the Blueprint. Check for someone else's unsaved changes before editing.
