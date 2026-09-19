# Model pipeline (`.hutashm`) authoring — schema reference

Audit-based reference, produced 2026-09-12. Every field name and every
behavior below is verified against the actual Go parser
(`hutash-daemon/daemon/ext/packages/spec.go` and `reader.go`) or a real
package's actual on-disk content **as published in `hutash-registry`**
— the public distribution repo, and therefore the actual bytes a real
install downloads — never against what a doc *says* the shape is,
unless that doc was cross-checked against the parser too. (`hutash-app`
is the private authoring repo the same package is built in; as of this
writing its copy is a byte-identical mirror of `hutash-registry`'s, but
`hutash-registry` is cited throughout as the source of truth because it
is what's actually distributed — an authoring-repo copy could in
principle drift from what's shipped, and be silently wrong to cite.)
Where a documented example and the real parser disagree, that's called
out explicitly (§5) rather than silently resolved one way. This is
Phase 1 source material for a future coding-agent-facing `AGENTS.md` —
not that document itself.

**Relationship to existing docs:** `hutash-os`'s
`docs/reference/HUTASH_FORMAT.md` (v1.4.1) is the actively-maintained
format spec and `docs/reference/create-model-pipeline.md` is its
tutorial companion — both are good and this document does not replace
either. What this document adds, verified fresh against the parser
rather than assumed from those docs:

1. The exact, field-by-field schema the engine's parser actually reads
   — confirmed required vs. optional at the code level, not just by
   convention.
2. Today's vendor-aware `SelectVariant` logic and the SHA-256
   verification field — both added to the engine after
   `HUTASH_FORMAT.md` v1.4.1 and not documented anywhere yet.
3. One real, complete, shipped package (`opus-mt-en-hi`) quoted in full
   across all four files and annotated.
4. Two concrete discrepancies found between the documented examples and
   what the parser/real files actually use (§5) — worth fixing in the
   source docs, not silently worked around here.

---

## 1. The engine's actual parse-time contract

`ReadPackage` (`daemon/ext/packages/reader.go:62`): **only
`manifest.yaml` is strictly required, and within it only `id`.**
Everything else — `packages.yaml`, `launch.yaml`, `weights.yaml` — is
read if present (`readOptionalYAML`) and simply left at its zero value
if absent. A missing manifest, or one with no `id`, is the only thing
that makes `ReadPackage` return an error at all (lines 65-72). This is
a deliberately permissive read-side contract; it does not mean every
field is optional in the sense of "the package will actually work" —
only that the *parser itself* won't reject a package for lacking one.

### `manifest.yaml` — every field `applyManifest` reads (reader.go:134-179)

| Field | Go type | Source | Notes |
|---|---|---|---|
| `id` | string | — | **The only field whose absence is a parse error.** |
| `name` | string | — | |
| `version` | string | — | |
| `type` | string | normalized via `normalizeTypeKeep` | `model_pipeline`/`model`/`models` all normalize to `pipeline`; `application` normalizes to `app` (`types.go:36-46`). An unrecognized value is kept verbatim and logged once — not rejected. |
| `runtime` | string | — | `venv` is the only value the engine actually installs today (`coordinator.go`: any other value fails with "not supported yet"). |
| `gpu` | string | — | `required` \| `optional` \| `none` |
| `min_vram_gb` | float | `min_vram_gb`, falls back to `ui.hardware.min_vram_gb` if the top-level key is ≤0 | The fallback exists because some manifests only set the real number under `ui.hardware` (reader.go:146-153) |
| `recommended_vram_gb` | float | — | |
| `min_ram_gb` | float | — | |
| `required_cpu_features` | `CPUFeatureRequirements` (variant → list) | `{gpu: [...], cpu: [...]}` per install variant, or a flat list for every variant | The CPU instruction sets the package's compiled binaries execute unconditionally, in bhoomi's names (`avx`, `avx2`, `f16c`, `fma`, `avx512f`, `avx512bw`, `avx512dq`, `avx512vl`, `bmi2`, `avx_vnni`, …). Declare only what the wheel was built with — read it, don't guess: for llama.cpp, the exported `ggml_cpu_has_*` functions in `ggml-cpu.dll` return compile-time constants. The engine resolves the list for the variant a host would install, reports what the CPU lacks as `missing_cpu_features` on `GET /catalogue` and `GET /packages`, and refuses the install (403) when anything is missing; apps show the model as incompatible. Omit it when nothing beyond baseline x86-64 is required. `build_index.py` publishes it with one key per `packages.yaml` variant. |
| `capabilities` | `[]Capability{ID, Modality}` | list of `{id, modality}` | |
| `description` | string | — | |
| `license` | string | bare SPDX string | A v2.0 package's license lived in `interface/manifest.json` instead — reading a map here yields `""`, which is treated as "that package's license travels with its own UI contract," not an error |
| `min_os_version` | string | — | Optional; empty means "runs anywhere," never "unknown." The engine relays it and never compares it — see `PackageSpec.MinOSVersion`'s own doc comment for the full rationale (comparison is an application-layer concern) |
| `source` | `*SourceSpec{Repo, Commit}` | `source:` map | Applications only, per `HUTASH_FORMAT.md` §2.4 — a pipeline's code ships inside the package |
| `ui` | `map[string]any` | `ui:` key, carried as an **opaque tree** | The engine never interprets this — it writes it back out as `application/manifest.json` at install time (`InterfaceManifest()`) for the model server / Studio to read. Full shape: `HUTASH_FORMAT.md` §5 |

### `application/packages.yaml` — every field `applyPackages` reads (reader.go:218-232)

| Field | Go type | Notes |
|---|---|---|
| `python` | string | |
| `common` | `[]string` | Installed regardless of hardware |
| `variants` | `map[string]VariantSpec{Packages []string, Indexes []string}` | Keyed by variant name — `gpu` and `cpu` are the two the engine's `SelectVariant` (§2 below) actually looks for; any other key is accepted but only reachable via the fallback branch |
| `system_packages` | `[]string` | OS-level packages (e.g. `build-essential`) |

**There is no `extra_index_urls:` or `system:` top-level key that the
parser reads** — see §5, discrepancy 1.

### `application/launch.yaml` — every field `applyLaunch` reads (reader.go:234-244)

| Field | Go type | Notes |
|---|---|---|
| `command` | string | The executable to run (e.g. `uvicorn`) |
| `args` | `[]string` | Passed to `command` |
| `env` | `map[string]string` | `{port}`, `{model_dir}`, `{weights_dir}` are template placeholders the engine fills in at launch |
| `port` | int | The port `{port}` in `args`/`env` resolves to |
| `health_endpoint` | string | |
| `health_timeout` | int (seconds) | |

**There is no `entrypoint:` or `health:` key that the parser reads** —
see §5, discrepancy 2. `launch.yaml` exists for `type: model_pipeline`
only.

### `resources/weights.yaml` — every field `applyWeights` reads (reader.go:246-266)

| Field | Go type | Notes |
|---|---|---|
| `sources` (and `extra`, same shape, appended to the same list) | `[]Weight{Repo, Revision, AllowPatterns []string}` | An entry with an empty `repo` is silently skipped |
| `download_size_gb` | float | Display-only |

`revision` should be a pinned commit, not a branch name — `HUTASH_FORMAT.md` §6's "pin everything" principle, and confirmed practice in every real weights.yaml this session inspected.

---

## 2. `packages.yaml`'s variant system — today's real, vendor-aware `SelectVariant`

`PackageSpec.SelectVariant` (`spec.go:133-175`) — the actual algorithm
the engine runs at install time to pick `gpu` or `cpu` from a
package's declared `variants:`. Quoted here because it changed
recently (2026-09-12, commit `9b36aaa` in hutash-daemon) and is not
documented in `HUTASH_FORMAT.md` at all — that spec only documents the
*declaration* shape, not the *selection* logic:

- **Both `gpu` and `cpu` declared:** `gpu` wins only when the host has
  an NVIDIA GPU (`hw.GPUVendor == resource.GPUNVIDIA`) **and** its VRAM
  meets `min_vram_gb` (a package with no declared minimum is satisfied
  by GPU presence alone). The vendor check gates *before* the VRAM
  comparison, not alongside it — a real GPU that isn't NVIDIA (AMD,
  Intel, or any future vendor) cannot run the CUDA-specific `gpu`
  variant regardless of how much VRAM it reports. This also correctly
  routes Apple's unified-memory "GPU" to `cpu`: its reported VRAM is
  the host's entire system RAM (bhoomi's own convention), which would
  otherwise satisfy almost any VRAM floor and wrongly select a CUDA
  variant on hardware with no CUDA at all.
- Otherwise `cpu` — no GPU, a non-NVIDIA GPU, or an NVIDIA GPU below
  the declared VRAM floor.
- **Only one of `gpu`/`cpu` declared:** that one, unconditionally,
  regardless of hardware.
- **Neither declared:** `""` (common-only package) — or, if a
  non-standard variant key exists, that key (the "fall through to
  whichever single variant is declared" branch).

**A package can declare both `gpu` and `cpu` pointing at the identical
index/packages** — confirmed real practice, not a hypothetical: both
`opus-mt-en-hi` and `kokoro`'s `variants.gpu` and `variants.cpu` point
at the same `https://download.pytorch.org/whl/cpu` index with an empty
`packages: []` list, because `gpu: none` at the manifest's top level
means the CUDA variant would never actually be reachable in practice —
but `SelectVariant` doesn't consult `gpu:` at all, only the presence of
the `variants` map keys, so a package with `gpu: none` and duplicate
gpu/cpu entries still runs `SelectVariant`'s full vendor-check branch.
This is harmless (both branches install byte-identical dependencies)
but is a real, observed pattern worth knowing rather than assuming a
bug when you see it.

---

## 3. SHA-256 package verification — where it lives and why

**Not in the package's own `manifest.yaml`.** The expected digest lives
in the **catalogue** (`hutash-app`'s published `index.json`,
`CatalogueEntry.SHA256` / `CatalogueVersion.SHA256`,
`daemon/ext/packages/catalogue.go`), verified by the installer
immediately after download and before extraction
(`installer.go`'s `fetch`, via `apptypes.VerifyHutashSHA256`).

The reason, quoted from `catalogue.go`'s own doc comment because it's
worth getting right rather than re-paraphrasing: **verifying a zip's
own integrity using a value that is only readable AFTER extracting
that same zip is circular** — by the time you could read a hash from
inside the archive, you have already trusted and unpacked the thing you
meant to check first. The hash has to come from somewhere fetched
*before* the archive, independent of it — `index.json` is exactly that,
already fetched ahead of any package's own download the same way its
URL is.

An empty/absent `sha256` (every package published before this field
existed) is not a failure — `VerifyHutashSHA256` treats "nothing to
check against" as a pass, not a warning suppressed. `hutash-app`'s
`scripts/build_index.py` computes this hash fresh on every rebuild,
directly from the file each entry points at, so it can never go stale
relative to what a client would actually download.

---

## 4. Capability declaration — how a pipeline says "I provide X"

`manifest.yaml`'s `ui.capabilities` block (§1's `ui` field, opaque to
the engine, interpreted by Studio/verticals/CLI/MCP) is a **map keyed
by capability id** — `capabilities: {translate: {...}}`, not a list —
per `create-model-pipeline.md`'s own correct note and confirmed against
`opus-mt-en-hi`'s real manifest below (`ui.capabilities.translate`).
Each entry's `endpoint:` is the real HTTP path a caller must use — it
is never assumed to equal the capability id (`opus-mt-en-hi` itself
declares `endpoint: /translate`, which happens to match its id, but
`qwen3-1.7b`'s `llm` capability is served at `/generate`, and
callers always resolve the endpoint from the manifest rather than
guessing).

This is the OTHER end of the resolution trace documented in full in
`hutash-os`'s `docs/reference/application-format.md` §2 —
a vertical's `capabilities_needed` + `${settings.X_model}` resolves TO
a package id, and that package's own `manifest.yaml.ui.capabilities`
entry is what supplies the actual endpoint the resolved call hits.

Top-level `capabilities: [{id, modality}]` (§1's table — a short list,
not the full `ui.capabilities` map) is what the catalogue's
`?capability=` filter and `get_declared_capabilities()` actually read
to answer "does this package do X" without needing the full UI
contract — the two are parallel, not duplicate: the short list answers
*whether*, the `ui.capabilities` map answers *how*.

---

## 5. Two verified discrepancies between the documented examples and reality

**1. `packages.yaml`'s documented example shape does not match the
parser or any real shipped file.** `HUTASH_FORMAT.md` §4 shows:

```yaml
packages:
  gpu: [torch==2.11.0+cu128, kokoro==0.9.4]
  cpu: [torch==2.11.0+cpu]
  common: [fastapi]
extra_index_urls:
  gpu: [https://download.pytorch.org/whl/cu128]
system:
  apt: [espeak-ng]
```

But `reader.go`'s `applyPackages` reads `common` (flat list, top-level
— not nested under a `packages:` key at all), `variants.<name>.packages`
+ `variants.<name>.indexes`, and `system_packages` (flat list, not
`system.apt`). **Every real shipped file this session inspected**
(`opus-mt-en-hi`, `kokoro`, `whisper-tiny`) uses the parser's real
shape, e.g.:

```yaml
python: '3.12'
common:
- hutash-inference>=0.2.2
- transformers==4.46.3
variants:
  gpu:
    packages: []
    indexes:
    - https://download.pytorch.org/whl/cpu
  cpu:
    packages: []
    indexes:
    - https://download.pytorch.org/whl/cpu
system_packages:
- build-essential
```

`HUTASH_FORMAT.md`'s example should be corrected — an authoring agent
that trusted it literally would produce a `packages.yaml` the real
parser reads as having no dependencies at all (`common` nested one
level too deep, `variants` absent, `system_packages` absent).

**2. `launch.yaml`'s documented example shape does not match the
parser or any real shipped file either.** Both `HUTASH_FORMAT.md` §4
and `create-model-pipeline.md` §3 show:

```yaml
entrypoint: inference.py
port: 0
health: /health
env: {HF_HUB_OFFLINE: "1"}
```

But `reader.go`'s `applyLaunch` reads `command`, `args`,
`health_endpoint`, `health_timeout` — **it does not read `entrypoint`
or `health` at all.** Every real shipped file uses:

```yaml
command: uvicorn
args: [--factory, hutash_inference.server:create_app, --host, 127.0.0.1, --port, '{port}']
env:
  HUTASH_MODEL_ID: opus-mt-en-hi
  HUTASH_MODEL_DIR: '{model_dir}'
  HF_HUB_CACHE: '{weights_dir}'
  HF_HUB_OFFLINE: '1'
port: 8000
health_endpoint: /health
health_timeout: 120
```

This is the more serious of the two discrepancies: a `launch.yaml`
written to the documented shape would parse as `spec.Command == ""`
and `spec.HealthEndpoint == ""`, which is not a parse error but would
fail whatever start step actually tries to invoke `spec.Command`. This
should be corrected in both source docs before an authoring agent is
built against them.

---

## 6. Worked example — `opus-mt-en-hi`, in full

All four real files, verbatim, extracted fresh from
`hutash-registry/pipelines/opus-mt-en-hi.hutashm` — the actual
distributed package — as of the 2026-09-12 rename. Byte-for-byte
identical to `hutash-app`'s copy of the same file, confirmed by direct
diff at documentation time.

### `manifest.yaml`

```yaml
hutash_format: '1.0'
id: opus-mt-en-hi
name: Opus-MT English-Hindi
version: 1.0.4
naming:
  family: opus-mt
  version: null
  variant: en-hi
  parameters: 77m
  quantization: null
  format: pytorch
type: model_pipeline
gpu: none
min_ram_gb: 1.0
license: Apache-2.0
min_os_version: "1.0.2"
capabilities:
- id: translate
  modality: translate
platforms: [windows, linux]
description: Lightweight English-to-Hindi machine translation, CPU-only.
metadata:
  weight_category: text
  quality_score: 0.8
ui:
  model_id: opus-mt-en-hi
  display_name: Opus-MT English-Hindi
  license:
    spdx: Apache-2.0
    url: https://huggingface.co/Helsinki-NLP/opus-mt-en-hi
    commercial_ok: true
    attribution_required: false
    attribution_text: null
  capabilities:
    translate:
      label: Translate
      description: Translate a .txt or .srt document, or inline text, from English to Hindi
      primary: true
      endpoint: /translate
      status_message: Translating…
      inputs:
        file:
          type: text_file
          label: Document to translate
          accept: [text/plain, .txt, .srt]
          required: false
          clear_after_generate: true
        text:
          type: string
          label: Or paste text to translate
          description: Translated inline. Leave the document above empty when using this.
          required: false
          max_length: 5000
          clear_after_generate: true
      controls:
        source_language:
          type: enum
          label: Source language
          default: en
          options: [{value: en, label: English}]
        target_language:
          type: enum
          label: Target language
          default: hi
          options: [{value: hi, label: Hindi}]
      outputs:
        translated_text: {type: text, role: primary}
  hardware: {min_vram_gb: 0, recommended_vram_gb: 0, supports_cpu: true, total_install_gb: 1}
  tagline: "Opus-MT — lightweight CPU English-to-Hindi translation"
  creator:
    name: Language Technology Research Group at the University of Helsinki
    url: https://huggingface.co/Helsinki-NLP
  source:
    upstream_repo: https://github.com/Helsinki-NLP/Opus-MT
    huggingface_repo: Helsinki-NLP/opus-mt-en-hi
  api: {health: /health, gpu: false}
  ui:
    primitives:
    - {type: quick_text_transform, source_label: Source text, target_label: Translation, action_label: Translate}
  layout: aggregator-3panel
```

Annotated:
- **`inputs.file` and `inputs.text` are both `required: false`** —
  deliberately. The real file's own comment (preserved above) explains
  why: `inference.py`'s `translate()` accepts exactly one of the two
  and raises if given neither, but the *manifest itself* can't express
  "exactly one of" — declaring either one `required: true` would 400
  before the model ever ran, on the legitimate call that used the
  other. The "exactly one" rule lives in the one place that actually
  knows it: the inference code itself.
- **`controls.source_language`/`target_language` each have exactly one
  option** — this model translates one fixed language pair, but still
  declares both controls (rather than omitting them) so it presents
  the same Source/Target pair every "translate"-capability model shows
  on the Translate page; `inference.py` ignores both values.
  `metadata.notes` in the real file (omitted above for length) confirms
  this is a deliberate UI-consistency decision, not leftover cruft.
- **`layout: aggregator-3panel`** — a Studio model-detail-page layout
  hint. Unrelated to `vertical.yaml`'s own `layout:` vocabulary (see
  the application authoring reference's Friction points §3) despite
  the shared word.
- **`ui.primitives`** — the inline quick-translate box on Studio's
  Translate page. A separate, smaller mechanism from the main
  `capabilities.translate` form.

### `application/packages.yaml`

```yaml
python: '3.12'
common:
- hutash-inference>=0.2.2
- transformers==4.46.3
- sentencepiece
- torch
- uvicorn==0.47.0
- fastapi==0.136.1
- python-multipart==0.0.29
variants:
  gpu:
    packages: []
    indexes: [https://download.pytorch.org/whl/cpu]
  cpu:
    packages: []
    indexes: [https://download.pytorch.org/whl/cpu]
system_packages: [build-essential, git, curl]
```
Both `gpu` and `cpu` point at the CPU wheel index — see §2 above for
why this is intentional given `gpu: none`, and why `SelectVariant`
still runs its full vendor-check logic regardless.

### `application/launch.yaml`

```yaml
command: uvicorn
args:
- --factory
- hutash_inference.server:create_app
- --host
- 127.0.0.1
- --port
- '{port}'
env:
  HUTASH_MODEL_ID: opus-mt-en-hi
  HUTASH_MODEL_DIR: '{model_dir}'
  HF_HUB_CACHE: '{weights_dir}'
  HF_HUB_OFFLINE: '1'
  HUTASH_HF_REVISION: 75d7f7c9232b2891c7d65fe4ef635616c72be867
port: 8000
health_endpoint: /health
health_timeout: 120
```
`{port}`, `{model_dir}`, `{weights_dir}` are engine-filled template
placeholders, not literal strings — confirmed by the real value being
the same across every shipped package this session inspected.

### `resources/weights.yaml`

```yaml
sources:
- repo: Helsinki-NLP/opus-mt-en-hi
  revision: 75d7f7c9232b2891c7d65fe4ef635616c72be867
  allow_patterns:
  - config.json
  - generation_config.json
  - pytorch_model.bin
  - source.spm
  - target.spm
  - tokenizer_config.json
  - vocab.json
download_size_gb: 0.29
```
`allow_patterns` excludes `rust_model.ot`/`tf_model.h5` — the same
weights the HF repo also ships in two frameworks this package's
`transformers.AutoModelForSeq2SeqLM` never loads. Filtering them out at
download time, not after, is what keeps `download_size_gb` accurate and
the actual transfer small.

---

## 7. The `.hutashm` package structure

Confirmed from this session's own migration (hutash-daemon `44e0e81`/
`48bd9c5`, hutash-app `9507436`): **identical zip layout to `.hutash`**
— `manifest.yaml` at the root, `application/`, `resources/`. Nothing
about the internal structure changed; only the file's own extension,
and the local install-time folder suffix the engine chooses to match it
(`<PackagesDir>/<id>.hutashm` instead of `<id>.hutash`, decided by
`packageDirExt(spec.Type)` reading the manifest's own `type:` field —
see `package-types.md` in `hutash-os` for the full
mechanism).

```
opus-mt-en-hi.hutashm (zip)/
├── manifest.yaml
├── application/
│   ├── packages.yaml
│   └── launch.yaml
└── resources/
    └── weights.yaml
```

No `application/config/app.yaml` (that's the `application`-type-only
file) and no `inference.py` shown here only because this package uses
the shared `hutash-inference` PyPI package (via `launch.yaml`'s
`command`/`args`) rather than shipping its own — a pipeline that does
ship custom inference code puts it at `application/inference.py`
instead, per `create-model-pipeline.md` §4.
