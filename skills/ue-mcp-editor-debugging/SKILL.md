---
name: ue-mcp-editor-debugging
description: "Use this skill when driving a live Unreal Editor remotely through the unreal-mcp server for runtime debugging or verification: executing editor Python, saving assets, recording/inspecting Rewind Debugger sessions, sending gameplay input to PIE from outside, applying C++ changes (Live Coding vs full rebuild), or editing Blueprint graphs programmatically. Complements the unreal-mcp skill (toolset discovery basics)."
metadata:
  version: 1.0.0
---

# UE Editor Remote Debugging via MCP

Practical techniques for gathering RUNTIME ground truth from a live editor when
you only have MCP tools — no hands on keyboard/mouse. Distilled from a long
Motion Matching debugging session; each item replaced a slower or broken
alternative.

## Principle

Prefer, in order:
1. Editor **Python** (full `unreal` API) — one-liners or script files
2. **Console commands** via Python `execute_console_command`
3. Direct MCP toolsets (ObjectTools / BlueprintTools / ...)
4. **Slate UI automation / OS input injection** — last resort only

UI automation refs go stale after editor restarts, panel screenshots can be
served from cache, and scrub/record buttons are unreliable. Everything that has
a Python or console equivalent should use it.

## Full editor Python (the master key)

There is no "execute python" MCP tool, but the status bar has a console in
Python mode: find its textbox via SlateInspector (`menu` near "Python" combobox
at the bottom of the main window), `Click` it, then `Type` with `submit: true`:

- One-liner: `import unreal; unreal.log('[TAG] ...')`
- Script file: `exec(open(r'C:/path/to/script.py').read())`
- Read results back with `LogsToolset.GetLogEntries(pattern='\\[TAG\\]')`.
  Always prefix log lines with a unique tag.

Notes:
- `unreal.SubsystemBlueprintLibrary` is not exposed; Enhanced Input injection
  from Python is effectively unavailable — use OS input injection instead.
- Non-UPROPERTY / protected fields (e.g. `K2Node_FunctionEntry.MetaData`,
  `PlayerController.Player`) are inaccessible from Python and ObjectTools.

## Recipes

### Save assets
`AssetTools.save_assets(asset_paths=[])` saves ALL dirty assets in one call.
Do not fight the console or Ctrl+S; this was the reliable path.

### Rewind Debugger
- Start/stop recording with console commands `RewindDebugger.StartRecording` /
  `RewindDebugger.StopRecording` (UI record-button refs are flaky).
- Read results from a full-window screenshot right after stopping: the visible
  tail window shows the last ~10s. Blend Weights rows = actually played clips.
- Timeline scrubbing via synthetic clicks/drags does not work reliably; design
  the capture so the interesting moment is at the end of the recording.

### Sending gameplay input to PIE
Slate `PressKey` is a down+up tap — useless for held movement keys. Use OS
level injection via PowerShell (`keybd_event`/`mouse_event`):
1. Find the editor window by title with `EnumWindows` (NOT
   `Get-Process.MainWindowTitle`, which is often empty), `SetForegroundWindow`.
2. Click inside the game viewport to give PIE input focus (mouse jump while
   the game has capture will spin the camera — accept it or compensate).
3. Hold keys with down → `Start-Sleep` → up.
Watch out: a level obstacle in front of the spawn can silently zero your
movement test; verify `Speed`/`Velocity` on the AnimInstance, not just "the
script ran". `MovementIntent != 0` while `Speed == 0` means a key is held but
the pawn is blocked.

### Probing live PIE objects
PIE object paths: `/Game/<Map>.UEDPIE_0_<Map>:PersistentLevel.<Actor>...` —
but instance name suffixes change between sessions/recompiles. Resolve
dynamically via Python (`GameplayStatics.get_player_controller(world, 0)
.get_controlled_pawn()` → components → `get_anim_instance()`), then read
properties with `get_editor_property`.
For continuous conditions, arm a watcher with
`unreal.register_slate_post_tick_callback` that logs once when the condition
holds (e.g. speed > 100 for 1.5s), instead of racing manual probes.

### Applying C++ changes
- **.cpp bodies only** (no reflection/layout change): console command
  `LiveCoding.Compile` — wait for `LogLiveCoding: ... Live coding succeeded`.
- **Any UPROPERTY / UFUNCTION / member addition**: Live Coding cannot patch it.
  Close the editor (graceful `taskkill /PID` after confirming "All Saved"),
  run `Engine/Build/BatchFiles/Build.bat <Project>Editor Win64 Development
  <uproject> -waitmutex`, relaunch detached, and wait for a boot marker in
  `Saved/Logs/<Project>.log` before touching MCP again (MCP reconnects to the
  new process automatically).

### Blueprint graph editing
- `write_graph_dsl` can PARTIALLY clear a graph when it errors mid-script —
  read the graph back after any failure before assuming state.
- `get_node_type_pins` CREATES a probe node in the target graph; delete it.
- Node type ids shown by `read_graph_dsl` with a `:Suffix` (e.g.
  `Animation|EvaluateChooser:CHT_Table`) are display names; create with the
  bare id and configure afterwards.
- Deterministic path for nontrivial edits: `find_node_types` →
  `create_node` → `get_node_infos` (pin indices) → `connect_pins` /
  `set_pin_value` → `compile_blueprint`, batching via ProgrammaticToolset.
- Moving a BP variable to a C++ UPROPERTY of the same name: the BP loader
  auto-redirects existing get/set nodes on next load; expect a one-time
  `ValidateVariableNames` warning, then it is clean.
- Anim node function refs (`updateFunction` etc.) set via raw property need
  `{className: None, functionName: <name>}` and only bind reliably to native
  thread-safe UFUNCTIONs (BP functions' thread-safe flag is UI-only; unbound
  functions fail silently → T-pose).

## Verification discipline

- The user-visible symptom lives on the game thread at runtime — screenshots
  and logs beat static asset inspection. Get the numeric ground truth (dump
  the actual query/trajectory/state) before forming hypotheses.
- After every mutation: compile → check `LogsToolset` for errors → save →
  reproduce → observe. Editor state (PIE running, dirty assets, stale UI refs)
  invalidates half of all assumptions; re-Observe windows after any restart.
