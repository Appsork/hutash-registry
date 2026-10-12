# Hutash Developer Guide

This is the one guide to building for Hutash. It covers what you can build, how to build and test
it with Developer Mode, and how a package reaches users. For the exact field-by-field specs it
points to the reference documents rather than repeating them:

- [Pipeline format](reference/pipeline-format.md) - every file in a model pipeline, field by field
- [Capability interfaces](reference/capability-interfaces.md) - the ten capabilities and their parameter names
- [Application format](reference/application-format.md) - every file in an application

## Overview

You can build two kinds of package:

| Package | File | What it is |
|---|---|---|
| **Model pipeline** | `.hutashm` | A model the engine installs, runs and exposes as one or more *capabilities* (`stt`, `tts`, `image`, ...). Any app, Studio, the CLI or MCP calls it the same way. |
| **Application** (a "vertical") | `.hutash` | A UI that consumes capabilities: pages and widgets written in YAML, plus workflows that call models. |

A package is a zip with three layers: **identity** (`manifest.yaml`), **runtime** (`application/`)
and **resources** (`resources/`, for example the weights to download).

Developer Mode lets you build and test either kind without publishing anything. It is driven over
MCP: an agent or a script calls a small set of `hutash_dev_*` tools that write your package into a
workspace folder, load it into the engine through the same install path a published package takes,
call it, and remove it again. The tools are listed under [Developer Mode](#developer-mode).

## Creating a Pipeline

A pipeline is five files:

```
my-pipeline/
├── manifest.yaml              identity, hardware needs, and the UI contract
├── application/
│   ├── inference.py           load() plus one method per capability
│   ├── launch.yaml            how the engine starts it
│   └── packages.yaml          the Python packages it needs
└── resources/
    └── weights.yaml           which Hugging Face files to download
```

| File | Role |
|---|---|
| `manifest.yaml` | `id`, `name`, `version`, `type: model_pipeline`, `gpu`, `min_vram_gb`, `min_ram_gb`, `license`; a top-level `capabilities: [{id, modality}]` list; `metadata.quality_score`; and `ui.capabilities`, a **map** keyed by capability id that declares each capability's `inputs`, `controls` and `outputs`. |
| `application/inference.py` | A subclass of `hutash_inference.Inference`. `load()` loads the model once; each `@capability("<id>")` method serves one capability. Its parameter names must match the declared inputs and controls. |
| `application/launch.yaml` | `command` and `args` (the shared server, `hutash_inference.server:create_app`), `env`, `port`, `health_endpoint`, `health_timeout`. |
| `application/packages.yaml` | `python`, then a flat `common:` list and `variants:` (`gpu`, `cpu`) with their `packages` and `indexes`. Not nested under a `packages:` key. |
| `resources/weights.yaml` | `sources:` (a Hugging Face `repo`, a pinned `revision` commit, `allow_patterns`) and `download_size_gb`. |

A complete working example is `real-esrgan-x4` (an image upscaler). Its core is small:

```yaml
# manifest.yaml (abridged)
hutash_format: '1.0'
id: real-esrgan-x4
name: Real-ESRGAN x4plus
version: 1.0.0
type: model_pipeline
gpu: optional
min_vram_gb: 2.0
min_ram_gb: 4.0
license: BSD-3-Clause
capabilities:
- id: upscale
  modality: upscale
metadata:
  weight_category: image
  quality_score: 0.85
ui:
  capabilities:
    upscale:
      inputs:   {image: {type: image_file, required: true}}
      controls: {scale: {type: enum, default: '4', options: [{value: '2'}, {value: '4'}]}}
      outputs:  {image: {type: png, role: primary}}
```

```python
# application/inference.py (abridged)
from hutash_inference import Inference, capability

class RealEsrganInference(Inference):
    def load(self):
        ...                       # load the weights once
        self.mark_loaded()

    @capability("upscale")
    def upscale_image(self, image: bytes, scale: int = 4):
        ...
        return {"image": result}  # a PIL image; the framework encodes the PNG
```

**Pin your weights.** Use a commit hash for `revision`, never a branch name, and list only the
files you need in `allow_patterns`. The engine downloads weights from Hugging Face repositories.

**The marketplace card.** A model's card reads four things you declare, and a missing one leaves a
hole in it: `metadata.quality_score` (0.0-1.0), `min_vram_gb` or `min_ram_gb`, the framework (the
engine detects it from `packages.yaml`), and `download_size_gb` in `weights.yaml`. See
[pipeline-format.md](reference/pipeline-format.md) section 4a. `hutash_dev_create_pipeline` warns
about each one you leave out.

### The ten capabilities

Every model of a capability takes and returns the **same** parameter names, so a workflow written
against `stt` works with any speech-to-text model. Declare the standard names exactly; a retired
spelling or a missing required input is an error. The full spec is
[capability-interfaces.md](reference/capability-interfaces.md).

| Capability | Required input | Output |
|---|---|---|
| `stt` | `file` (audio) | `text`, `segments` |
| `tts` | `text` | `audio` (wav) |
| `translate` | `text` | `text` |
| `llm` | `prompt` | `text` |
| `image` | `prompt` | `image` (png) |
| `audio-sep` | `file` (audio) | `vocals`, `instrumental` (wav) |
| `music` | `prompt` | `audio` (wav) |
| `voice-clone` | `text` | `audio` (wav) |
| `video` | `prompt` | `video` (mp4) |
| `upscale` | `image` | `image` (png) |

Optional inputs (a `language`, a `voice`, `width`, `scale`, ...) are listed in the spec.

### Endpoint and wire format

A capability is served at **`POST /{capability-id}`**: `/stt`, `/image`, `/upscale`, and so on. The id is
the key under `capabilities` in your manifest, whatever your Python method is called, and a manifest
does not declare an `endpoint`. Through the engine the same call is
`POST /packages/{package-id}/infer/{capability-id}`.

The request body is JSON with your inputs and controls under a `fields` key (flat JSON and a
multipart upload are also accepted):

```json
{"fields": {"image": "<base64 of the file>", "scale": 4}}
```

A file input is sent as base64 in JSON, or as a file part in a multipart request. A model that
returns text answers with JSON; one that returns an image, audio or video answers with those bytes
and the matching `Content-Type`.

## Creating a Vertical App

An application is a UI built from YAML on top of shared packages; a standard vertical contains no
meaningful Python. Its files:

```
my-app/
├── manifest.yaml                  identity, and which capabilities it needs installed
├── application/
│   ├── vertical.yaml              every page, widget and setting
│   ├── config.yaml                how it installs and launches
│   ├── packages.yaml
│   ├── requirements.txt
│   ├── workflows/*.yaml           what the app can run (capability steps)
│   └── api/main.py                a few lines: create_app(spec=SPEC)
└── resources/icon.svg
```

- `manifest.yaml` lists `capabilities_needed`; the app will not open until a model for each is installed.
- `vertical.yaml` composes pages from [widgets](#widget-registry). A widget's `bind:` names a workflow input.
- A workflow step calls a capability by its short id and passes every parameter by its standard name under `params:`.
- Projects and settings are stored under the Hutash data folder, in `local/apps/<app-id>/`.

Start from `template-app/` in your workspace and edit a copy; do not build one from scratch. The
full specification is [application-format.md](reference/application-format.md).

## Developer Mode

Turn it on in **Settings → Developer**. It creates your workspace, `<storage root>/developer/`:

```
developer/
├── apps/                 your applications
├── pipelines/            your model pipelines
├── template-app/        a complete working app to copy
├── template-pipeline/   a complete working pipeline to copy
├── AGENTS.md            instructions for a coding agent working in this folder
└── developer-guide.md   this guide
```

The tools are served over MCP at `POST /mcp/execute` with `{"tool": "<name>", "params": {...}}`, and
only while Developer Mode is on.

| Tool | Args | What it does |
|---|---|---|
| `hutash_dev_plan_app` | `description` | Call first. Maps an app description to the capabilities it needs, checks which are installed, and returns a build order. |
| `hutash_dev_check_capabilities` | none | Lists every installed capability and which package provides it. Check before building a new pipeline. |
| `hutash_dev_get_template` | `type` (`app` or `pipeline`) | Returns every file of the working template. |
| `hutash_dev_list_widgets` | none | Returns the widget catalogue (see [Widget Registry](#widget-registry)). |
| `hutash_dev_create_app` | `id`, `name`, `files`, optional `capabilities_needed` | Validates, then writes `developer/apps/<id>/`. Writes nothing and returns the errors if validation fails. |
| `hutash_dev_create_pipeline` | `id`, `name`, `files` | Validates, then writes `developer/pipelines/<id>/`. Missing marketplace fields are warnings, not errors. |
| `hutash_dev_load_app` | `id` | Installs `developer/apps/<id>/` into the engine from its folder. |
| `hutash_dev_load_pipeline` | `id` | Installs `developer/pipelines/<id>/` into the engine. It returns at once; the environment and weights install in the background. |
| `hutash_dev_remove_package` | `id`, `type` (`app` or `model`) | Uninstalls a loaded package. It leaves your folder in `developer/` alone. |
| `hutash_dev_get_status` | `id` | Install and run state of a loaded package. |
| `hutash_dev_get_logs` | `id` | The last 200 lines of the package's log. |
| `hutash_dev_list_dev_packages` | none | Lists everything currently loaded from a local folder. |
| `hutash_dev_test_inference` | `package_id`, `capability`, `input` | Calls one capability on an installed pipeline. |

`files` is `{relative path: content}` for every file of the package. Both `create_*` tools validate
before writing, so a bad call costs nothing.

**Pipeline workflow:** `check_capabilities` → `get_template` → `create_pipeline` → `load_pipeline`
→ poll `get_status` until it is ready → `test_inference` → fix and `create_pipeline` again, then
`load_pipeline` → `remove_package` when you are done.

**App workflow:** `plan_app` → build any missing pipeline first → `get_template` → `create_app` →
`load_app` → test it in the OS → `remove_package`.

A package you load from a folder is refused if its id is already installed from the catalogue, so it
cannot silently replace a real install.

## Widget Registry

An app's UI is composed from these widgets; name them in `vertical.yaml` and set their props. The
full list of props and defaults for each is the widget reference (`hutash_dev_list_widgets`).

| Group | Widgets |
|---|---|
| Inputs | `file_upload`, `dropdown`, `text_input`, `number_input`, `slider`, `toggle`, `model_selector` |
| Media | `video_player`, `audio_player`, `output_player`, `source_switcher`, `media_meta`, `subtitle_list`, `waveform_timeline`, `style_picker`, `hanu_panel` |
| Display and layout | `section_header`, `status_badge`, `progress_steps`, `toolbar`, `sidebar`, `tabbed_panel`, `save_status` |
| Page-scale | `project_grid`, `models_page`, `setup_guard`, `settings_page` |
| Actions | `action_button`, `export_controls` |

**Pick a control by the shape of the choice**, so the same question looks the same in every app:

| The choice | Control |
|---|---|
| Yes / no | `toggle` |
| One of 2-3 options | card selection (`export_controls` format cards) |
| One of 4 or more | `dropdown` |
| An action | `action_button` with a `variant`: `primary`, `secondary`, `ghost` or `danger` |
| Several items for one action | a checkbox per item and one `action_button` below |

If an existing widget does what you need, use it; if one does most of it, add a prop to it rather
than copying it. Use the design tokens for every colour, spacing and type value, and rem rather
than px for every dimension.

## Testing

- **`hutash_dev_test_inference`** calls a capability on an installed pipeline. `input` is a flat
  object of the capability's inputs and controls; a file input is the file's content,
  base64-encoded, not a path. An image, audio or video result comes back as
  `{content_type, size_bytes, data_base64}`.
- **Direct HTTP** against the engine, with the same body:
  `POST /packages/{package-id}/infer/{capability-id}` and `{"fields": {...}}`.
- **`hutash_dev_check_capabilities`** shows what is installed and which package serves each
  capability.
- **`hutash_dev_get_status`** and **`hutash_dev_get_logs`** are the first things to read when a
  load stalls or a call fails: the status names the install step that is running or failed, and the
  log shows the model server's own output.
- **Before you finish:** call every capability once with a small real input, call it once with a
  bad value to see the error, and confirm the response carries every key your `outputs:` declares.

## Publishing

### Internal (maintainer)

A package that ships to every user goes through the packages repository, `hutash-app`, and is
mirrored to this registry. Packages are authored in `hutash-app`, never in the registry.

1. Promote the source from your workspace to `hutash-app/models/<id>/` as five flat files:
   `manifest.yaml`, `inference.py`, `launch.yaml`, `packages.yaml`, `weights.yaml`.
2. **Normalise line endings to LF** before packing. A workspace on Windows writes CRLF, and the
   archive's sha256 depends on the exact bytes. The repository's `.gitattributes` enforces LF for
   `models/**/*.py` and `models/**/*.yaml`.
3. `python scripts/pack_model.py <id>` writes `hutash-app/pipelines/<id>.hutashm`.
4. `python scripts/build_index.py` regenerates the `hutash-app` index; `--check` fails if it is stale.
5. Copy the archive to `hutash-registry/pipelines/`.
6. `python scripts/build_index.py --root <path to hutash-registry>` regenerates the registry's index
   from the archives there. Never edit `index.json` by hand.
7. Commit both repositories on a branch, run their tests, and push both. Pushes to the registry are
   made by the maintainer from the terminal.

A model with no `quality_score` makes `build_index.py` fail, so the card is never published empty.

### External (third-party)

Build and test your package locally with Developer Mode. There is no submission process open yet:
the route is a pull request to this registry, reviewed before it merges, because a package contains
code that runs on a user's machine. The full contribution workflow is planned for v1.1.0.
