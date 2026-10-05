---
name: comfy-mcp-fixes
description: >
  Fixes and gotchas for driving a local ComfyUI through comfy-mcp / comfy-cli and
  an in-app browser, and for custom-node web UIs that do not show up.
  Use when a ComfyUI custom node's widget/overlay is missing or empty, when
  restart_comfyui fails with no_background_server, when ComfyUI needs a restart,
  when verifying a node UI in the live ComfyUI, or when a comfy-cli server blocks
  the Desktop app from updating. Trigger phrases: "comfy mcp fixes", "node has
  no UI", "widget not showing", "restart comfy", "can't update comfy desktop".
---

# comfy-mcp fixes

Gotchas collected while building a custom ComfyUI node with a web UI and
driving it from an agent. Tested on ComfyUI Desktop, frontend 1.53, ComfyUI 0.37.
Examples assume the server is on `http://127.0.0.1:8188`; substitute your port.

## 1. Custom node JS never loads: check WEB_DIRECTORY
A pack's `web/` folder is served ONLY if its `__init__.py` exports it:
```python
WEB_DIRECTORY = "./web"
__all__ = ['NODE_CLASS_MAPPINGS', 'NODE_DISPLAY_NAME_MAPPINGS', 'WEB_DIRECTORY']
```
Verify: `GET /api/extensions` must list the pack's JS. Adding or changing
`WEB_DIRECTORY` needs a **backend restart**. Editing an already-served JS file
only needs a **page reload**.

## 2. DOM widget renders empty: use node.addDOMWidget
On this frontend, `window.comfyAPI.domWidget.addWidget(node, name, el, opts)`
does NOT throw but returns a widget named `"undefined#1"` with `element: null`.
try/catch fallbacks never fire. Always call:
```js
this.addDOMWidget("name", "custom", container, { serialize: false });
```
Data that must persist goes in a normal hidden/STRING widget (e.g.
`lora_stack_json`), not in the DOM widget.

## 3. restart_comfyui says no_background_server
comfy-cli can only stop/restart servers IT launched. A ComfyUI started by
something else (the Desktop shell, a launcher) is untracked.
1. `server_info`, then `job`/`system_stats`: confirm the queue is empty and
   nothing is generating. Never kill a server mid-generation.
2. Find the python PID (`main.py --listen 127.0.0.1 --port 8188`) and kill it
   (`taskkill /PID <pid> /T /F` on Windows; in Git Bash first
   `export MSYS2_ARG_CONV_EXCL='*'` so `/PID` is not mangled into a path).
3. `launch_comfyui` with the same args. It is tracked from then on, so later
   `restart_comfyui` works.
4. Confirm with `server_info` and `/api/extensions`.

## 4. Verify UI in a browser pane, not with screen automation
Desktop-automation access is often denied for the ComfyUI Desktop window. Use a
browser tool pointed at `http://127.0.0.1:8188` instead:
- Reload after JS edits. If the page stalls at the splash, reload again.
- Create a test node: `LiteGraph.createNode("ClassName")`, `app.graph.add(n)`.
  Wait until the workflow restore has settled or the node is dropped.
- Inspect: `app.extensions`, `LiteGraph.registered_node_types`,
  `node.widgets.map(w=>[w.name,w.type,!!w.element])`.
- Screenshot `zoom` may not crop; resize the viewport instead and reset it when done.
- Remove test nodes afterward and never save the user's workflow.

## 5. Keep the repo source and the installed pack in sync
Install a pack under development as a directory junction/symlink to the repo
folder so there is one copy: JS edits then need only a page reload, Python edits
a restart. On Windows, `python -c "import _winapi; _winapi.CreateJunction(src, dst)"`
works where `mklink` through Git Bash does not. If you must copy instead, diff
the two afterward.

## 6. Do not leave a comfy-cli server running (Desktop update loop)
A server started with `launch_comfyui` runs from the same env as ComfyUI Desktop
and blocks its updater ("can't update, loop"). Before updating or opening Desktop:
`stop_comfyui`, then kill stragglers (`taskkill /F /IM "Comfy Desktop.exe" /T`),
then launch Desktop and let it update. Afterwards verify with `/object_info` and a
real workflow, and prefer the Desktop's own server over a second one. Desktop may
show a newer version string (e.g. a master build) than `comfyui_version.py`.

## 7. Model browser shows few items / no previews, or Desktop is slow to open
That is the duplicate-model-root bug class. See the companion skill
`comfy-duplicate-model-roots`.

## More gotchas
- Do not call `launch_comfyui` and file operations in the same parallel batch;
  the server starts before the swap finishes. Run `stop_comfyui` first.
- Python `urllib` inside the ComfyUI venv can fail CivitAI with `certificate has
  expired`; build the SSL context from `certifi.where()`.
- Ollama "thinking" models burn `num_predict` on reasoning and return empty
  content; send `"think": false`.
- LoRA file headers are identical for same-rank LoRAs, so duplicate detection
  must sample file content, not just the header.
- Several packs register similarly named DOM (e.g. `.card-grid`); find a node's
  own widget element via `node.widgets.find(w => w.name === "...").element`, not
  `document.querySelector`.
- Do not copy another pack's code into yours without checking its licence; a
  GPL pack's code makes your pack GPL.
