# The .hutash format

A `.hutash` file is how every model, app and update reaches Hutash. This section documents the format as the engine parses it.

## Context: The three parts

They stack in this order, and each layer only knows the one below it.

- **Hutash OS** — The window everything else runs inside. It lists the applications you have, installs new ones from a catalogue, and gives each one its tab.
- **Hutash Studio** — The flagship application — one screen per kind of generation (speech, music, transcription, translation, chat), where you pick a model, fill in a form the model itself describes, and press Generate.
- **The engine** — `hutashd` — no interface at all. It downloads model weights, manages Python environments, ports, and GPU memory for you, and starts and stops model processes on demand.

## Definition: What a package is

A `.hutash` file is a zip archive with a fixed three-part structure: `manifest.yaml` at the root (who this is), an `application/` folder (how it runs and what it depends on), and an optional `resources/` folder (what data must be downloaded).

### An inference pipeline

`type: model_pipeline` — a single model with no interface of its own. It ships an `inference.py`, a list of Python packages, a pointer at weights on HuggingFace, and, under a `ui:` key in its manifest, a declaration of the inputs and controls it accepts, which Studio reads and renders as a form.

### A full application

`type: application` — brings its own interface: either a third-party program pinned to an exact git commit, which the engine clones and launches, or a Hutash *vertical* like Subs or Podcast, whose entire interface is itself a YAML file (`vertical.yaml`) rendered by a shared widget catalogue, so the package contains no interface code of its own either.

The same extension, the same three layers, the same installer, the same update path — a new version of anything is a new `.hutash` in the catalogue.

## Layouts: Three layouts ship today

`type:` in `manifest.yaml` is the discriminator between the first and the other two; whether the code is inside the zip separates the second from the third.

### A — model pipeline

`type: model_pipeline` — for example `kokoro.hutash`, `whisper-tiny.hutash`.

```
kokoro.hutash (zip)
├── manifest.yaml                 identity + the ui: contract Studio renders
├── application/
│   ├── packages.yaml             Python dependencies
│   ├── launch.yaml               how to start the inference server
│   └── inference.py              the model code
└── resources/
    └── weights.yaml              which HuggingFace files to download
```

### B — external application

`type: application`, code fetched from elsewhere — for example `hivision-idphotos.hutash`, `hutash-studio.hutash`.

```
hivision-idphotos.hutash (zip)
├── manifest.yaml                 identity only — no ui: contract
└── application/
    ├── config/
    │   └── app.yaml              source:, entrypoint, ports, health, build
    └── packages.yaml             dependencies
```

### C — Hutash vertical

`type: application`, code ships inside the zip — `hutash-subs.hutash`, `hutash-podcast.hutash`.

```
hutash-subs.hutash (zip)
├── manifest.yaml                 identity + capabilities_needed + a small ui: block
├── requirements.txt              the Python dependency list packages.yaml points at
├── application/
│   ├── config.yaml               port, health, managed, entrypoint
│   ├── packages.yaml             { requirements: requirements.txt, python: "3.12" }
│   ├── vertical.yaml             THE ENTIRE USER INTERFACE
│   ├── workflows/*.yaml          what the app can do
│   ├── api/                      the app's own backend routes
│   └── dist/                     the built frontend shell
└── resources/                    (empty — verticals download no weights of their own)
```

Layout C is what the two shipped verticals contain. Against layout B, the differences to author for are: `application/config.yaml` rather than `application/config/app.yaml`; no `source:` block anywhere, because the code is inside the zip; a root `requirements.txt`, with `packages.yaml` reduced to a pointer at it; and a `ui:` block carrying only `display_name`, `icon` and `category` — store-listing data, not a control contract, and not read by the engine for an app.

## Continue: The rest of this section

- [Format reference](reference/index.html) — Every file and every field, as the engine reads them.
- [Widget catalogue](widgets/index.html) — What a declared input or control renders as, on both sides of the format.
- [Walkthrough](walkthrough/index.html) — Package a model from scratch, install it, and run it.
