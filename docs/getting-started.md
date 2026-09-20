# Getting started — build a `.hutash` application

A step-by-step build of one real, minimal application: **Hutash Say**, a
one-page app that turns typed text into a spoken audio file. Every file is
shown in full — copy it, don't paraphrase it — and every field is listed
with its real options, not just the one value this example picks.

This is a tutorial, not a reference. It teaches the shape by building one
real thing. For the exhaustive field-by-field schema behind any step
below, follow the `reference/*.md` link next to it. Read `AGENTS.md` first
if you haven't — it explains what this file is for and what it isn't.

Works the same way for a human reading it and for an LLM following it: every
step names the exact file, shows its exact full contents, and says exactly
what changes if you want something different.

## What you're building

An app with one page. A text box, a voice picker, a button, and a player
for the result. Press the button, the engine's installed text-to-speech
model turns the text into a WAV file, and the app plays it back. No
projects, no multi-page navigation, no export formats — the smallest real
thing that exercises the whole chain: an input, an action, a workflow, a
capability call, and an output.

Once this works, `reference/application-format.md` §3 walks through
**hutash-subs** — a real, bigger, currently-shipped app (multi-page,
projects, file uploads, an editor) — annotated the same way, for when you
need more than this tutorial covers.

## Step 0 — decide the package type

Two package types exist: an **application** (`.hutash`) renders its own
UI; a **model pipeline** (`.hutashm`) has none and is called through the
engine's inference proxy. Hutash Say has a UI (a page, a button, a
player), so it's an application. If what you're building has no UI of its
own — it just takes input and returns output — stop here and read
`reference/pipeline-format.md` instead; the two package types share
almost nothing past the outer zip format. Full decision rule:
`reference/package-types.md`.

## Step 1 — the folder

```
hutash-say/
  manifest.yaml
  application/
    config.yaml
    packages.yaml
    vertical.yaml
    workflows/
      speak.yaml
```

Create `hutash-say/` and everything under it now; each step below writes
one of these files in full.

## Step 2 — `manifest.yaml`

The package's identity. Lives at the package root, one level above
`application/`.

```yaml
id: hutash-say
name: Hutash Say
version: 1.0.0
type: application
description: Turn text into speech, locally
author: Your Name
license: MIT

capabilities_needed:
  - text-to-speech
```

| Field | Required? | Your options |
|---|---|---|
| `id` | **yes** | Lowercase, hyphenated, globally unique among installed packages. This is the only field whose absence breaks the install outright. |
| `name` | yes | Display name — shown in the app store, the window title. |
| `version` | yes | Semver string, e.g. `1.0.0`. |
| `type` | yes | `application` (or `app`) for this kind of package. `model_pipeline` (or `model`/`models`) is the other type — see Step 0. |
| `description` | no | One sentence. Falls back to `vertical.yaml`'s own `app.description` if that's set instead. |
| `author` | no | Not read by anything — purely informational. |
| `license` | no | A bare SPDX id (`MIT`, `Apache-2.0`, ...). No license *text* is required by this field — that's a separate concern if you want one. |
| `capabilities_needed` | no, but load-bearing | A flat list of capability ids this app needs installed to work — `text-to-speech`, `automatic-speech-recognition`, `translation`, `text-generation`, ... (the exact names — see "Capability names" after Step 6). This is what gates first launch (the app won't open until at least one model for each listed capability is installed) and what the generated Models page groups by. **This is a different key from a model pipeline's `capabilities:` field** — don't confuse the two; see `reference/package-types.md` if you want the full explanation of why they're separate. |

Do not add a `ui:` or `capabilities:` block here — those belong to a
model pipeline's manifest, not an application's. Full field-by-field
table, including what's parsed but unused for an application:
`reference/application-format.md` §6.

## Step 3 — `application/config.yaml`

How the engine launches the app's backend process.

```yaml
health: /health
managed: true
entrypoint: python -m uvicorn api.main:app --host 127.0.0.1 --port 8080
ports:
  - internal: 8080
```

| Field | Required? | Your options |
|---|---|---|
| `entrypoint` | not enforced at parse time, but the app won't start without it | The FULL command the engine execs — never a bare script path. `hutashd` passes this string straight to `fork/exec` (`ext/appmanager`'s `parseEntrypoint`) with no interpreter of its own, so `entrypoint: api/main.py` fails immediately (`%1 is not a valid Win32 application` on Windows) — confirmed live, hutash-karaoke's first install. A first-party vertical's OWN repo looks like it gets away with the bare form because `hutash-os/scripts/stage-verticals.ps1` rewrites it into exactly this shape before that vertical is ever installed for real (see that script's own `application/config/app.yaml` generation). Developer Mode has no equivalent staging step — it installs your `application/config.yaml` verbatim — so write the full command yourself. `8080` above is an arbitrary placeholder; only `ports[].internal` matching the port literal in `entrypoint` matters, so the engine's `templatePort` can find and rewrite it to `{port}` at registration — the real port comes from the engine's pool at start time. |
| `ports` | needed whenever `entrypoint` embeds a port number | A list of `{internal: N}` — `N` must equal the port literal in `entrypoint` exactly (same digits), or `templatePort` has nothing to match and your app starts pinned to whatever placeholder you wrote instead of the engine's real assignment. |
| `health` | no | The health-check endpoint path. `/health` is what the shared framework serves automatically — leave it as-is unless you wrote a custom backend. |
| `managed` | no | `true` for a first-party-style app using the shared framework (this one). Leave `false`/unset only for a genuinely third-party, unmanaged process. |

`reference/application-format.md` §6 has the full field list (`workdir`,
`env`, `packages` as a richer list-of-objects shape, `source`, `frozen`,
`preserve`, `build`) for when your app needs more than these four.

## Step 4 — `application/packages.yaml`

Python dependencies.

```yaml
requirements: requirements.txt
python: "3.12"
```

| Field | Required? | Your options |
|---|---|---|
| `requirements` | no | Name of a `requirements.txt` your app ships (relative to `application/`). |
| `python` | no | Interpreter version string. `"3.12"` matches every real shipped vertical. |

This application-side `packages.yaml` is a **flat list/requirements
shape** — not the same schema as a model pipeline's own `packages.yaml`
(which is a `python`/`common`/`variants`/`system_packages` map). Same
filename, two different schemas depending on package type — see
`reference/package-types.md` if you want the full explanation.

Also write `application/requirements.txt`. **`hutash_workflow` (the
shared framework) is plain sys.path-imported source, not a pip-installed
package — it has no dependency metadata of its own and installs nothing
by itself.** Every app that calls `create_app()` must declare that
function's own runtime dependencies directly, the same list every real
shipped vertical repeats in its own `requirements.txt` (confirmed live,
hutash-karaoke's second install failure — `No module named uvicorn`,
after fixing the entrypoint above got the process to actually launch):

```
fastapi>=0.111.0
uvicorn>=0.30.0
httpx>=0.27.0
pyyaml>=6.0
pydantic>=2.7
python-multipart>=0.0.9
```

Leave it at exactly that unless your backend imports something beyond
`create_app` itself.

## Step 5 — `application/vertical.yaml`, one block at a time

This is the whole frontend. Build it up piece by piece; the full file is
assembled at the end of this step.

### 5a. `app:` and `project:`

```yaml
app:
  title: Hutash Say
  description: Turn text into speech, locally

project:
  subfolders: [audio]
```

`app.title`/`app.description` are the window/tab chrome. `project.subfolders`
adds folders your workflows write into, beyond the framework's own
default (`media`, `exports`) — this app writes generated audio into
`audio/`.

### 5b. The one page

```yaml
pages:
  - id: home
    label: Home
    default: true
    layout: single
    widgets:
      - type: section_header
        position: main
        props: {title: Say something}
      - type: text_input
        position: main
        bind: text
        props:
          label: Text to speak
          multiline: true
          placeholder: Type or paste what you want spoken
      - type: dropdown
        position: main
        bind: voice
        props:
          label: Voice
          options_from: api:/available-voices
      - type: action_button
        position: main
        props:
          label: Generate speech
          action: run_workflow
          requires: [text]
          workflow: speak
      - type: progress_steps
        position: main
        props: {visible_during: workflow_run}
      - type: output_player
        position: main
        props: {output: speech, label: Result}
```

| Page field | Your options |
|---|---|
| `layout` | `single` (used here — one column), `two-panel`, `three-panel`, `editor`. Anything else silently falls back to `single` — no error. Full layout rules: `reference/application-format.md` §1. |
| `default` | `true` marks the landing page. Exactly one page should set this. |

| Widget used | What it's for | Some other widgets you could reach for instead |
|---|---|---|
| `section_header` | A heading, optionally with an action button | — |
| `text_input` | The text box. `bind: text` means its value fills the workflow input named `text` | `number_input`, `toggle`, `slider`, `file_upload` for other input shapes |
| `dropdown` | The voice picker, populated live from the engine (`options_from: api:/available-voices`) rather than a hardcoded list | `model_selector` (picks an installed model directly, rather than an option the model serves) |
| `action_button` | Runs the `speak` workflow, and stays disabled until `text` (its one `requires:` entry) is filled | — |
| `progress_steps` | Shows step-by-step run progress while the workflow executes | `status_badge` for a simpler single-state indicator |
| `output_player` | Plays the file the run produced (`speech`, matching the workflow's own output id — Step 6) | `audio_player` if you were playing an uploaded file instead of a run's output |

**`bind:` is the entire data-binding mechanism.** A widget's `bind: text`
means its value lands under the key `text` in the workflow's `inputs` —
there is no separate mapping layer. The workflow you write in Step 6
must declare an input with that exact id.

Every widget type that exists, with every prop it reads: `reference/widgets.md`.

### 5c. The Hanu block — copy this verbatim

```yaml
  - id: hanu
    label: Hanu
    overlay: true
    overlay_size: md
    layout: single
    widgets:
      - type: hanu_panel
        position: main
```

This isn't part of Hutash Say's own design — it's required boilerplate.
The assistant's *backend* is automatic for every app using the shared
framework, but the page that hosts it in the UI is not generated yet, so
every real shipped vertical declares exactly this block, unchanged. Add
it as a second entry in `pages:`, right after `home`.

### 5d. `workflows:`, `settings:`, `setup:`

```yaml
workflows:
  - {id: speak, file: workflows/speak.yaml, label: Speak, icon: mic, description: Convert text to speech}

settings:
  tts_model:
    type: model_selector
    capability: text-to-speech
    label: Voice Model
    description: Leave on Automatic to use the best installed model

setup:
  required_capabilities:
    - capability: text-to-speech
      label: Text to Speech
      recommended_model: kokoro
```

- `workflows:` is a lookup table from an id (what `action_button`'s
  `workflow:` field names) to the YAML file that defines it — written in
  Step 6.
- `settings:` adds a "Voice Model" picker to the generated Settings page.
  Its `type` could also be `dropdown`, `toggle`, or `folder_picker` — see
  `reference/widgets.md`'s `settings_page` entry for all four and what
  each is for. Left on "Automatic", the workflow resolves to whatever TTS
  model is installed — see the next paragraph.
- `setup:` only *labels* a capability `manifest.yaml`'s
  `capabilities_needed` already declared — it cannot add a new
  requirement, only describe one. `recommended_model` is offered first on
  the first-launch install screen; omit it and the smallest compatible
  model is offered instead. Mark an entry `optional: true` for a
  capability your app can run without.

### The complete file

```yaml
app:
  title: Hutash Say
  description: Turn text into speech, locally

project:
  subfolders: [audio]

pages:
  - id: home
    label: Home
    default: true
    layout: single
    widgets:
      - type: section_header
        position: main
        props: {title: Say something}
      - type: text_input
        position: main
        bind: text
        props:
          label: Text to speak
          multiline: true
          placeholder: Type or paste what you want spoken
      - type: dropdown
        position: main
        bind: voice
        props:
          label: Voice
          options_from: api:/available-voices
      - type: action_button
        position: main
        props:
          label: Generate speech
          action: run_workflow
          requires: [text]
          workflow: speak
      - type: progress_steps
        position: main
        props: {visible_during: workflow_run}
      - type: output_player
        position: main
        props: {output: speech, label: Result}

  - id: hanu
    label: Hanu
    overlay: true
    overlay_size: md
    layout: single
    widgets:
      - type: hanu_panel
        position: main

workflows:
  - {id: speak, file: workflows/speak.yaml, label: Speak, icon: mic, description: Convert text to speech}

settings:
  tts_model:
    type: model_selector
    capability: text-to-speech
    label: Voice Model
    description: Leave on Automatic to use the best installed model

setup:
  required_capabilities:
    - capability: text-to-speech
      label: Text to Speech
      recommended_model: kokoro
```

### 5f. `application/api/main.py` and `application/api/_bootstrap.py`

Every other file in this tutorial is declarative — `vertical.yaml`
describes the UI, the workflow YAML describes the pipeline, and the
shared framework reads both. This is the one file with real code in it,
and it is short: two calls, `configure` and `create_app`.

`create_app` needs a real `VerticalSpec` object — not a plain dict. It
reads `spec.root` directly, so `{"id": ..., "workflows_dir": ...}` fails
at import time with `AttributeError: 'dict' object has no attribute
'root'` (confirmed live, hutash-karaoke's third crash this session).
`root` is this app's OWN directory — the one containing `application/`
and `manifest.yaml` — so it's three `.parent` hops up from
`application/api/main.py`:

```python
# application/api/main.py
from pathlib import Path

import api._bootstrap  # noqa: F401  — puts the OS packages on sys.path

from hutash_workflow.server import create_app
from hutash_workflow.server.config import VerticalSpec, configure

SPEC = VerticalSpec(
    app_id="hutash-say",
    title="Hutash Say",
    version="1.0.0",
    root=Path(__file__).resolve().parent.parent.parent,
    projects_folder="Hutash Say",
    projects_env="HUTASH_SAY_PROJECTS_DIR",
)
configure(SPEC)

app = create_app(spec=SPEC)
```

`import api._bootstrap` must come BEFORE `from hutash_workflow import
...` — `hutash_workflow`/`hutash_vertical_ui` are plain sys.path-imported
source from your hutash-os checkout, never pip-installed, so nothing puts
them on the import path automatically. Without it: `ModuleNotFoundError:
No module named 'hutash_workflow'` (karaoke's second crash). Copy this
file verbatim into `application/api/_bootstrap.py` — it needs no changes
per app:

```python
# application/api/_bootstrap.py
"""Puts the OS-provided shared packages on the import path."""
from __future__ import annotations

import os
import sys
from pathlib import Path

_SIBLING_LAYOUT = Path("..") / ".." / "hutash-os-files" / "hutash-os"


def os_packages_dir() -> Path | None:
    declared = os.environ.get("HUTASH_OS_PATH", "").strip()
    candidates = []
    if declared:
        candidates.append(Path(declared) / "packages")
    here = Path(__file__).resolve().parent.parent
    candidates.append((here / _SIBLING_LAYOUT / "packages").resolve())
    for candidate in candidates:
        if (candidate / "hutash_workflow").is_dir():
            return candidate
    return None


def install() -> Path:
    found = os_packages_dir()
    if found is None:
        raise RuntimeError(
            "the shared Hutash packages could not be found. Set HUTASH_OS_PATH "
            "to your hutash-os checkout (the directory containing packages/)."
        )
    path = str(found)
    if path not in sys.path:
        sys.path.insert(0, path)
    return found


install()
```

`_SIBLING_LAYOUT`'s guess only works when your app repo sits in the
standard workspace layout, sibling to `hutash-os-files/`. A Developer
Mode app living anywhere else (e.g. a scratch folder under
`developer/apps/`) needs `HUTASH_OS_PATH` declared explicitly — set it in
`application/config.yaml`'s `env` map (Step 3 above), not `env_vars`; see
that step's own field table for why the key name matters.

`register_format`, if your app needs a custom output format (an ASS
subtitle document, say), is NOT exported from top-level
`hutash_workflow` — import it from `hutash_workflow.steps.formatter_step`
specifically. See `reference/patterns.md`'s "custom output format"
technique for the full pattern.

## Step 6 — `application/workflows/speak.yaml`

The one workflow: take the typed text, call an installed `text-to-speech` model,
hand back the audio file.

```yaml
id: speak
name: Speak
version: 1.0

inputs:
  text:
    type: text
    label: Text to speak
    required: true
  voice:
    type: dropdown
    label: Voice
    required: false

steps:
  - id: generate
    name: Generate Speech
    type: capability
    capability: text-to-speech
    model: ${settings.tts_model}
    input: ${inputs.text}
    params:
      voice: ${inputs.voice}

outputs:
  speech:
    type: file
    source: ${generate.output}
    label: Generated Speech
    primary: true
```

| Piece | What's happening |
|---|---|
| `inputs.text` / `inputs.voice` | Must match `vertical.yaml`'s `bind:` ids exactly (Step 5b) — this is the other half of the same contract. `required: true` on `text` means a run with nothing typed never reaches the model. |
| `steps[0].type: capability` | The one step type that calls an installed model, through the engine's inference proxy. Five other step types exist for file conversion, format conversion, and composing workflows — full list and every field: `reference/application-format.md` §5. |
| `model: ${settings.tts_model}` | Resolves to whatever the Settings page's "Voice Model" picker holds (Step 5d). Left on Automatic (empty), the step falls back to the first installed model that declares `text-to-speech` — never a hard failure as long as *something* is installed. |
| `params.voice` | Passed to the model alongside the text. The exact param name a model expects is declared in *that model's own manifest* (`kokoro`'s, if that's what's installed) — `voice` is what this tutorial assumes; if you picked a different TTS model, check its manifest before assuming the name matches. Full resolution mechanics: `reference/application-format.md` §2. |
| `outputs.speech` | `type: file` because the model returns audio bytes, not text. `primary: true` marks it as the run's one deliverable — matters once a workflow has more than one output. `source: ${generate.output}` reads the `generate` step's raw result. |

Every `${...}` reference here is checked when the app loads: a typo'd
input name, or a reference to a step that hasn't run yet, fails loudly
at load time — before anyone presses the button. Full `${...}`
resolution rules and every step type: `reference/application-format.md` §5.

## Capability names — never a model, always a category

`capability: text-to-speech` above names a *category of model*, never a
specific one — that's the entire reason `type: capability` steps exist. A
workflow that hardcoded `model: kokoro` would break the moment kokoro isn't
installed, and it would ignore every other text-to-speech model a user has
instead. Never reference a model by id in a workflow; always reference the
capability it provides, and let `${settings.X_model}` (or an empty value,
auto-picking whatever's installed) resolve it to a real model at run time.

Capability names follow **HuggingFace's own task taxonomy exactly** —
hyphens, not underscores
(https://huggingface.co/docs/transformers/main_classes/pipelines).

**Before building, check what capabilities actually exist right now —
don't assume:**

```bash
curl http://localhost:47990/catalogue
```

This returns every package — installed or not — with the capabilities
each one declares. Your app needs a capability already in there? Use it
directly, exactly as spelled in the response — no new package required.
**Your app needs a capability that's NOT in the catalogue?** Building the
application alone won't make it real — you also need a `.hutashm` model
pipeline that provides it before any workflow step naming that capability
can resolve to anything. See `reference/pipeline-format.md`.

Adding a model for a task the catalogue has nothing for? **Check
HuggingFace's task list first.** If a matching task exists, use its exact
name. Only invent a new capability name when HuggingFace genuinely has no
equivalent — these have come up so far:

| Capability | What it does |
|---|---|
| `audio-source-separation` | vocal/stem separation |
| `object-detection` | detect objects in images |
| `image-segmentation` | pixel-level image masks |
| `image-to-image` | image transformation |
| `summarization` | text condensation |
| `depth-estimation` | image to depth map |

**The engine accepts any string as a capability — there is no hardcoded
list, and nothing validates a name against HuggingFace's taxonomy or
against the registry's `index.json`** (verified directly against the Go
engine's source: capability matching is a bare string-equality check
against whatever each installed package's own manifest declares; the
`index.json` `features`/`modalities` blocks are read as opaque data for
website/Studio display only, never for validation). A capability is real
the moment any installed package's manifest says so. The vocabulary above
is real only because everyone adding a model agrees to use it — enforced
by PR review, not by code, so get it right in review; the engine will not
catch a typo'd or invented capability name for you.

## Step 7 — package it

```
hutash-say.hutash (zip)/
├── manifest.yaml
└── application/
    ├── config.yaml
    ├── packages.yaml
    ├── vertical.yaml
    ├── requirements.txt
    └── workflows/
        └── speak.yaml
```

Zip the `hutash-say/` folder's contents (not the folder itself) into
`hutash-say.hutash`. No `resources/` folder, no `launch.yaml` — both are
model-pipeline-only; an application's launch info lives in
`application/config.yaml`, already written in Step 3. Full container
format (why it's a plain zip, what the three layers are, what's forbidden
for each package type): `reference/hutash-format.md`.

## Step 8 — try it

The fastest loop while building: turn on Developer Mode in Hutash OS
Settings, then use its "Load Application" picker on the unzipped
`hutash-say/` folder directly — no zip, no publish step, and reloading
after an edit just means picking the folder again. Once it opens, the
first-launch screen should ask to install a `text-to-speech` model (Step 5d's
`setup:` block) before showing the page built in Step 5.

## Where to go next

- **A bigger application** (multiple pages, projects, file uploads, an
  editor): `reference/application-format.md` §3, the hutash-subs worked
  example, annotated in full.
- **A model pipeline instead** (no UI, called by other apps):
  `reference/pipeline-format.md`, starting from Step 0 above.
- **A specific widget's full prop list**, beyond what Step 5b's table
  covers: `reference/widgets.md`.
- **Every `vertical.yaml` key this tutorial didn't use** (multi-panel
  layouts, overlays with `confirm_leave`, persisted settings,
  `filter_by`/`describe_by` on a dropdown, ...): `reference/application-format.md` §1.

## Testing loop — build, load, call, verify, without the UI

Step 8's "Load Application" picker is the human way. An agent (or a
script) drives the same load through hutash-os's own backend (port 8780,
**not** the engine's 47990) — four endpoints, all under `/api/dev`:

| Action | Endpoint |
|---|---|
| Load a `.hutash` application folder | `POST http://localhost:8780/api/dev/packages/app` — body `{"path": "<absolute folder path>"}` |
| Load a `.hutashm` pipeline folder | `POST http://localhost:8780/api/dev/packages/model` — body `{"path": "<absolute folder path>"}` |
| List currently-loaded dev packages | `GET http://localhost:8780/api/dev/packages` — `[{id, name, type: "app"\|"model", source_path}]` |
| Remove a loaded dev package | `DELETE http://localhost:8780/api/dev/packages/{id}?type=app` (or `?type=model`) |

```bash
# Load hutash-say
curl -X POST http://localhost:8780/api/dev/packages/app \
  -H "Content-Type: application/json" \
  -d '{"path": "E:/path/to/hutash-say"}'

# List what's loaded
curl http://localhost:8780/api/dev/packages

# Remove it
curl -X DELETE "http://localhost:8780/api/dev/packages/hutash-say?type=app"
```

The build-test loop:

1. **Build the package files** — everything through Step 7 above.
2. **`GET http://localhost:47990/health`** — engine alive? Returns
   `{"status": "ok", "version": "..."}`. Nothing below works if this fails —
   start the engine first.
3. **Load it** — one of the two `POST /api/dev/packages/...` calls above,
   pointed at your unzipped folder.
4. **`GET http://localhost:47990/packages`** (needs
   `Authorization: Bearer <token>` — read `api_token` from `hutashd.json`,
   never hardcode it in a script) — confirm your package id is listed.
5. **For a `.hutashm` pipeline:** call its capability directly —
   `POST http://localhost:47990/packages/{id}/infer/{endpoint}`. The
   endpoint comes from that model's own manifest
   (`GET http://localhost:47990/packages/{id}/manifest`) — never assume it
   matches the capability name (see `reference/application-format.md` §2).
   Verify the response.
6. **For a `.hutash` application:** find its assigned port first —
   `GET http://localhost:47990/apps` lists every installed app with its
   port — then `POST http://localhost:<app_port>/api/v1/workflows/{workflow_id}/run`
   with your inputs (multipart for a file input, JSON otherwise). Verify
   the streamed `result` event.
7. **If it fails:** fix the files, reload (loading again over the same id
   updates it — no need to remove first), retest. This loop is the entire
   point of Developer Mode: no packaging, no publish step between edits.

**MCP note:** the moment a package loads, every workflow it declares is
also live as an MCP tool automatically — no separate registration step.
An MCP-compatible client (or you, testing) can call the same workflow that
way instead of a raw `run_workflow` POST.
