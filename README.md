# genui-codelabs

**Generative UI, step by step: A2UI, AG-UI, MCP Apps, json-render, OpenUI Lang and coding-harness integration, each in four modes (static, declarative, open-ended, mixed).**
One Colab notebook, free to run, with every example executed and rendered inline.

[Open the notebook in Colab](https://colab.research.google.com/drive/1LUdZIYEWlvV2LfKDj4GiuuHs1NdDNH8r?usp=sharing)
 ·  [Watch the demo video](https://youtu.be/AGRW1Pwfj88)

## What is generative UI?

A normal agent answers with text. A generative-UI agent decides **what interface to show**: a list of cards, a form, a chart, a whole mini-app.
The hard questions are *who decides the UI* and *how much freedom is safe*. That gives four modes, used throughout this repo:

| Mode | Who decides the UI? | Freedom | Safety model |
|---|---|---|---|
| **Static** | The developer ships fixed components; the agent only supplies data | lowest | components are trusted code |
| **Declarative** | The agent composes a UI from an allow-listed catalog (JSON or a DSL) | medium | the payload is *data*, validated against the catalog |
| **Open-ended** | The agent writes the UI itself (HTML, CSS, JS) | highest | must be sandboxed (iframe, CSP, no network) |
| **Mixed** | One response combines the modes | varies | each part keeps its own safety model |

## The six codelabs

| Part | Technology | Idea | Video |
|---|---|---|---|
| 1 | **A2UI** (Google) | declarative JSON messages (`createSurface`, `updateComponents`, `updateDataModel`) with data binding and an action loop | _<link>_ |
| 2 | **AG-UI** (CopilotKit) | the event stream between agent and app: text, tool calls, state, activities | _<link>_ |
| 3 | **MCP Apps** | MCP tools that return interactive `ui://` HTML apps rendered in a sandboxed iframe | _<link>_ |
| 4 | **json-render** (Vercel Labs) | catalog + flat JSON spec + state/actions + streamed JSON patches | _<link>_ |
| 5 | **OpenUI Lang** (Thesys) | a compact, line-oriented, streaming-friendly language the model writes | _<link>_ |
| 6 | **Harness integration** | plugging all of it into a coding-agent loop with real tool execution | _<link>_ |

| | Static | Declarative | Open-ended | Mixed |
|---|---|---|---|---|
| **A2UI** | pre-generated JSON, progressive render, click -> action -> update | catalog-validated JSONL + repair loop | custom `SandboxedFrame` component | one surface with all three + action round trip |
| **AG-UI** | tool call -> pre-built component | `ACTIVITY_SNAPSHOT` carrying A2UI | `ACTIVITY_DELTA` stream of HTML | state + tool + A2UI + sandbox in one run |
| **MCP Apps** | fixed `ui://` app | `render_ui` spec + generic viewer app | model-written `ui://` app | shell + spec + nested sandbox |
| **json-render** | hand-written spec | validated spec + patch streaming | `SandboxedFrame` added to the catalog | tabs with state, visibility, actions + widget |
| **OpenUI Lang** | hand-written Lang | library-generated prompt + validation | registered `Sandbox` component | streaming document with everything |
| **Harness** | one card component per tool | report as A2UI, json-render and OpenUI | agent-written run visualization | all three on one page |

## Quick start (Google Colab)

1. Open the notebook in Colab (link above, or **File -> Upload notebook**).
2. *(Optional)* add a free Gemini key from <https://aistudio.google.com/apikey> to Colab **Secrets** as `GOOGLE_API_KEY` and enable notebook access.
3. Run the cells **top to bottom, one part at a time**. Each part has a short explanation, the code, then the run cell with its output.

No key needed: without one, every generation step uses a clearly tagged scripted fallback, so the notebook always runs and never hits quota.

### LIVE vs OFFLINE: how generation works

Every model-written payload goes through one loop (`genui_lab/llm.py`):

```
model output -> validate against the catalog/library -> (invalid? send the errors back once) -> still invalid or no model? -> scripted fallback
```

Each result is tagged in the output:

| Tag | Meaning |
|---|---|
| `LIVE (model)` | the model's output passed validation |
| `LIVE after repair loop` | the first output was rejected, the corrected one passed |
| `OFFLINE scripted fallback` | no key, quota/network problem, or the model could not produce valid output |

The validators, renderers, sandbox and protocol code run identically in all cases. With a key you will see real model output vary between runs.
Set `os.environ["GENUI_OFFLINE"] = "1"` to force the offline path. The default model is `gemini-3.5-flash-lite`; override with `GENUI_MODEL`.

## How it is built

```
                     +-------------------- genui_lab (shared runtime) --------------------+
  model / fallback   |  llm.py      generate -> validate -> repair -> fallback            |
        |            |  catalog.py  one allow-list of components + prop validation        |
        v            |  ui.py       normalized UI tree -> escaped HTML -> sandboxed iframe |
  a2ui.py  jsonrender.py  openui.py  agui.py  mcp_*.py  harness.py  (one module per part) |
        |            |  widgets.py  open-ended HTML: no network, CSP, window.render(data)  |
        v            +---------------------------------------------------------------------+
   adapters turn each wire format (A2UI JSONL, json-render spec, OpenUI Lang, MCP app) into the same UI tree
```

Design choices worth noting:
- **One renderer, many formats.** Every declarative format is converted to the same tree, so the safety logic (HTML escaping) lives in one place.
- **Validation is the contract.** The catalog or library is the allow-list; errors are precise enough to feed back to the model.
- **Open-ended UI is always sandboxed.** `sandbox="allow-scripts"` iframe (no same-origin), a Content-Security-Policy with `connect-src 'none'`, and no external URLs allowed in generated widgets.
- **Opt-in escape hatch.** Open-ended UI exists only where a catalog or library explicitly adds a sandboxed component.

## What is real and what is re-implemented

| Piece | Status |
|---|---|
| AG-UI events and SSE encoding | the real `ag-ui-protocol` Python package |
| MCP server and client | the real `mcp` SDK over stdio, with real `_meta.ui.resourceUri` tools and `ui://` resources (MIME `text/html;profile=mcp-app`) |
| MCP Apps host | an **emulator** in the notebook: real `ui/initialize` and `ui/notifications/*` JSON-RPC over `postMessage`. A notebook iframe cannot call back into Python, so the app's own `tools/call` requests are answered from results pre-fetched from the real server |
| A2UI | a small client, validator and generator written for this repo, following the **v0.9** message format; the action message is simplified |
| json-render | the concepts (catalog, flat spec, `$state`, actions, JSON-patch streaming) re-implemented in Python; the real library is TypeScript/React and some field names are simplified |
| OpenUI Lang | a parser, stream parser, validator and renderer for the documented syntax; the component library here is our own small one |
| Harness | a minimal agent loop with real tool execution (file edits and pytest in a throw-away workspace) |

These are teaching implementations of the published specs, not the official SDKs. Specs evolve, so check the links below before relying on an exact field name.

## Repository contents

```
genui_codelabs.ipynb   the notebook (setup, shared library, six parts, gallery)
docs/                  pages rendered by the notebook (one per example) + index.html gallery
README.md
```

Running the notebook also creates the `genui_lab/` package (14 files written by `%%writefile` cells) in the Colab VM.
To publish the gallery: commit `docs/` and enable **GitHub Pages** (Settings -> Pages -> branch `main`, folder `/docs`).

## Saving your results for GitHub

1. Run the notebook in Colab so every cell shows its output.
2. **File -> Download -> Download .ipynb** (outputs are included), or **File -> Save a copy in GitHub**.
3. Download the `docs/` folder from the Files panel and commit it next to the notebook.

GitHub's notebook viewer shows the text output but not the live iframes; the `docs/` pages and the Colab link show the rendered UI.

## Troubleshooting

| Problem | Fix |
|---|---|
| Output says `OFFLINE scripted fallback` but you set a key | check the Colab secret name and that notebook access is on; the printed reason shows quota or model errors |
| `404 ... model is no longer available` | set `os.environ["GENUI_MODEL"]` to a current model name |
| `No module named 'genui_lab'` | run the setup cell and the `%%writefile` cells first, from the folder that contains `genui_lab/` |
| `No module named 'mcp.server.fastmcp'` | `pip install "mcp<2"` |
| Part 3 cell fails with an event-loop error | run it in Colab/Jupyter (it uses top-level `await`), not as a plain script |
| A rendered iframe is blank | re-run the cell; very large pages may need a larger `height` |

## References

- A2UI: <https://a2ui.org> and <https://github.com/a2ui-project/a2ui>
- AG-UI: <https://github.com/ag-ui-protocol/ag-ui> and CopilotKit's generative-UI docs
- MCP Apps: <https://modelcontextprotocol.io/docs/extensions/apps>
- json-render: <https://github.com/vercel-labs/json-render>
- OpenUI: <https://github.com/thesysdev/openui>
- Coding-harness reference for the lecture: <https://github.com/dlmastery/simple-coding-harness>

## Author

**Soham Raj Jain**: M.S. Software Engineering (Data Science), San Jose State University.
