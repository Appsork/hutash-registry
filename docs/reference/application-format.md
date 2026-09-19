# Application (`.hutash`) authoring — schema reference

Audit-based reference, produced 2026-09-12. Every claim below traces to
a file:line in this workspace or to a real shipped package; nothing
here is invented or "how it should work." This is Phase 1 source
material for a future coding-agent-facing `AGENTS.md` — not that
document itself.

**This is not a tutorial.** For a walkthrough of writing a new vertical
from scratch, use `docs/reference/create-vertical.md` (this repo) — it
is accurate and already good; this document does not replace it. What
this document adds:

1. The **complete, exhaustive schema** of `vertical.yaml` — every
   top-level key, every page field, every widget-entry field — cross-
   checked against the actual parser, not just the common case.
2. The **full runtime trace** of capability resolution: how
   `${settings.stt_model}` in a workflow step turns into a real
   `POST /packages/{id}/infer/{endpoint}` call, end to end.
3. **One real, complete, shipped `vertical.yaml`** (hutash-subs),
   quoted in full and annotated section by section — not the
   `create-vertical.md` guide's intentionally minimal "hello" example.
4. The `.hutash` package structure as independently confirmed by this
   session's own byte-level audit of a real, shipped `.hutash` file.
5. A list of concrete gaps and inconsistencies found during this audit
   — the friction points a future authoring agent will actually hit.
6. The **complete, exhaustive schema** of a `workflows/*.yaml` file —
   every step type the runner actually dispatches, every output field,
   and the SSE progress event shape a run streams to the frontend —
   cross-checked against the loader/runner source (`hutash_workflow`),
   not inferred from one example.
7. The **field-by-field `manifest.yaml` schema for an application**, at
   the same detail level `pipeline-format.md` gives
   the pipeline side — verified against the same shared Go parser, plus
   the separate `application/config/app.yaml` merge an application
   install actually goes through that a pipeline install never does.

For the widget catalogue's exhaustive prop-by-prop tables, this
document does not re-transcribe them — `packages/hutash_vertical_ui/
REGISTRY.md` (this repo) already has them, is the widget code's own
declared source of truth, and would drift from a second copy the first
time a prop changed. Read it directly. `hutash-subs/docs/reference/
VERTICAL-UI-DESIGN-SPEC.md` covers the design-token/layout-system level
above individual widgets. This document is the connective tissue: the
YAML schema those widgets slot into, and the capability system that
gets them their data.

---

## 1. `vertical.yaml` — complete schema

**Parser:** `packages/hutash_workflow/server/services/vertical_config.py`
is the single place this file is read (`_load()`, lines 28-32). The
frontend never parses YAML directly — it only ever consumes the JSON
`GET /api/v1/vertical` returns, built by `get_vertical_config()` (lines
151-192).

**There is no schema validation for `vertical.yaml`.** `_load()`
swallows any parse error or missing file into `{}`, and every getter
below defaults silently on a missing or malformed key. Contrast this
with `hutash_workflow`'s own workflow-YAML loader (`loader.py`), which
raises a `WorkflowValidationError` with a specific message on a bad
file. A vertical.yaml with a typo'd key, or a value of the wrong shape,
produces no error at all — just a page, setting, or workflow that
silently isn't there. **A future authoring agent must not rely on any
error signal from a malformed vertical.yaml; there isn't one.**

### Top-level keys

| Key | Read at | Required? | Default if absent |
|---|---|---|---|
| `app` | vertical_config.py:159 | no | `{}` |
| `pages` | vertical_config.py:163 | no | — (renderer falls back to the legacy `layout:` key if `pages` is absent) |
| `workflows` | vertical_config.py:41,168 | no | `[]` |
| `layout` | vertical_config.py:169 | no | `{}` (legacy, pre-`pages` single-workspace shape) |
| `settings` | vertical_config.py:187 | no | `{}` |
| `project` | vertical_config.py:62,104,140 | no | see below |
| `setup` | vertical_config.py:195-228 | no | `{}` |
| `ui_overrides` | vertical_config.py:191 | no | `{}` — legacy, superseded by widget-level `customWidgets` |

`app`: `{title?: string, description?: string}` — the OS shell's app
chrome (window/tab title, subtitle). Nothing else reads it.

`project`: `{subfolders?: string[], transcript_path?: string,
working_file?: string}`. Every path value is sanitized against `..`,
absolute paths, and backslashes (vertical_config.py:44-148) before use.
Defaults: `subfolders` → framework's own default list if absent or
invalid; `transcript_path` → `"transcripts/subtitles.srt"`;
`working_file` → `""` (empty = the working-file feature is off for
this vertical — it's opt-in, not a placeholder).

`setup.required_capabilities`: a list that can only ever be a subset of
what `manifest.yaml`'s own `capabilities_needed` already declares — an
entry naming a capability the manifest doesn't declare is dropped
(vertical_config.py:195-228). This block *refines presentation* of an
already-declared requirement; it cannot add a new one. The requirement
itself lives one file up, in the package's `manifest.yaml`
(`create-vertical.md` §1), not here.

**`capabilities` in the served JSON config is not a `vertical.yaml`
key at all** — it's derived live, on every request, from the app's own
`manifest.yaml` `capabilities_needed` field
(`get_declared_capabilities()`, `engine_client.py:439-452`). Don't look
for it in the YAML file; it isn't there to find.

### Page object

Backend passes pages through structurally; the authoritative shape is
the frontend contract, `PageDef` (`packages/hutash_vertical_ui/src/
types.ts:103-138`):

| Field | Type | Required? | Meaning |
|---|---|---|---|
| `id` | string | **yes** | Stable page identifier; also the navigation target other widgets' `to:` refer to |
| `label` | string | no | Nav label |
| `icon` | string | no | Nav icon name |
| `default` | boolean | no | This page is the landing page |
| `hidden` | boolean | no | Not in primary nav; reachable only via an explicit `navigate` action |
| `reset_inputs` | boolean | no | `false` means this overlay is a panel over live work and must not clear its own form on open/close |
| `overlay` | boolean | no | Opens on top of whatever's open; the underlying page stays mounted (never unmounted by opening an overlay) |
| `overlay_size` | `"md"` \| `"lg"` | no, only meaningful with `overlay: true` | `md` = 40rem, `lg` = 52rem panel width |
| `dialog` | boolean | no | Overlay draws its own content with no title bar (vs. a titled overlay panel) |
| `confirm_leave` | string | no | Shown as a leave-guard prompt when navigating away with unsaved state |
| `layout` | string | no, default `"single"` | See Layout types below — **anything not in the closed set silently falls back to `single`**, no error |
| `left_width`, `right_width`, `top_height`, `middle_height` | number \| string | no | Panel sizing for multi-panel layouts (`"auto"` is valid alongside a fixed unit like `"26rem"`) |
| `widgets` | `WidgetDef[]` | no | The page's actual content |

**Layout types** (`PageLayoutType`, types.ts:93): `single` |
`two-panel` | `three-panel` | `editor`. **This is a closed set distinct
from two other, unrelated "layout" vocabularies in this codebase** —
see Friction points below; do not assume any layout string works
everywhere just because it appears in some YAML file somewhere in the
workspace.

### Widget entry

`WidgetDef` (types.ts:85-90):

| Field | Type | Required? | Meaning |
|---|---|---|---|
| `type` | string | **yes** | Selects from the registered-widget catalogue (29 entries as of this writing — see §4) |
| `position` | string | no | Layout slot the page's `layout` type defines (e.g. `main`, `left`, `center`, `right`) |
| `bind` | string | no, input widgets only | Names the **workflow input id** this widget's value reads/writes — see §3, "the whole connection, no mapping layer" |
| `props` | object | no | Widget-specific configuration — see `REGISTRY.md` for the exact shape per widget type |

### Legacy `layout:` block (pre-`pages`)

`LayoutConfig` (types.ts:37-44): `{type: "three-panel"|"two-panel"|
"single", left?, center?, right?: PanelConfig}`.
`PanelConfig = {content: "project_list"|"workflow_view"|
"output_panel"|"settings", width?, height?, label?}`. `content` names
one of four hardcoded slots the renderer fills directly — this is the
older, less flexible shape `pages` superseded; every shipped vertical
this session inspected uses `pages`, not this.

### Settings field

`SettingFieldDef` (types.ts:68-76): `{type: string, label?: string,
description?: string, capability?: string (model_selector only),
options?, default?}`. Confirmed real `type` values in shipped
settings blocks: `model_selector`, `dropdown`, `folder_picker` (see §3
worked example) — the settings page is generated entirely from this
block; there is no separate settings-page-layout YAML.

---

## 2. Capability / `model_selector` resolution — full runtime trace

How a vertical asks for "whatever is installed that can transcribe"
without ever naming a specific model, traced end to end:

1. **Package-level declaration** — `manifest.yaml`'s
   `capabilities_needed: [stt, translate]` (top-level, package root,
   NOT `vertical.yaml`). This is what gates first launch
   (`setup_guard`, mounted automatically by `VerticalApp`) and what
   `get_declared_capabilities()` reads.

2. **Settings-level declaration** — `vertical.yaml`'s `settings:` block
   declares a `model_selector` field per capability the user might want
   to pin (`capability: stt`). This is what the generated Settings page
   renders as a picker. Leaving it on "Automatic" means the field
   resolves to nothing, which triggers step 4 below.

3. **Workflow-level reference** — a workflow step declares
   `type: capability`, `capability: stt`, `model: ${settings.stt_model}`.
   The model is *never* named directly in the workflow file — always a
   template reference to whatever the Settings page holds (or empty).

4. **Runtime resolution** — `packages/hutash_workflow/steps/
   capability_step.py`:
   - `execute()` (line 660) resolves `${settings.stt_model}`. If empty
     (line 674), it calls `_only_installed_model()` (line 798).
   - `_only_installed_model()` calls the ONE shared implementation,
     `installed_with_capability()` (`packages/hutash_workflow/
     capabilities.py`), and takes the first match in the engine's own
     catalogue order — **a fallback, never a preference**; an explicit
     Settings choice always wins over this (capability_step.py doc
     comment, lines 806-820).
   - `installed_with_capability()` intersects two engine facts:
     `GET /packages` (authoritative for *installed*, but knows nothing
     about capability) and `GET /catalogue?capability=X`
     (authoritative for *capability*, but knows nothing about
     install state). Note: **being listed by `/packages` is not being
     installed** — a completed uninstall leaves an `installed: false`
     tombstone row (this session's own earlier `handleDelete` reading
     confirms this daemon-side; `is_installed()` on the Python side
     reads the flag, never row presence, for exactly this reason).
   - Once a model id is resolved, the step reads THAT model's own
     manifest to find the capability's real HTTP endpoint
     (`manifest.capabilities.<name>.endpoint`) — **the endpoint is
     never assumed to equal the capability name.** Documented failure
     this exists to prevent: `qwen3-1.7b` declares capability
     `improve` served at `/generate`, not `/improve` — assuming they
     matched 404'd every call (capability_step.py module docstring).
   - The actual call: `POST /packages/{model}/infer/{endpoint}` — the
     engine's transparent inference proxy.

5. **The Settings picker's own options** —
   `server/routers/settings.py`'s `GET /models` groups installed models
   by every capability the vertical's `settings:` block names, so each
   `model_selector` widget's dropdown is populated from real installed
   state, never a static list.

**Pickers get their groups too.** `GET /models` serves a group for every
capability a `model_selector` names, not only the ones `capabilities_needed`
declares — so a vertical offering Hanu's Assistant picker (`capability: llm`)
gets an `llm` group whether or not it needs a language model to run. A
language model's capability is `llm`; the `chat` pseudo-capability and its
expansion were removed on 2026-09-14.

**`featureRegistry.ts` (hutash-studio) is unrelated** — it's Studio's
own private sidebar/feature-identity registry (display name, icon,
color, per-feature project subfolders), not a shared capability
resolution mechanism. It has no bearing on how Subs/Podcast/Dub resolve
`${settings.X_model}`.

---

## 3. Worked example — hutash-subs, in full

`hutash-subs-files/hutash-subs/application/vertical.yaml`, complete,
annotated section by section. This is a real, shipped, working
vertical — not a teaching skeleton.

**A note on which copy this is, and why:** checked against
`hutash-registry`'s actually-published `apps/hutash-subs.hutash` for
this revision, and **the two are not the same file** — unlike the
pipeline packages (§7 of the model-pipeline authoring reference), this
is not a case of "identical mirrors." The published package's
`vertical.yaml` is 367 lines against this repo's 440, has **no `hanu`
page at all**, has no `llm_model` entry in `settings:`, and lays its
editor out with separate top-level `export`/`styles` pages rather than
the `tabbed_panel` composition shown below. The version quoted here is
the current `hutash-subs` repo source — chosen deliberately over the
stale published copy, because it's the one that actually demonstrates
`hanu_panel` and `tabbed_panel`, both real, currently-registered
widgets (§4 of `REGISTRY.md`) that a from-scratch authoring reference
needs to show working. **This means what's live in `hutash-registry`
right now is missing a real, shipped feature (Hanu) relative to this
repo's own source** — worth treating as an actual product gap, not
just a documentation discrepancy; flagged again in §5's friction points.

```yaml
app:
  title: Hutash Subs (Beta)
  description: Transcribe and translate video subtitles locally

project:
  subfolders: [media, transcripts, translations, exports]
```
`subfolders` extends the framework's `[media, exports]` default with
the two folders this app's own workflows actually write into
(`transcribe` → `transcripts/`, `translate` → `translations/`). The
file's own comment notes these four subfolders used to be compiled
into the shared package, which is why Podcast and Dub were both given
a `translations/` folder neither ever writes to — a real, historical
bug this per-vertical declaration fixed.

```yaml
pages:
  - id: projects
    label: Projects
    default: true
    layout: single
    widgets:
      - type: section_header
        position: main
        props: {title: Your Projects, action_label: New Project, action: navigate, to: new_project}
      - type: project_grid
        position: main
        props: {show_thumbnail: true, show_status: true, create_button: false, create_opens: new_project, opens: editor}
```
Landing page. `create_button: false` on the grid is deliberate — the
`section_header` above it already offers "New Project"; a second
button with the same label would be a duplicate control on the same
page.

```yaml
  - id: new_project
    label: New
    hidden: true
    overlay: true
    dialog: true
    layout: single
    widgets:
      - type: file_upload
        bind: media_file
        props: {label: Drop video or audio here, accept: [video/*, audio/*]}
      - type: text_input
        bind: project_name
        props: {label: Project name, placeholder: Untitled project, autofill_from: media_file}
      - type: dropdown
        bind: source_language
        props: {label: Spoken language, options_from: "workflow:transcribe.inputs.source_language"}
      - type: dropdown
        bind: target_language
        props:
          label: Subtitle language
          options_from: "api:/available-languages"
          prepend: [{value: same, label: "Same as source (transcription only)"}]
      - type: action_button
        props:
          label: Generate Subtitles
          action: run_workflow
          requires: [media_file]
          workflow: transcribe-translate
          routes: [{when: {target_language: same}, workflow: transcribe}]
          create_project: "${inputs.project_name}"
          then_navigate_to: editor
          navigate_params: {projectName: "${inputs.project_name}"}
      - type: progress_steps
        props: {visible_during: workflow_run}
```
The single most instructive page in this file. Three things worth
naming explicitly:

- **`bind:` is the entire data-binding mechanism.** Every control's
  value lands under that exact key in the workflow's `inputs`; there
  is no separate mapping/adapter layer anywhere.
- **`target_language`'s `options_from: api:/available-languages`** is
  deliberately narrower than the workflow's own full language list —
  the endpoint filters to languages an actually-installed translation
  model can produce, by reading that model's manifest. The comment in
  the real file is explicit that the alternative (offering the
  workflow's full menu) would let a user pick a language whose model
  isn't installed, and fail on Run instead of failing to appear at all.
- **`routes:` is conditional workflow dispatch from one form.** The
  user never picks a workflow by name — picking `target_language: same`
  silently routes the SAME button to `transcribe` instead of
  `transcribe-translate`. `create_project` + `then_navigate_to` fire
  before the workflow even runs, so the created project has somewhere
  for the transcript to land the moment it exists.

```yaml
  - id: editor
    label: Editor
    hidden: true
    layout: three-panel
    left_width: auto
    right_width: 26rem
    confirm_leave: "Leave this project? Unsaved subtitle edits will be lost."
    widgets:
      - type: sidebar
        position: left
        props:
          label: Sections
          items:
            - {label: Projects, icon: folder, action: navigate, to: projects}
            - {label: Models, icon: marketplace, action: navigate, to: models}
            - {label: Hanu, icon: chat, action: navigate, to: hanu}
            - {label: Settings, icon: gear, action: navigate, to: settings}
            - {label: Save, icon: disk, action: save}
      - type: video_player
        position: center
        props: {show_subtitles: true}
      - type: waveform_timeline
        position: center
        props: {show_segments: true, draggable: true, click_to_seek: true}
      - type: tabbed_panel
        position: right
        props:
          label: Editor panels
          default_tab: subs
          tabs:
            - id: subs
              label: Subs
              widgets:
                - type: subtitle_list
                  props:
                    editable: true
                    show_timestamps: true
                    click_to_seek: true
                    follow_playback: true
                    save_endpoint: "api:/projects/{project_id}/transcript"
                    working_path: transcripts/subtitles.ass
            - id: style
              label: Style
              widgets:
                - type: dropdown
                  bind: platform
                  props: {label: "Where will this be posted?", options_from: "api:/platforms", persist: true}
                - type: style_picker
                  bind: style_id
                  props: {styles_from: "api:/styles", persist: true, default: classic}
            - id: export
              label: Export
              widgets:
                - type: export_controls
                  props:
                    filename: subtitles
                    save_folder: exports
                    styled_format: mp4
                    preview_lines: 5
                    formats:
                      - {value: srt, label: "SRT (SubRip)"}
                      - {value: vtt, label: "VTT (WebVTT)"}
                      - {value: txt, label: "Plain Text"}
                      - {value: mp4, label: "Video with Subtitles (MP4)"}
```
The main work surface, and the widget catalogue's one genuinely
recursive structure: `tabbed_panel` is itself a widget, and each of its
`tabs` entries carries its OWN `widgets:` list. Everything else in this
schema is a flat page → widgets list; this is the only place a widget
contains widgets.

`left_width: auto` (a string, not a number) lets the sidebar rail
expand/contract without a fixed column clipping its labels — confirmed
distinct from `right_width: 26rem`, a fixed unit, on the same page.
`confirm_leave` is per-page, not global — Settings and Models both
still open as overlays over this same editor and do NOT trigger it,
because an overlay never unmounts the page underneath it.

The `subtitle_list` widget's `working_path` pointing at `.ass` (not
`.srt`) is a real, load-bearing decision documented in the file's own
comment: SRT can only encode a cue's start/end time, so round-tripping
through it silently discarded the per-word timings and style
information this app actually produces. `.ass` is the working format;
`.srt` is written alongside it purely because every external player
and upload form expects one.

`style_picker`'s `default: classic` is pinned to equal
`registry.DEFAULT_STYLE` by a real test
(`tests/unit/frontend/verticalContract.test.ts`) — not a coincidence,
a checked invariant.

```yaml
  - id: models
    label: Models
    overlay: true
    overlay_size: lg
    layout: single
    widgets:
      - type: models_page
        position: main
        props: {capability_labels: {stt: Transcription, translate: Translation}}

  - id: hanu
    label: Hanu
    overlay: true
    overlay_size: md
    layout: single
    widgets:
      - type: hanu_panel
        position: main
```
`models_page` and `hanu_panel` are each a single widget filling the
whole page — `capability_labels` on the former is the only thing this
vertical customizes; the widget itself sources the actual marketplace
data. `hanu_panel` takes no props at all — Hanu reads the open
project, the visible transcript, and the current page from context
server-side rather than being told anything here.

```yaml
workflows:
  - {id: transcribe, file: workflows/transcribe.yaml, label: Transcribe, icon: mic, description: Convert speech to subtitles}
  - {id: translate, file: workflows/translate.yaml, label: Translate, icon: globe, description: Translate existing subtitles to another language}
  - {id: transcribe-translate, file: workflows/transcribe-translate.yaml, label: "Transcribe + Translate", icon: languages, description: Transcribe and translate in one step}

settings:
  stt_model:
    type: model_selector
    capability: stt
    label: Transcription Model
    description: Leave on Automatic to use the best installed model
  translate_model:
    type: model_selector
    capability: translate
    label: Translation Model
    description: Leave on Automatic to pick the model for the chosen language
  llm_model:
    type: model_selector
    capability: llm
    label: Assistant Model
    description: Which model answers Hanu. Automatic uses the first installed model that can serve a chat turn.
  default_format:
    type: dropdown
    options: [{value: srt, label: "SRT (SubRip)"}, {value: vtt, label: "VTT (WebVTT)"}, {value: txt, label: "Plain Text"}]
    default: srt
    label: Default Export Format
  output_dir:
    type: folder_picker
    default: null
    label: Output Folder
    description: Leave empty to save in the project folder
```
`workflows:` is a lookup table `pages` widgets reference by `id` (see
the `new_project` page's `routes:`, above). `settings:` shows all four
real field types this session confirmed: three `model_selector`
entries (each tied to a capability, per §2 above), a plain `dropdown`,
and a `folder_picker` — confirming `model_selector` is one of several
settings field types, not the only shape that block supports.

---

## 4. The `.hutash` package structure (applications)

Confirmed by this session's own byte-level audit of a real, shipped
package (`apps/hutash-studio.hutash` in `hutash-registry`, ~43.7 MB,
self-contained). Matches `HUTASH_FORMAT.md` §2.2/§4 exactly:

```
<app-id>.hutash (zip)/
├── manifest.yaml                 ← identity ONLY: name, version,
│                                    type: application, license, gpu.
│                                    No ui:/capabilities: block — an
│                                    app renders its own UI.
└── application/
    ├── config/
    │   └── app.yaml              ← source: (git/self-bootstrap/pypi/
    │                                local), entrypoint, ports, health
    └── packages.yaml             ← shape depends on source.type — a
                                     frozen `uv pip freeze` snapshot for
                                     a git-sourced app, hand-written for
                                     a `local`-sourced one (Studio
                                     itself)
```

No `resources/` — forbidden for this type
(`HUTASH_FORMAT.md` §2.4's requirements table). No `launch.yaml` —
that file name is `model_pipeline`-only; every real shipped
`application` package folds the same information (`entrypoint`/
`ports`/`health`) into `application/config/app.yaml` instead. The 29
widget-type list, the capability-resolution mechanism (§2 above), and
`vertical.yaml` itself all live *inside* `application/` for a vertical
specifically — they are additional content this package type carries,
not a change to the three-layer shape itself.

For the `.hutash` vs `.hutashm` distinction (why an application stays
`.hutash` while a model pipeline is now `.hutashm`), see
`package-types.md` in this same directory.

---

## 5. `workflows/*.yaml` — complete schema

**Parser:** `hutash_workflow/loader.py`'s `load_workflow()` (this repo,
`packages/hutash_workflow/`) — the same package `vertical_config.py` (§1)
belongs to. **Unlike `vertical.yaml`, a malformed workflow file is a hard
error at load time**, not a silent default: `load_workflow` raises
`WorkflowValidationError` for a missing required field, a dangling
`${...}` reference, or a workflow-composition cycle/over-deep nesting —
this is the asymmetry §1 already flags from the other side.

### Top-level keys

| Key | Required? | Notes |
|---|---|---|
| `id` | **yes** | Also the filename stem — the runner loads `<workflows_dir>/<id>.yaml`. |
| `name` | **yes** | Display name; not otherwise used by the runner. |
| `version` | **yes** | Any scalar — coerced to `str()`. |
| `inputs` | no (default `{}`) | Map of input id → input spec (below). |
| `steps` | **yes, non-empty** | List of steps (below) — zero steps is a load-time error. |
| `outputs` | no (default `{}`) | Map of output id → output spec (below). |

### `inputs.<id>`

| Field | Required? | Meaning |
|---|---|---|
| `type` | **yes** | One of `file`, `text`, `dropdown`, `slider`, `toggle`, `number`, `model_selector`, `folder_picker` (`models.py`'s own comment) — this is metadata for validating/defaulting a run's inputs, not a widget catalogue; the workflow file never renders anything itself. |
| `label` | **yes** | Informational (`GET /workflows/{id}` exposes it); not read by the runner. |
| `required` | no, default `true` | Enforced by `WorkflowRunner._resolve_inputs`: a required input with no value and no `default` raises before any step runs. |
| `default` | no | Fills an unsupplied input. |
| `options` | no | A plain inline list for a `dropdown`-shaped input — no `options_from`-style dynamic loading; that's `vertical.yaml`'s own dropdown widget, a different layer. |
| `accept` | no | MIME patterns, `file`-typed inputs only. |

**This id list is a contract, not a form.** A `vertical.yaml` page's
widgets `bind:` to these same ids, and `action_button`'s `requires:` plus
its "filter inputs to what the chosen workflow declares" behaviour
(REGISTRY.md) is what actually connects the two files — the workflow
YAML itself never renders UI.

### Steps — `steps[]`

| Field | Required? | Meaning |
|---|---|---|
| `id` | **yes**, unique per workflow | What `${id.output}` / `${id.outputs.x}` reference. |
| `name` | no, defaults to `id` | Shown in progress events. |
| `type` | **yes** | `capability`, `ffmpeg`, `formatter`, `workflow`, `file_read`, or `file_write` — `runner.py`'s `_STEP_HANDLERS` dispatch table. Anything else fails at RUN time (not load time) with `unknown step type`. |
| *everything else* | step-specific | Collected into `step.config` (every raw key except `id`/`name`/`type`) — each handler reads what it needs by key. |

Every step's `config` values may contain `${...}` references, resolved
recursively through dicts and lists before the handler runs (see
"Variable resolution" below).

#### `type: capability` (`steps/capability_step.py`)

Calls a model through the engine's transparent inference proxy:
`POST /packages/{model}/infer/{endpoint}`.

| Config key | Required? | Meaning |
|---|---|---|
| `capability` | **yes** | e.g. `stt`, `translate`, `tts`, `llm`. |
| `model` | no | Usually `${settings.<x>_model}` — see "Model resolution" below. Left unresolved, the step calls `_only_installed_model()`: the first model the engine reports installed for this capability, in catalogue order. Zero installed is a hard failure naming the capability. |
| `input` | no | Resolved value sent as the request body. |
| `params` | no | Resolved dict, sent as sibling form fields (file input) or merged into the JSON body (non-file input). |

**The endpoint is never the capability name** — it's read from the
resolved model's own `GET /packages/{model}/manifest` →
`capabilities.<name>.endpoint` (cached per engine+model), falling back
to the capability name only when that manifest can't be read.

**Wire shape**, from the model's own declared inputs: a file-shaped
`input` (`{"path": ...}` or raw bytes) is sent multipart, keyed `"file"`;
anything else is sent as JSON `{<key>: input, **params}`, where `<key>`
is resolved from the model's manifest (its sole required non-file input
when there's exactly one; `"input"` otherwise). A binary response (e.g.
TTS audio) is written to `work_dir` and returned as a file-shaped
`{path, format, content_type, size_bytes}` — never raw bytes — so it
flows into a following `ffmpeg` step unchanged.

**Response unwrapping** picks ONE field from the model's full JSON
response, richest first: `segments` (has timing) → `text` → `transcript`
(whisper-tiny's own field name) → `translated_text` → `response` (a
plain LLM reply) → the whole dict if none match.

**`capability: translate` with segment-shaped input** (a
`{start,end,text}` dict list, or a formatted SRT/VTT string from an
upstream `transcribe` sub-workflow) takes a SEPARATE path: contextual
sliding-window translation, one call per segment with up to 2 neighbors
either side (shrinking to fit a ~400-token budget; sentences are split
first, since Marian-family models translate one sentence at a time),
emitting its own intermediate progress events between the step's own
0%/100%. Free-text (quick-translate) input skips this and goes through
the plain single-shot path.

#### `type: ffmpeg` (`steps/ffmpeg_step.py`)

| Config key | Required? | Meaning |
|---|---|---|
| `action` | **yes** | `extract_audio`, `convert_format`, `get_duration`, `get_metadata`, `burn_subtitles_ass`, `replace_audio`. |
| `input` | **yes** | File-shaped (`{"path": ...}`) or a literal path string. |
| `params` | action-specific | See below. |

| Action | `params` | Returns |
|---|---|---|
| `extract_audio` | `format` (default `wav`), `sample_rate` (default `16000`), `channels` (default `1`) | `{path, format, sample_rate, channels}` |
| `convert_format` | `format` (**required**) | `{path, format}` |
| `get_duration` | — | a float (seconds), via ffprobe |
| `get_metadata` (also the fallback for any action not otherwise matched) | — | the full ffprobe JSON dict |
| `burn_subtitles_ass` | `subtitles` (**required**, path to a `.ass` file), `height` (optional, scales before burn-in) | `{path, format: "mp4"}` — uses the `ass` libass filter (never `subtitles`, which also accepts a style-flattening `force_style`); H.264 encoder auto-probed per host |
| `replace_audio` | `audio` (**required**, path), `audio_codec` (default `aac`), `format` (default `mp4`), `shortest` (bool, default off) | `{path, format}` — `-c:v copy` (frames untouched), explicit stream mapping (`-map 0:v:0 -map 1:a:0`), and deliberately does NOT truncate to the shorter stream unless `shortest: true` |

#### `type: formatter` (`steps/formatter_step.py`)

| Config key | Required? | Meaning |
|---|---|---|
| `format` | **yes** | `srt`, `vtt`, `txt`, `json` (built in), or a name a HOST app registered via `register_format()` (a caption-styling `ass` renderer is the real example — the styling catalogue is a product decision, never generic runner code). |
| `input` | **yes** | A segment list, a capability response dict (its `segments` field is used), or an SRT/VTT string (auto-parsed). |
| `params` | no | Passed through to a registered renderer only. |

Rendering a TIMED format (`srt`/`vtt`/any registered one) from input that
parses to zero cues but is non-empty text is a hard error (a model
returned a flat transcript, not timed segments) — never a silently empty
file.

#### `type: workflow` (`steps/workflow_step.py`) — composition

| Config key | Required? | Meaning |
|---|---|---|
| `workflow` | **yes** | The sub-workflow's `id`, loaded from `<workflows_dir>/<id>.yaml`. |
| `inputs` | no | Map of the CHILD workflow's input ids → `${...}` expressions resolved against the PARENT's context. |

Runs through the SAME `WorkflowRunner` instance (shares
`engine_url`/`settings`/`workflows_dir` — not a separate execution
context). The child's full `outputs` dict becomes this step's `.output`
AND is threaded into `.outputs.<name>` (only `type: workflow` steps
populate `.outputs` — see "Variable resolution"). Circular references
and nesting deeper than **3 levels** (`MAX_WORKFLOW_NESTING_DEPTH`) are
rejected at LOAD time, recursively, before anything runs.

#### `type: file_read` / `type: file_write` (`steps/file_step.py`)

| Type | Config keys | Notes |
|---|---|---|
| `file_read` | `input` (**yes**, file-shaped or literal path), `parse` (no; `srt`, `vtt`, `json`, `auto`, or omitted for raw text) | `auto` sniffs the extension, then a `WEBVTT` header, defaulting to `srt`. |
| `file_write` | `input` (**yes**), `path` (**yes**, resolved), `format` (`srt` default, `vtt`, `txt` for a segment list — a plain string input is written as-is regardless of `format`) | A `${project.<subfolder>}` reference in `path` is **not yet wired** (the module's own docstring) — it resolves through the ordinary "no such step" error today; none of the shipped subs/podcast/dub workflows use `file_write`, for exactly this reason. |

### Outputs — `outputs.<id>`

| Field | Required? | Meaning |
|---|---|---|
| `type` | **yes** | `file`, `text`, or `subtitle_editor` (`models.py`'s comment) — a HOST-app value; the runner itself only acts on `words_from`/`primary` below. `subtitle_editor` is what `_transcript_output` (`routers/workflows.py`) looks for to decide which output is written to the project's `project.transcript_path` and shown by the editor. |
| `source` | **yes** | A `${...}` reference, resolved once all steps complete. |
| `label` | **yes** | Display name. |
| `formats` | no | Which export formats this output can render to (informational — consumed by `export_controls`, REGISTRY.md). |
| `primary` | no, default `false` | At most ONE output per workflow may set this — a second `primary: true` is a load-time error. Decides which output an editor opens / an export defaults to when a workflow declares several. |
| `words_from` | no | The id of ANOTHER output of this SAME workflow carrying this cue list's per-word timings (SRT/VTT can't encode a word time, so the rendered `source` output has already lost them — `words_from` points at the pre-formatting segments that still have them). Validated at load time: naming a non-existent output is an error. |

**`type` here is a different vocabulary from a model pipeline's
`ui.capabilities.<name>.outputs.<name>.role: primary`**
(`pipeline-format.md` §6) — same word "primary" even,
but one is a workflow-output flag `_resolve_outputs`/`_transcript_output`
actually check, the other is an opaque field inside a pipeline
manifest's `ui:` tree that only Studio/a vertical interprets. A value
valid in one means nothing in the other.

### Variable resolution — `${...}` (`resolver.py`)

Context available to every step, built up as the run progresses:

```
{
  "inputs":   {<input_id>: <value>, ...},
  "settings": {<setting_id>: <value>, ...},
  "<step_id>": {
    "output":  <that step's raw output>,
    "outputs": {<name>: <value>, ...}   # only for type: workflow steps
  },
  ...
}
```

- `${inputs.<id>}` — the resolved run input.
- `${settings.<key>}` — looked up in the `settings` dict the
  `WorkflowRunner` was built with (the vertical's persisted Settings, or
  whatever a per-run resolver returned — see below). **Never validated
  at load time** (`loader.py` accepts any `settings.*` reference
  unconditionally, since settings live in `vertical.yaml`/user prefs,
  invisible to the workflow loader) — a typo'd setting key is only
  caught at RUN time, as `VariableResolutionError`.
- `${<step_id>.output}` — that step's raw result.
- `${<step_id>.outputs.<name>}` — only resolvable for a `type: workflow`
  step.
- A string that is EXACTLY one `${...}` reference resolves to the
  referenced value's real type (a list, dict, bytes) — this is what lets
  a step's `input:` carry a segment list through unchanged, not a
  stringified version of it. A `${...}` embedded alongside other text is
  stringified and substituted in place.

**Load-time reference validation** (`loader.py`, workflows only —
`vertical.yaml` has no equivalent, per §1): every `${...}` in a step's
config or an output's `source` must resolve to a known input, an
(unchecked) `settings.*`, or a PRECEDING step's id — referencing a step
that runs AFTER the current one, or a name matching nothing at all, is a
`WorkflowValidationError` raised before the workflow ever executes,
naming exactly which step/output and which reference.

### `${settings.X_model}` resolution, and the per-run override seam

The engine-facing half of this — how a resolved model id turns into a
real `POST .../infer/{endpoint}` call — is documented in full in §2
above ("Capability / `model_selector` resolution"). This section covers
the OTHER half: how `context["settings"]` itself gets built for one run.

By default it's exactly what `GET /settings` returns — the vertical's
persisted Settings page values, unchanged. But **a vertical may register
a per-run override**
(`hutash_workflow.server.routers.workflows.set_settings_resolver`,
called once from the app's own `api/main.py` at startup — the module's
own docstring calls this "the one app-specific seam"): an async
`(settings, inputs) -> settings` hook, awaited before every run, that
can change what a `${settings.X}` reference resolves to based on THIS
run's own inputs.

Real example — hutash-subs's `resolve_translate_model`
(`api/services/translation_languages.py`), registered via
`create_app(spec=SPEC, settings_resolver=resolve_translate_model)`:

1. If the user explicitly picked a model AND the "Choose Translation
   Model" toggle is on (`translate_enabled` + `translate_model` both
   set) — use it. Ignoring a stale `translate_model` while the toggle is
   off is what makes switching the toggle off actually mean "back to
   automatic."
2. Otherwise, pick whatever installed model supports the run's OWN
   `target_language` input — the user picked a language, so which model
   serves it is resolved automatically, not left to a fixed setting that
   might not cover it.
3. Otherwise, left exactly as stored — `capability_step`'s own "no model
   installed" error is the honest answer, not a guess.

Without a registered resolver, `${settings.X_model}` is simply whatever
Settings holds — right for a vertical whose model choice never depends
on the run itself.

### SSE progress — `POST /workflows/{id}/run`

Request: `multipart/form-data` (file fields saved to a run-scoped
scratch dir under a `{"path", "filename"}` context value; non-file
fields as form strings) or a plain JSON body. Two keys are popped out of
`inputs` before the workflow ever sees them: `project_name` (creates a
project) / `project_id` (attaches to an existing one) — never real
workflow inputs.

Response: `text/event-stream`. Every line is `data: <json>\n\n`; exactly
one final event always arrives, even if the runner itself crashes (a
queued `None` sentinel in a `finally` guarantees it — the router's own
comment calls out the earlier bug where this hung the request forever).

**`{"type": "progress", ...}`** — zero or more, per step:

```json
{"type": "progress", "workflow_id": "...", "step_id": "transcribe",
 "step_name": "Transcribe Speech", "progress": 0.0,
 "message": "Transcribe Speech…", "state": "running"}
```

`state` is `running` (emitted once, `progress: 0.0`, before the step
starts), `complete` or `error` (once, `progress: 1.0`, after). Only
`type: capability` steps can emit MORE than these two — the contextual
translation path reports `progress` per segment translated, since it
makes many engine calls inside one step.

**`{"type": "result", ...}`** — exactly once, last:

```json
{"type": "result", "run_id": "<32-hex>", "workflow_id": "...",
 "outputs": {"subtitles": {"path": "...", "file": "subtitles.srt", "run_id": "..."}},
 "steps": [{"step_id": "...", "output": "...", "state": "complete", "error": null}],
 "state": "complete", "error": null,
 "project": {"id": "...", "name": "..."},
 "saved_to": {"subtitles": "transcripts/subtitles.srt"},
 "working_file": "notes.json"}
```

- `outputs` — every declared output, resolved. A file-shaped one gets
  `file` (its name under this run) and `run_id` added ONLY when it
  actually lives inside the run's own scratch dir — fetch it at
  `GET /runs/{run_id}/files/{file}`. Run directories, and every such
  URL, do not survive a restart.
- `project` / `saved_to` / `working_file` are present only when the
  request named or created a project. `saved_to` maps each output id to
  where it landed inside the project; `working_file` (present only when
  `vertical.yaml`'s `project.working_file` is set) is the path to the
  ONE JSON record holding every text output the run produced, keyed by
  output id.
- **A FAILED run (`state: "error"`) still carries a partial
  `outputs`/`saved_to`** — every output whose source steps completed
  before the failure is resolved and persisted; only outputs that
  genuinely can't be reached (their step never ran) are dropped. A
  four-step dub that dies on the final mux still keeps its transcript,
  translation, and generated voice-over on disk — deliberate (see
  `runner.py`'s own comment on `_resolve_outputs`), not a bug to route
  around.

Related, narrowly-scoped endpoints: `GET /workflows` (registered
workflow ids), `GET /workflows/{id}` (its `id`/`name`/`version`/
`inputs`/`outputs` — never its `steps`, which are not exposed over the
wire), `POST /runs/save-to-project` (a server-side copy of a run's own
file into a project, for a binary deliverable like a dubbed MP4 that a
browser should never have to round-trip).

### Complete annotated example — hutash-subs's `transcribe-translate.yaml`

The same workflow §3's worked `vertical.yaml` wires up (`new_project`'s
`action_button` names `workflow: transcribe-translate`), quoted in full:

```yaml
id: transcribe-translate
name: Transcribe + Translate
version: 1.0

inputs:
  media_file:
    type: file
    accept: [video/*, audio/*]
    label: Drop your video or audio here
    required: true
  source_language:
    type: dropdown
    options:
      - {value: auto, label: Auto Detect}
      - {value: en, label: English}
      - {value: hi, label: Hindi}
      - {value: ta, label: Tamil}
    default: auto
    label: Source Language
  target_language:
    type: dropdown
    options:
      - {value: hi, label: Hindi}
      - {value: ta, label: Tamil}
      - {value: te, label: Telugu}
      - {value: kn, label: Kannada}
      - {value: ml, label: Malayalam}
      - {value: bn, label: Bengali}
      - {value: mr, label: Marathi}
    label: Translate To
    required: true

steps:
  # Reuse the transcribe workflow as a sub-step
  - id: transcribe
    name: Transcribe Speech
    type: workflow
    workflow: transcribe
    inputs:
      media_file: ${inputs.media_file}
      source_language: ${inputs.source_language}

  - id: translate
    name: Translate Segments
    type: capability
    capability: translate
    model: ${settings.translate_model}
    input: ${transcribe.outputs.subtitles}
    params:
      source_language: ${inputs.source_language}
      target_language: ${inputs.target_language}

  - id: format_translated
    name: Format Translated Subtitles
    type: formatter
    format: ${settings.default_format}
    input: ${translate.output}

outputs:
  original:
    type: subtitle_editor
    source: ${transcribe.outputs.subtitles}
    label: Original Subtitles
    formats: [srt, vtt, txt]
  translated:
    type: subtitle_editor
    source: ${format_translated.output}
    label: Translated Subtitles
    formats: [srt, vtt, txt]
    primary: true
    words_from: translated_segments
  translated_segments:
    type: file
    source: ${translate.output}
    formats: [json]
    label: Translated Segments (JSON)
```

Annotated:
- **`transcribe` is `type: workflow`, not a copied set of steps.** It
  reuses the standalone `transcribe.yaml` (its own `ffmpeg` →
  `capability: stt` → `formatter` chain) so "transcription-only" and
  "transcribe + translate" run the exact same transcription — one
  implementation, two entry points.
- **`translate.input` reads `${transcribe.outputs.subtitles}`**, not
  `${transcribe.output}` — because the parent-side `transcribe` step
  here is `type: workflow`, its `.output` IS its child's whole outputs
  dict, and `.outputs.subtitles` is how the parent reaches one named
  member of it (see "Variable resolution" above).
- **`translate.model` is `${settings.translate_model}`**, resolved
  through hutash-subs's registered `resolve_translate_model` — which,
  for THIS run, ignores whatever is literally stored in Settings and
  instead picks a model supporting `target_language`, unless the user
  explicitly turned on "Choose Translation Model."
- **Three outputs, one `primary`.** `translated` is what an editor opens
  and an export defaults to; `original` is kept for reference, never
  defaulted to, because handing back the untranslated transcript would
  read as "translation didn't happen" even though it did.
- **`translated.words_from: translated_segments`** — SRT (what
  `format_translated` rendered `translated` to) can't carry a per-word
  time, so the animated caption styles need the pre-formatting
  `translate` step's own output, which still has them (proportionally
  re-derived across each cue, since a translation model returns words
  with no timings of their own — see `capability_step.proportional_words`).
- **No `file_read`/`file_write`/`ffmpeg` step appears in THIS file** —
  they live one level down, inside the `transcribe` sub-workflow it
  composes. A `full-dub`-style workflow (hutash-dub) is the shape that
  uses `ffmpeg`'s `replace_audio` directly, at its own top level, for
  its final mux step.

---

## 6. Application `manifest.yaml` — field-by-field schema

**There is only one `manifest.yaml` parser in the engine.**
`daemon/ext/packages/reader.go`'s `applyManifest()` → `PackageSpec`
(`spec.go`) is the SAME function documented field-by-field in
`pipeline-format.md` §1 — `type: application` and
`type: model_pipeline` are read by identical code. What differs between
the two package types is which fields an application actually
populates, and that installing an application additionally goes through
a SEPARATE Go subsystem afterward.

### Two install paths, two Go packages

| | Model pipeline | Application |
|---|---|---|
| Install call | `POST /packages/{id}/install` | `POST /apps/install` |
| Go package | `ext/packages` (`PackageSpec`, `reader.go`) | `pavaka` (`Manifest`, `manifest.go`/`request.go`) |
| What it reads | `manifest.yaml` directly, as the whole identity source | `manifest.yaml` merged with a SECOND file (below) into one flat spec before Pavaka ever sees it |

An application's `manifest.yaml` is read at least twice in practice:
once, minimally, by whatever needs its declared `type:` before an
engine call is even made (Developer Mode's own
`dev_packages.declared_type`/`parse_local_application` read it directly
with Python's `yaml.safe_load`), and again — merged with
`application/config/app.yaml` — as the flat JSON body
`pavaka.ParseInstallRequest` actually installs from. The shared Go
`applyManifest()`/`PackageSpec` reader (§ below) is written generically
enough to parse either package type's `manifest.yaml`, but this session
did not confirm it is what an INSTALLED application is tracked by
afterward — this machine's own engine storage root keeps app installs
under a separate `environments/` directory (Pavaka's own `<envDir>/
application` convention) alongside, not inside, the pipeline-oriented
`packages/` vault. Treat `ext/packages.PackageSpec` and `pavaka.Manifest`
as two independent readings of the same file for two different
purposes, not one canonical parse feeding two subsystems.

### `manifest.yaml` — the shared parser's fields, applied to `type: application`

(Same parser `pipeline-format.md` §1 documents in
full; this table calls out what changes for an application.)

| Field | Application usage |
|---|---|
| `id` | **Required** — the only field whose absence fails parsing outright. |
| `name`, `version` | Same as a pipeline. |
| `type: application` (or `app`) | Routes hutash-os's own dispatch to the app-install path rather than a pipeline install — `dev_packages._APPLICATION_TYPES = {"application", "app"}`. |
| `runtime`, `gpu`, `min_vram_gb`, `recommended_vram_gb`, `min_ram_gb`, `required_cpu_features` | Parsed identically, but meaningless for an app that never loads a model into its own process — every real shipped vertical (subs/podcast/dub) leaves these unset. **`runtime` here is NOT the field Pavaka's install request also calls `runtime`** — that one maps to `Manifest.Backend` (`"venv"`/`"container"`) and comes from the SEPARATE merged install spec below, not from this key. Two same-named, differently-scoped fields. |
| `capabilities` (`[{id, modality}]`) | **Not what an application declares its needs with.** This is the pipeline-side "what do I PROVIDE" list; a real application manifest (`hutash-subs/manifest.yaml`) leaves it empty — see `capabilities_needed` below. |
| `description` | Falls back to the Setup gate's Welcome-screen sentence when `vertical.yaml`'s own `app.description` is absent (REGISTRY.md's `setup_guard`). |
| `license` | Bare SPDX string — hutash-subs/podcast/dub all declare `MIT` here with no license TEXT anywhere in the package (a separately audited finding — see this workspace's `AGENTS.md`, Architecture Protection). |
| `min_os_version` | Same as a pipeline — relayed, never compared, by design. |
| `source` (`{repo, commit}`) | An application's actual code pin, for a git-sourced app. **Developer Mode always overwrites this** to `{type: local, path: <picked folder>}` regardless of what the file declares (`dev_packages.parse_local_application`) — loading a local folder always means "run THIS folder," never whatever the manifest's own `source:` says. |
| `ui` (opaque map) | **Not the pipeline's UI contract.** `package-types.md` calls a `ui:`/`capabilities:` block "Forbidden" for an application — but the real, currently-shipped `hutash-subs/manifest.yaml` declares `ui: {display_name, icon, category}` anyway. The shared parser stores it into `PackageSpec.UI` regardless of type (nothing rejects it), and this session found no code path on the hutash-os side that reads those three keys back out for an app — an open question (consumed somewhere, or vestigial) rather than one resolved here. |

**`capabilities_needed` — a completely different mechanism, same file,
unrelated to the Go parser above.** This is what a real application
manifest actually uses to declare what it needs
(`hutash-subs/manifest.yaml`'s own
`capabilities_needed: [stt, translate, llm]`). It is read ONLY by
hutash-os's own Python, `get_declared_capabilities()`
(`packages/hutash_workflow/server/services/engine_client.py:437-450`) —
`yaml.safe_load`ing the manifest file directly and pulling the flat
`capabilities_needed` list straight out, entirely independent of the Go
engine's `applyManifest`/`PackageSpec.Capabilities` field (a
differently-shaped `[{id, modality}]` list meant for a PIPELINE's
declared capabilities). Two keys, two shapes, two separate parsers (one
Go, one Python), one file — this is what `setup_guard`, the Models
page's grouping, and `vertical.yaml`'s `setup.required_capabilities` all
actually key off of; the Go-side `capabilities` field plays no part in
it for an application.

### `application/config/app.yaml` (or legacy `application/config.yaml`) — application-only

Merged on top of `manifest.yaml` (`dev_packages.parse_local_application`,
mirroring `catalogue_reader._merge_package_zip`'s precedence for a
zipped install) into the flat JSON body `pavaka.ParseInstallRequest`
(`daemon/pavaka/request.go`) reads for `POST /apps/install`:

| Field | Required? | Meaning |
|---|---|---|
| `entrypoint` | not enforced by the parser | The command Pavaka launches. Nothing at parse time rejects its absence — an app installs successfully and then fails at first launch with no command to run, rather than failing the install. |
| `workdir` | no | Working directory for the launched process. |
| `health` | no | A bare string (`/health`) OR `{endpoint: "/health"}` — both forms are tolerated and normalized to the plain string internally. |
| `port` | no | `0` means "engine assigns from its pool" (the real hutash-subs value) — int, string, or a list of ints all parse; a `.hutash` catalogue package may also use `{internal: N}` (confirmed against `hutash-studio.hutash`), normalized the same way. |
| `python` (or nested `environment.python`) | no | The interpreter version; a top-level `python` wins if both are present. |
| `runtime` | no, defaults `"venv"` | `"venv"` \| `"container"` — becomes `Manifest.Backend`. |
| `env` | no | Map of strings, passed to the launched process. |
| `managed` | no | `true` only for a first-party engine backend (Studio, a vertical) — never for a community app. |
| `packages` | no | A LIST of package entries (each may carry `variant`/`index_url`/`no_deps`/`force_reinstall`), resolved against host hardware at install time. **A different shape from `application/packages.yaml`'s `common`/`variants`/`system_packages` map that a PIPELINE's own `packages.yaml` uses** — same filename, two incompatible schemas, read by two different Go packages. |
| `requirements` / `extra_requirements` | no | `requirements.txt` filename(s), relative to the cloned/copied source. Real hutash-subs value: `requirements.txt`. |
| `frozen` | no | `true` layers `packages`/`requirements` as a locked `--constraint` rather than installing them directly. |
| `preserve` | no | Paths carried forward across a reinstall of the same app id (e.g. ComfyUI's `custom_nodes`). |
| `build` | no | A shell command run after dependencies install, before the app is considered ready (Studio's frontend bundle step). |
| `source` | conditionally required | `{type: git\|local\|pypi\|self-bootstrap\|url, url, ref, commit, path, venv_dir_env_var, bootstrap_cmd, shell, inference, manifest, assets}` — **`commit` is required, and parsing fails without it, when `type` is `git` or `self-bootstrap`** — a `ref` alone (a branch/tag) is rejected, since two installs of "the same" entry could otherwise resolve to different upstream commits. |

Real, complete, shipped example — hutash-subs's own three files, in full:

```yaml
# manifest.yaml (repo root)
id: hutash-subs
name: Hutash Subs (Beta)
version: 1.0.0
type: application
description: Transcribe and translate video subtitles locally using AI models
author: Hutash          # not read by the Go parser at all — decorative only
license: MIT

capabilities_needed:
  - stt
  - translate
  - llm                  # Hanu's Models-page section, marked optional in vertical.yaml's setup:

ui:
  display_name: Hutash Subs
  icon: resources/icon.svg
  category: Video Tools
```

```yaml
# application/config.yaml
port: 0                    # 0 = engine assigns from pool (49200-65535)
health: /health
managed: true
entrypoint: api/main.py
```

```yaml
# application/packages.yaml (application shape — NOT the pipeline shape)
requirements: requirements.txt
python: "3.12"
```

`author` in the real manifest above is not one of `applyManifest`'s
fields, and Pavaka's `installRequest` doesn't read it either — present
in the shipped file, read by nothing this session found. Not harmful,
just worth knowing before assuming every key in a real file is
load-bearing.

---

## 7. Friction points found during this audit

0. **`hutash-registry`'s published `hutash-subs.hutash` is stale
   relative to the `hutash-subs` repo's actual current source** — not a
   documentation problem, a real distribution gap. The published
   package's `vertical.yaml` (367 lines) is missing the entire `hanu`
   page, has no `llm_model` setting, and uses an older editor-page
   layout (separate `export`/`styles` pages) instead of the current
   `tabbed_panel`-based one (440 lines, quoted in full in §3 above).
   Confirmed by direct extraction and diff of both files at
   documentation time (2026-09-12). Whatever real users installing
   Hutash Subs from `hutash-registry` today are running, it does not
   have the Hanu assistant feature that exists in this repo's current
   source and current widget registry. This should be looked at as a
   release/publishing gap, separately from this documentation task.

1. **`REGISTRY.md`'s own widget count was stale** — its "Creating new
   widgets" section said "Twenty-three widgets exist" against an actual
   count (verified against `src/widgets/builtins.ts`'s `registerWidgets`
   call) of **29**. Fixed directly in `REGISTRY.md` in this pass; noted
   here so a future audit knows this one is closed rather than
   re-discovering it.

2. **No schema validation exists for `vertical.yaml`** (see §1's own
   note). A future package-authoring agent generating a malformed file
   gets silence, not an error — every downstream default just quietly
   activates. Contrast with the workflow-YAML loader, which validates
   strictly. This asymmetry is worth fixing before an agent is trusted
   to generate `vertical.yaml` unsupervised.

3. **Three unrelated "layout" vocabularies share loose terminology in
   this workspace**, and nothing enforces which applies where:
   - `vertical.yaml` page `layout:` — closed set `single` |
     `two-panel` | `three-panel` | `editor` (types.ts:93)
   - The legacy top-level `layout:` block's own `type:` — a
     differently-scoped `three-panel`|`two-panel`|`single` (types.ts:37)
   - `aggregator-3panel` — a Studio **model-pipeline** manifest.yaml
     layout hint (e.g. opus-mt-en-hi's `manifest.yaml`), for Studio's
     own model-detail page — **has nothing to do with `vertical.yaml`
     at all**, despite the naming similarity.

   An authoring agent that has seen `aggregator-3panel` in a pipeline
   manifest and reaches for it in a `vertical.yaml` page's `layout:`
   will silently get `single` instead (per the "anything else falls
   back to single" rule) with no error explaining why.

4. **`hutash-app`'s `CONTRIBUTING.md` (step 5) references
   `docs/HUTASH_FORMAT.md`** — that path does not exist in that repo.
   The real, actively-maintained file is `hutash-os`'s
   `docs/reference/HUTASH_FORMAT.md`. A stale cross-repo pointer.

5. **The widget/design reference itself is split across two repos in a
   way that isn't obvious from either one**: the widget *code* and
   `REGISTRY.md` live in `hutash-os` (`packages/hutash_vertical_ui/`),
   but the design-token/layout-system-level spec
   (`VERTICAL-UI-DESIGN-SPEC.md`) lives in `hutash-subs`, not
   `hutash-os` — a vertical repo, chosen presumably because it was
   authored while building that vertical, not because it's
   conceptually scoped to it. Anyone looking for "the" UI spec starting
   from `hutash-os` (where the actual implementation is) won't find it
   there.

6. **Neither `create-vertical.md` nor `REGISTRY.md` nor
   `VERTICAL-UI-DESIGN-SPEC.md` mentions `.hutashm`** — all three
   predate today's migration and describe every application/vertical
   package as `.hutash` (correctly, for that type) without noting that
   pipelines a vertical depends on are now `.hutashm`. Not a
   correctness bug in those docs (verticals genuinely stay `.hutash`),
   but a reader could reasonably wonder why a capability's own package
   isn't shown with the same extension anywhere in the vertical-side
   documentation.

7. **An application declares its needed capabilities through
   `capabilities_needed`, a key the Go engine's own manifest parser does
   not read at all.** `applyManifest` (`daemon/ext/packages/reader.go`)
   only ever populates `PackageSpec.Capabilities` from a `capabilities:`
   key shaped `[{id, modality}]` — the PIPELINE side's "what do I
   provide" list. An application's real declaration
   (`hutash-subs/manifest.yaml`'s `capabilities_needed: [stt, translate,
   llm]`, a flat string list) is read by a second, independent parser —
   hutash-os's own Python `get_declared_capabilities()`
   (`packages/hutash_workflow/server/services/engine_client.py:437-450`)
   — that `yaml.safe_load`s the same file directly and never touches the
   Go struct. An authoring agent that declares `capabilities` (the
   Go-documented key) on an application, expecting it to gate
   `setup_guard`, would produce a manifest that parses without error and
   simply never gates anything — see §6 above.

8. **`hutash-subs/manifest.yaml` declares a `ui:` block
   (`{display_name, icon, category}`) despite `package-types.md`
   calling `ui:`/`capabilities:` "Forbidden" for an application.** Nothing
   in the shared parser rejects it — `applyManifest` stores it into
   `PackageSpec.UI` unconditionally, regardless of `type:`. This session
   found no hutash-os code path reading those three keys back out for an
   app (unlike a pipeline's `ui.capabilities`, which Studio/a vertical
   genuinely interprets), so it's left as an open question — a real
   consumer this audit didn't find, or three keys nobody reads — rather
   than resolved either way. Worth a direct check before an authoring
   agent is told to omit or include this block on the strength of either
   doc alone.

9. **`application/packages.yaml` means two different, incompatible
   things depending on package type**, despite the identical filename
   and identical position in the folder layout. A pipeline's version is
   a map (`python`/`common`/`variants`/`system_packages`,
   `pipeline-format.md` §1); an application's version
   (merged into `POST /apps/install`'s body by
   `dev_packages.parse_local_application`) is `requirements`/
   `extra_requirements`/`packages` (a flat list, optionally of richer
   per-entry objects) read by an entirely different Go package
   (`pavaka`, not `ext/packages`). Copying one file's shape into the
   other package type produces a file that parses (both are permissive)
   but supplies none of the fields the actual installer for that type
   reads.

10. **`application/config/app.yaml`'s `entrypoint` is not enforced at
    parse time.** `pavaka.requiredFieldsMissing` checks `id`/`name`/
    `version` and, conditionally, `source.commit` — never `entrypoint`.
    An application missing it installs successfully and only fails when
    the engine actually tries to launch it, which is a much later and
    less legible failure than a rejected install would be.
