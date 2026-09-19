# `.hutash` vs `.hutashm` — the type distinction

Companion to `create-vertical.md` (application authoring reference) and
`hutash-app`'s `docs/reference/pipeline-format.md`
(model pipeline authoring reference). Read this first — it's the
architectural frame both of those sit inside.

**Source of this document:** the actual migration performed in this
workspace on 2026-09-12 — hutash-daemon commits `44e0e81` (recognize
`.hutashm` at the 6 check sites) and `48bd9c5` (stage pipelines under
`.hutashm` on disk), and hutash-app commit `9507436` (rename the 17
published pipelines). Every claim below traces to those commits or to
the code they left in place, not to invented framing. **Neither
`HUTASH_FORMAT.md` nor `REGISTRY.md` nor either `create-*.md` guide
mentions `.hutashm` at all as of this writing** — all four predate the
migration and describe a `.hutash`-only world. This document is the one
place the distinction is currently written down; the other docs should
eventually be updated to at least cross-reference it (out of scope for
this phase — see the note at the bottom).

## Same container, different content, different extension

A `.hutash` and a `.hutashm` file are **byte-for-byte the same kind of
archive** — a zip, containing `manifest.yaml` at the root and an
`application/` folder (Layer 1 + Layer 2 of `HUTASH_FORMAT.md`'s
three-layer spec). Nothing about the zip format, the compression, or
the Layer 1/2/3 structure changes between the two extensions. The
migration was named correctly in its own goal document: "a
naming/distribution change only, not an architecture change."

What differs is what `type:` the `manifest.yaml` inside declares, and
therefore what content is expected:

| | `.hutash` (application) | `.hutashm` (model pipeline) |
|---|---|---|
| `manifest.yaml` `type:` | `application` (normalises to `app` internally — `daemon/ext/packages/types.go:20-21,29-30`) | `model_pipeline` / `model` / `models` (all normalise to `pipeline` — same file) |
| `ui:` / `capabilities:` block | Forbidden — an app renders its own UI (`HUTASH_FORMAT.md` §2.4) | Required — this is the whole point of the package (§2.4) |
| `resources/weights.yaml` | Forbidden — apps manage their own downloads if any | Required for anything that downloads a model |
| `application/launch.yaml` | N/A — apps use `application/config/app.yaml` instead | Required |
| Renders a UI itself? | Yes — Studio, a vertical, a third-party tool | No — headless; something else's UI drives it |

## How the installer tells them apart

**Not by extension.** The daemon does not branch on `.hutash` vs
`.hutashm` anywhere to decide how to parse a package — `ReadPackage`
(`daemon/ext/packages/reader.go:62`) reads `manifest.yaml` from
whatever directory it's given and trusts the `type:` field inside. The
extension is a **distribution and local-storage convention**, derived
FROM the already-known type, never the other way around:

- **Deciding what to name the local staged folder** —
  `StagePackage` (`daemon/ext/packages/installer.go`, fixed in
  `48bd9c5`) computes the folder suffix from the package's own parsed
  `spec.Type`, via `packageDirExt(pkgType)`
  (`daemon/ext/packages/types.go`):

  ```go
  func packageDirExt(pkgType string) string {
      if pkgType == TypePipeline {
          return ".hutashm"
      }
      return ".hutash"
  }
  ```

  So the type is read from the manifest FIRST (during fetch/parse),
  and the extension is chosen SECOND, from that.

- **Finding an already-installed package again** — `packageDir()`
  (`daemon/ext/packages/registry.go`) has only an id to go on (not a
  type), so it tries both conventional suffixes (`.hutash` then
  `.hutashm`) before falling back to a full directory scan matching on
  the manifest's own `id` field. This is the one place the daemon
  genuinely treats the extension as ambiguous rather than derived —
  documented directly in that function's own comment as of `48bd9c5`.

- **Recognizing an installed folder as a real package at all** —
  `ListInstalled` (`registry.go`) accepts a directory ending in either
  suffix; a folder with neither is skipped, regardless of what's
  inside it.

- **hutash-app's `build_index.py`** (fixed alongside the daemon, commit
  `9507436`) enforces the reverse direction: `pipelines/` may only
  contain `*.hutashm`, `apps/`/`community/`/`plugins/` may only contain
  `*.hutash` — a file with the wrong extension in a folder now fails
  the index build loudly (`build()`'s per-folder `other_ext` check)
  instead of silently vanishing from `index.json` or being
  misclassified.

- **hutash-registry's local pre-push hook** (`.git/hooks/pre-push`,
  never committed — see that repo's own `AGENTS.md`) matches
  `*.hutash|*.hutashm` identically in its archive-scanning `case`
  statement; both extensions get unzipped and scanned for branding
  words the same way. This is local-only and must be reapplied by hand
  on any fresh clone of `hutash-registry`.

## Architectural framing

A **model pipeline** (`.hutashm`) is a headless capability provider. It
has no UI of its own — it declares one or more `capabilities` (`stt`,
`translate`, `tts`, …) in its manifest's `ui.capabilities` block
(`HUTASH_FORMAT.md` §5), the engine installs and runs it, and it is
reached exclusively through `POST /packages/{id}/infer/{endpoint}`. It
does not know or care who is calling it.

An **application** (`.hutash`) is a UI that *consumes* one or more
capabilities. A vertical (Hutash Subs, Podcast, Dub) declares
`capabilities_needed` in its own manifest and never names a specific
model — it asks the shared resolution mechanism (see the application
authoring reference's capability-resolution section) for "whatever
installed package can do `stt`," and gets back a package id it then
calls through the same `/infer/{endpoint}` route. Studio, and any other
UI-carrying app, is exactly the same shape, just usually consuming many
capabilities across many pipelines rather than one or two.

This is the same split `docs/architecture.md` already draws for the
engine as a whole (facts vs. verdicts, install vs. presentation) — a
pipeline is pure engine-managed fact-and-capability; an application is
where a human ever sees anything.

## Practical consequence for anyone authoring a new package

Decide the `type:` first — it is not a naming choice, it is an
architecture choice. If the thing you're building renders any UI a
human interacts with directly, it is an application (`.hutash`). If it
is a model that does inference and returns data, it is a pipeline
(`.hutashm`), and something else (a vertical, Studio, the CLI) will be
the thing a human actually sees. There is no third option, and nothing
in the daemon reads the extension to decide which of the two behaviors
to apply — the extension only ever follows a `type:` decision already
made inside the manifest.

## What this document does not cover

- The full `manifest.yaml`/`packages.yaml`/`launch.yaml`/`weights.yaml`
  field-by-field schema — see the model pipeline authoring reference
  (hutash-app) and `HUTASH_FORMAT.md` (hutash-os).
- The full `vertical.yaml` schema and widget catalogue — see the
  application authoring reference (this repo) and `REGISTRY.md` /
  `VERTICAL-UI-DESIGN-SPEC.md` (hutash-subs).
- Whether `HUTASH_FORMAT.md`, `REGISTRY.md`, `create-vertical.md`, and
  `create-model-pipeline.md` themselves get updated to mention
  `.hutashm` — they should, but that edit wasn't part of this audit
  phase (which was scoped to producing new reference material, not
  revising the existing four documents) and is flagged here as
  follow-up work rather than done silently.
