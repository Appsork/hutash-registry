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
  - tts
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
| `capabilities_needed` | no, but load-bearing | A flat list of capability ids this app needs installed to work — `tts`, `stt`, `translate`, `llm`, `clone`, `music`, ... This is what gates first launch (the app won't open until at least one model for each listed capability is installed) and what the generated Models page groups by. **This is a different key from a model pipeline's `capabilities:` field** — don't confuse the two; see `reference/package-types.md` if you want the full explanation of why they're separate. |

Do not add a `ui:` or `capabilities:` block here — those belong to a
model pipeline's manifest, not an application's. Full field-by-field
table, including what's parsed but unused for an application:
`reference/application-format.md` §6.

## Step 3 — `application/config.yaml`

How the engine launches the app's backend process.

```yaml
port: 0
health: /health
managed: true
entrypoint: api/main.py
```

| Field | Required? | Your options |
|---|---|---|
| `entrypoint` | not enforced at parse time, but the app won't start without it | The command/script the engine runs. `api/main.py` here assumes a `create_app(spec=SPEC)`-based Python backend (the framework every real shipped vertical uses) — see `reference/application-format.md` §"What the framework provides automatically" for what that gives you for free. |
| `port` | no | `0` means "engine assigns one from its own pool" — the normal choice; every real shipped vertical uses `0`. A fixed number is legal but never needed. |
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
| `requirements` | no | Name of a `requirements.txt` your app ships (relative to `application/`), installed alongside the shared framework's own deps. |
| `python` | no | Interpreter version string. `"3.12"` matches every real shipped vertical. |

This application-side `packages.yaml` is a **flat list/requirements
shape** — not the same schema as a model pipeline's own `packages.yaml`
(which is a `python`/`common`/`variants`/`system_packages` map). Same
filename, two different schemas depending on package type — see
`reference/package-types.md` if you want the full explanation.

Also write `application/requirements.txt` with whatever your backend
needs beyond the shared framework (often just empty, if you add nothing
beyond `create_app`'s own dependencies).

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
    capability: tts
    label: Voice Model
    description: Leave on Automatic to use the best installed model

setup:
  required_capabilities:
    - capability: tts
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
    capability: tts
    label: Voice Model
    description: Leave on Automatic to use the best installed model

setup:
  required_capabilities:
    - capability: tts
      label: Text to Speech
      recommended_model: kokoro
```

## Step 6 — `application/workflows/speak.yaml`

The one workflow: take the typed text, call an installed `tts` model,
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
    capability: tts
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
| `model: ${settings.tts_model}` | Resolves to whatever the Settings page's "Voice Model" picker holds (Step 5d). Left on Automatic (empty), the step falls back to the first installed model that declares `tts` — never a hard failure as long as *something* is installed. |
| `params.voice` | Passed to the model alongside the text. The exact param name a model expects is declared in *that model's own manifest* (`kokoro`'s, if that's what's installed) — `voice` is what this tutorial assumes; if you picked a different TTS model, check its manifest before assuming the name matches. Full resolution mechanics: `reference/application-format.md` §2. |
| `outputs.speech` | `type: file` because the model returns audio bytes, not text. `primary: true` marks it as the run's one deliverable — matters once a workflow has more than one output. `source: ${generate.output}` reads the `generate` step's raw result. |

Every `${...}` reference here is checked when the app loads: a typo'd
input name, or a reference to a step that hasn't run yet, fails loudly
at load time — before anyone presses the button. Full `${...}`
resolution rules and every step type: `reference/application-format.md` §5.

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
first-launch screen should ask to install a `tts` model (Step 5d's
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
