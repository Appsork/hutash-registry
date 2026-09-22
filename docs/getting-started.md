# Getting started — modifying `template-app/` into your own app

This is a **modification guide**, not a build-from-scratch tutorial.
`AGENTS.md`'s own first instruction is: copy `template-app/` into
`apps/<your-app-name>/`, then edit it. This doc explains what each file
in that template does and how to change it — walking through one real
modification (turning it into a translator) so you can see the pattern
before you apply it to your own idea.

If you haven't copied the template yet, do that first — everything below
assumes you're looking at your own copy, not the pristine original.

## What the template is

`template-app/` transcribes an audio file to text. One page: drop a file,
press a button, get a timed transcript back. It uses exactly one
capability (`stt`, already installed on a fresh machine via
whisper-tiny) — the smallest real thing that still exercises the whole
chain: an input, an action, a workflow, a capability call, and an output.
Confirmed live: it dev-loads and opens with no errors exactly as it
ships.

```
template-app/
├── manifest.yaml
├── resources/
│   └── icon.svg
└── application/
    ├── config.yaml
    ├── packages.yaml
    ├── requirements.txt
    ├── vertical.yaml
    ├── api/
    │   ├── __init__.py
    │   ├── _bootstrap.py
    │   └── main.py
    └── workflows/
        └── transcribe.yaml
```

## The files, one by one

### `manifest.yaml` — identity

```yaml
hutash_format: '1.0'
id: hutash-template
name: Hutash Template
version: 1.0.0
type: application
description: Transcribe an audio file to text — the minimal template every new app starts from
license: MIT

capabilities_needed:
  - stt

ui:
  display_name: Hutash Template
  icon: resources/icon.svg
  category: Utilities
```

**Change for your app:** `id` (lowercase, hyphenated, globally unique —
the only field whose absence breaks the install outright), `name`,
`description`, `ui.display_name`, `ui.category`. **`capabilities_needed`
is load-bearing** — it's the flat list of capability ids your app needs
installed to work, and it's what gates first launch (the app won't open
until at least one model for each listed capability is installed) and
what the generated Models page groups by. Add or remove entries here to
match whatever capabilities your workflows actually call — see
"Capability names" below before inventing one. Full field table:
`reference/application-format.md` §6.

### `application/vertical.yaml` — the whole frontend

```yaml
app:
  title: Hutash Template
  description: Transcribe an audio file to text

project:
  subfolders: [audio, transcripts]

pages:
  - id: home
    label: Home
    default: true
    layout: single
    widgets:
      - type: file_upload
        bind: audio_file
        props: {label: Drop an audio file here, accept: [audio/*]}
      - type: dropdown
        bind: source_language
        props: {label: Spoken language, options_from: "workflow:transcribe.inputs.source_language"}
      - type: action_button
        props:
          label: Transcribe
          action: run_workflow
          requires: [audio_file]
          workflow: transcribe
      - type: progress_steps
        props: {visible_during: workflow_run}
      - type: subtitle_list
        props:
          editable: true
          show_timestamps: true
          click_to_seek: false
          save_path: transcripts/transcript.srt
          working_path: transcripts/transcript.json

workflows:
  - {id: transcribe, file: workflows/transcribe.yaml, label: Transcribe, icon: mic, description: Turn an audio file into a timed transcript}

settings:
  stt_model:
    type: model_selector
    capability: stt
    label: Transcription Model
    description: Leave on Automatic to use the best installed transcription model

setup:
  required_capabilities:
    - capability: stt
      label: Transcription
      recommended_model: whisper-tiny
```

**Change for your app:** `app.title`/`app.description`, `project.subfolders`
(the folders a new project gets — match what your workflow actually
writes), the `pages:` list (swap `file_upload`/`dropdown`/`subtitle_list`
for whatever widgets your idea needs — full catalogue, every prop:
`reference/widgets.md`), the `workflows:` list (one entry per workflow
file you add under `application/workflows/`), and `settings:`/`setup:`
(one `model_selector` + one `required_capabilities` entry per capability
your workflows call — see the next section for why these two blocks must
stay in step). **Do not** hand-write a `models`, `settings`, or `hanu`
page — those are generated automatically from this file; see
`AGENTS.md` §4 ("What the framework provides automatically") for the
full list of what you never need to build yourself.

### `application/workflows/transcribe.yaml` — the pipeline

```yaml
id: transcribe
name: Transcribe
version: 1.0

inputs:
  audio_file:
    type: file
    accept: [audio/*]
    label: Audio file
    required: true
  source_language:
    type: dropdown
    options:
      - {value: auto, label: Auto Detect}
      - {value: en, label: English}
    default: auto
    label: Spoken Language
    required: false

steps:
  - id: transcribe
    name: Transcribe Audio
    type: capability
    capability: stt
    model: ${settings.stt_model}
    input: ${inputs.audio_file}
    params: {word_timestamps: true, language: "${inputs.source_language}"}

outputs:
  transcript:
    type: subtitle_editor
    source: ${transcribe.output}
    label: Transcript
    primary: true
    formats: [srt, vtt, txt]
    words_from: segments
  segments:
    type: file
    source: ${transcribe.output}
    formats: [json]
    label: Transcript Segments (JSON)
```

**Change for your app:** `inputs:` (what your page collects — must match
what `vertical.yaml`'s widgets `bind` to), `steps:` (one `type: capability`
step per model call — **never** `model: whisper-tiny`, always
`model: ${settings.<name>_model}`, resolved to whatever the user picked
or, left on Automatic, whatever's installed — see "Capability names"
below), and `outputs:` (what the page shows back). This is the one file
most of your actual idea lives in. Full step-type reference (`ffmpeg`,
`formatter`, `file_read`/`file_write`, a nested sub-`workflow`, beyond
just `capability`): `reference/application-format.md` §2.

### `application/config.yaml`, `application/packages.yaml`, `application/requirements.txt` — usually untouched

These three declare how the engine launches your app and what Python
packages it needs. Loaded through Developer Mode, `dev_stage()`
(`hutash-os/api/services/dev_packages.py`) overwrites `config.yaml`'s
`entrypoint`/`ports`/`health`/`managed` and tops up
`requirements.txt`'s floor dependencies automatically, so **you don't
need to touch any of these three files unless your app needs a package
beyond `create_app` itself** (add it to `requirements.txt` — never
remove the six already there) or a custom `env` var in `config.yaml`.

### `application/api/main.py` — usually untouched

```python
# application/api/main.py
from pathlib import Path

import api._bootstrap  # noqa: F401  — puts the OS packages on sys.path

from hutash_workflow.server import create_app
from hutash_workflow.server.config import VerticalSpec, configure
from hutash_workflow.server.workflow_source import declared_settings

_ROOT = Path(__file__).resolve().parent.parent
_VERTICAL_YAML = "vertical.yaml"

SPEC = VerticalSpec(
    app_id="hutash-template",
    title="Hutash Template",
    version="1.0.0",
    root=_ROOT,
    vertical_yaml=_VERTICAL_YAML,
    workflows_dir="workflows",
    projects_folder="Hutash Template",
    projects_env="HUTASH_TEMPLATE_PROJECTS_DIR",
    default_settings=declared_settings(_ROOT / _VERTICAL_YAML),
)
configure(SPEC)

app = create_app(spec=SPEC)
```

Two calls — `configure` and `create_app` — and nothing else, because
`create_app()` provides everything else your app needs (project
management, workflow execution, Settings, the Models page, Hanu, auth,
static hosting) without you declaring it. `default_settings=
declared_settings(...)` is why `${settings.stt_model}` in the workflow
above actually resolves: `declared_settings()` reads `vertical.yaml`'s
`settings:` block straight into the runtime values dict, so **every
setting you add to `vertical.yaml` is automatically resolvable — you
never edit this file just because you added a setting.**

**Change `app_id`/`title`/`projects_folder`/`projects_env`** to match
your app (keep `projects_env` unique — it's the environment variable a
user can set to relocate their projects folder). **Edit the rest of this
file only if you need a custom output format** (an ASS subtitle
document, a non-standard export) — `register_format`, imported from
`hutash_workflow.steps.formatter_step` specifically (not top-level
`hutash_workflow`), registers one; see `reference/patterns.md`'s "custom
output format" technique for the full pattern. Everything about *why*
`root` is computed this exact way (two `.parent` hops, not three) is in
this file's own comments — read them before changing it, since getting
it wrong doesn't fail cleanly (see that comment for what actually
breaks).

### `application/api/_bootstrap.py` — never touched

Puts `hutash_workflow`/`hutash_vertical_ui` on the import path. Copy it
verbatim into any new app — it needs no changes per app, and
`dev_stage()` writes it automatically if you delete it.

## Worked example: turning the template into a translator

Say you want a text translator instead: paste text in, pick a target
language, get translated text back. Same shape, different capability.

1. **`manifest.yaml`**: change `id: hutash-translate`, `name`,
   `description`; `capabilities_needed: [translation]` (HuggingFace's
   task name for this — see "Capability names" below).
2. **`application/vertical.yaml`**: swap the `file_upload` widget for a
   `text_input` (`bind: source_text`), add a `dropdown` for the target
   language (`bind: target_language`); change `settings:` from
   `stt_model` to `translation_model` (`capability: translation`); change
   `setup.required_capabilities` to match.
3. **`application/workflows/transcribe.yaml`** → rename to
   `translate.yaml`, update the `id`/`name`, change `inputs:` to
   `source_text`/`target_language`, change the step's `capability: stt`
   to `capability: translation`, `model: ${settings.translation_model}`,
   and its `params:` to whatever that capability's models expect
   (check an installed translation model's own manifest for its accepted
   params — `GET /packages/{id}/manifest`). Change `outputs:` from a
   `subtitle_editor` to a plain `type: file`/text result.
4. **`application/vertical.yaml`**'s `workflows:` list: point at
   `workflows/translate.yaml` with the new id.
5. **`application/api/main.py`**: only `app_id`/`title`/
   `projects_folder`/`projects_env` change — nothing else, since there's
   still no custom output format.

An image generator follows the identical pattern: `capability:
text-to-image`, a `text_input` for the prompt, an `image_display` output
widget instead of `subtitle_editor`. The shape — manifest declares the
capability, `vertical.yaml` collects input and shows output, the
workflow calls the capability by name, settings resolve which model —
never changes; only the capability name, the widgets, and the workflow's
`params:` do.

## Capability names — never a model, always a category

`capability: stt` above names a *category of model*, never a specific
one — that's the entire reason `type: capability` steps exist. A
workflow that hardcoded `model: whisper-tiny` would break the moment
that model isn't installed, and it would ignore every other
speech-to-text model a user has instead. Never reference a model by id
in a workflow; always reference the capability it provides, and let
`${settings.X_model}` (or an empty value, auto-picking whatever's
installed) resolve it to a real model at run time.

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
**Your app needs a capability that's NOT in the catalogue?** Changing
the application alone won't make it real — you also need a `.hutashm`
model pipeline that provides it before any workflow step naming that
capability can resolve to anything. See `reference/pipeline-format.md`.

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

## Where to go next

- **A bigger application** (multiple pages, projects, file uploads, an
  editor): `reference/application-format.md` §3, the hutash-subs worked
  example, annotated in full.
- **A model pipeline instead** (no UI, called by other apps):
  `reference/pipeline-format.md`.
- **A specific widget's full prop list**, beyond what the template uses:
  `reference/widgets.md`.
- **Every `vertical.yaml` key the template doesn't use** (multi-panel
  layouts, overlays with `confirm_leave`, persisted settings,
  `filter_by`/`describe_by` on a dropdown, ...): `reference/application-format.md` §1.

## Testing loop — build, load, call, verify, without the UI

Turning on Developer Mode in Hutash OS Settings and using its "Load
Application" picker on your app folder directly is the human way — no
zip, no publish step, and reloading after an edit just means picking the
folder again. An agent (or a script) drives the same load through
hutash-os's own backend (port 8780, **not** the engine's 47990) — four
endpoints, all under `/api/dev`:

| Action | Endpoint |
|---|---|
| Load a `.hutash` application folder | `POST http://localhost:8780/api/dev/packages/app` — body `{"path": "<absolute folder path>"}` |
| Load a `.hutashm` pipeline folder | `POST http://localhost:8780/api/dev/packages/model` — body `{"path": "<absolute folder path>"}` |
| List currently-loaded dev packages | `GET http://localhost:8780/api/dev/packages` — `[{id, name, type: "app"\|"model", source_path}]` |
| Remove a loaded dev package | `DELETE http://localhost:8780/api/dev/packages/{id}?type=app` (or `?type=model`) |

```bash
# Load your copy of the template
curl -X POST http://localhost:8780/api/dev/packages/app \
  -H "Content-Type: application/json" \
  -d '{"path": "E:/path/to/apps/my-app"}'

# List what's loaded
curl http://localhost:8780/api/dev/packages

# Remove it
curl -X DELETE "http://localhost:8780/api/dev/packages/my-app?type=app"
```

The build-test loop:

1. **Copy and edit the template** — the steps at the top of this doc.
2. **`GET http://localhost:47990/health`** — engine alive? Returns
   `{"status": "ok", "version": "..."}`. Nothing below works if this fails —
   start the engine first.
3. **Load it** — the `POST /api/dev/packages/app` call above, pointed at
   your app folder.
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
An MCP-compatible client (or you, testing) can call the same workflow
this way instead of the HTTP endpoint above.
