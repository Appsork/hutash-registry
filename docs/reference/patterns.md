# Patterns — copyable skeletons

Three app shapes and two techniques, as skeletons to copy and edit, not
prose to read. Copy the YAML, replace what the comments say. Full schema:
`application-format.md`. Widget props: `widgets.md`. Build
`getting-started.md`'s tutorial first — everything below assumes that shape.

---

## Pattern 1 — single-page app

One page, no persistence. Right for input-in, output-out in one shot —
`getting-started.md`'s `hutash-say` is this pattern, built in full there.

```yaml
# application/vertical.yaml
app:
  title: Your App Name
  description: One sentence, shown in the window chrome
pages:
  - id: home
    label: Home
    default: true            # exactly one page sets this
    layout: single
    widgets:
      - type: section_header
        position: main
        props: {title: What this page does}
      # ... your input widgets (bind: to a workflow input id) ...
      - type: action_button
        position: main
        props: {label: Run, action: run_workflow, requires: [your_input_id], workflow: your_workflow_id}
      - type: progress_steps
        position: main
        props: {visible_during: workflow_run}
      - type: output_player       # or a different output widget — see widgets.md
        position: main
        props: {output: your_output_id, label: Result}
  # Mandatory boilerplate — copy verbatim, every app needs this page:
  - id: hanu
    label: Hanu
    overlay: true
    overlay_size: md
    layout: single
    widgets: [{type: hanu_panel, position: main}]
workflows:
  - {id: your_workflow_id, file: workflows/your_workflow.yaml, label: Run, icon: mic, description: One sentence}
setup:
  required_capabilities:
    - {capability: your_capability, label: Human name, recommended_model: some-model-id}
    # ^ capability must also be in manifest.yaml's capabilities_needed
```

Need to save work and reopen it later? That's Pattern 2.

---

## Pattern 2 — three-page project app

`projects` → `new_project` (upload+run dialog) → `editor` (work surface).
Every real shipped vertical bigger than the tutorial uses this shape.
Persistence is entirely framework-provided — you declare pages, it stores.

```yaml
# application/vertical.yaml
app:
  title: Your App Name
  description: One sentence
project:
  subfolders: [media, exports]   # add your own, e.g. [media, transcripts, exports]
pages:
  # 1. Landing page — lists existing projects
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
        # create_button: false — section_header's button already offers "New
        # Project"; two controls for one action is the bug.
        props: {create_button: false, create_opens: new_project, opens: editor}
  # 2. New-project dialog — an overlay, not a page you navigate TO and stay on
  - id: new_project
    label: New
    hidden: true              # not in the main nav — only reachable via navigate
    overlay: true
    dialog: true
    layout: single
    widgets:
      - type: file_upload
        bind: media_file
        props: {label: Drop your file here, accept: [video/*, audio/*]}
      - type: text_input
        bind: project_name
        props: {label: Project name, placeholder: Untitled, autofill_from: media_file}
      - type: action_button
        props:
          label: Create
          action: run_workflow
          requires: [media_file]
          workflow: your_workflow_id
          create_project: "${inputs.project_name}"   # <- this line IS project creation
          then_navigate_to: editor
          navigate_params: {projectName: "${inputs.project_name}"}
      - type: progress_steps
        props: {visible_during: workflow_run}
  # 3. The work surface — opens on an existing project
  - id: editor
    label: Editor
    hidden: true
    layout: three-panel
    left_width: auto
    right_width: 26rem
    confirm_leave: "Leave this project? Unsaved edits will be lost."
    widgets:
      # A page carrying `sidebar` gets ONLY the breadcrumb up top — the
      # rail already names these destinations.
      - type: sidebar
        position: left
        props:
          items:
            - {label: Projects, icon: folder, action: navigate, to: projects}
            - {label: Models, icon: marketplace, action: navigate, to: models}
            - {label: Hanu, icon: chat, action: navigate, to: hanu}
            - {label: Settings, icon: gear, action: navigate, to: settings}
            - {label: Save, icon: disk, action: save}
      - type: video_player          # or audio_player — whichever fits your media
        position: center
        props: {show_subtitles: true}
      - type: tabbed_panel
        position: right
        props:
          default_tab: main
          tabs:
            - id: main
              label: Main
              widgets:
                - {type: subtitle_list, props: {editable: true}}   # or whatever widget edits your content
            - id: export
              label: Export
              widgets:
                - type: export_controls
                  props: {save_folder: exports, formats: [{value: srt, label: "SRT"}]}
  # Mandatory boilerplate — copy verbatim:
  - id: hanu
    label: Hanu
    overlay: true
    overlay_size: md
    layout: single
    widgets: [{type: hanu_panel, position: main}]
workflows:
  - {id: your_workflow_id, file: workflows/your_workflow.yaml, label: Run, icon: mic, description: One sentence}
```

**Do NOT declare `models` or `settings` as pages** — auto-generated from
`manifest.yaml`'s `capabilities_needed` and `vertical.yaml`'s `settings:`
block; the sidebar navigates to them by id. Redeclaring either is a second
design of something that already exists (rare exception:
`application-format.md`'s "Take-over rule").

**Automatic vs. what you write:** `create_project` creates the project;
`project.subfolders` names the folders your workflow writes into;
`subtitle_list` with `editable: true` autosaves on its own — you never
write project CRUD (full list: "What the framework provides automatically"
in `application-format.md`).

---

## Pattern 3 — multi-step pipeline workflow

Several steps, later ones consuming earlier ones' output, composed from a
sub-workflow. Full step-type reference: `application-format.md` §5.

```yaml
# application/workflows/your_workflow.yaml
id: your_workflow_id
name: Your Workflow
version: 1.0
inputs:
  media_file: {type: file, label: Input file, required: true}
steps:
  # Step 1: pre-process with ffmpeg (built-in actions: extract_audio,
  # convert_format, get_duration, get_metadata, burn_subtitles_ass, replace_audio)
  - id: extract_audio
    name: Extract Audio
    type: ffmpeg
    action: extract_audio
    input: ${inputs.media_file}
    params: {format: wav, sample_rate: 16000, channels: 1}
  # Step 2: call an installed model — references step 1's OUTPUT, not the input
  - id: transcribe
    name: Transcribe
    type: capability
    capability: stt
    model: ${settings.stt_model}          # empty = auto-pick an installed model
    input: ${extract_audio.output}        # <- chaining: previous step's result
    params: {word_timestamps: true}
  # Step 3: reuse a WHOLE other workflow as one step (composition)
  - id: translate_step
    name: Translate
    type: workflow
    workflow: translate                    # application/workflows/translate.yaml
    inputs: {segments: ${transcribe.output}}  # parent's step output -> child's input
  # Step 4: `.outputs.name` (not `.output`) is how a `type: workflow` step's
  # own named outputs are reached — `.output` there is its WHOLE outputs dict.
  - id: format_output
    name: Format
    type: formatter
    format: srt                              # srt | vtt | txt | json | a registered custom format
    input: ${translate_step.outputs.result}
outputs:
  result:
    type: subtitle_editor      # subtitle_editor | text | file
    source: ${format_output.output}
    label: Result
    primary: true               # exactly one output per workflow may set this
```

**Chaining rule:** a step may reference only inputs, settings, or a **preceding** step — a later one is a load-time error. **Intermediate files** live in the run's scratch directory and vanish with it unless given their own `outputs:` entry with `type: file`.

---

## Technique — adding a custom output format

Built-in formats are `srt`/`vtt`/`txt`/`json`. For anything else (e.g. an
`.ass` file with burned-in karaoke timing), register one from your backend.

```python
# application/api/main.py
from hutash_workflow.server import create_app
# CORRECT import — register_format is NOT exported from top-level
# hutash_workflow, only this submodule. Importing from `hutash_workflow`
# raises ImportError every time; caught by a bare try/except, that's
# SILENT — every run then hits "unsupported formatter format 'x'".
from hutash_workflow.steps.formatter_step import register_format

SPEC = {"id": "your-app-id", "workflows_dir": "..."}
app = create_app(spec=SPEC)

def render_my_format(segments, params: dict | None = None) -> bytes:
    """segments: list of {start, end, text, words?}. Return the document's bytes."""
    ...

register_format("my_format", render_my_format)   # now `format: my_format` works anywhere
```

For a `formatter` step producing something beyond SRT/VTT/TXT/JSON — not a new MEDIA file (that's `capability`/`ffmpeg`); formatters only convert timed-text segments to a different *text* document.

---

## Technique — app-specific backend logic vs. capability steps

| What you need | Use |
|---|---|
| Call an installed AI model | `type: capability` step |
| Audio/video ffmpeg already handles | `type: ffmpeg` step |
| Timed-text format conversion | `type: formatter` step (built-in, or `register_format`) |
| Anything else app-specific, as one step in a chain | **No extension point exists — see below.** |

**No mechanism registers a custom workflow step type.** Dispatch (`capability`/`ffmpeg`/`formatter`/`workflow`/`file_read`/`file_write`) is fixed — doesn't fit one of those five, it can't be a step, full stop. A workflow *input* declared for it (a toggle) with a Python helper nobody calls is a real production bug: does nothing, dead code, no signal either way.

**Correct pattern: a custom API route, called from the frontend — never
spliced into `run_workflow`.**

```python
# application/api/main.py
from fastapi import APIRouter
from hutash_workflow.server import create_app

router = APIRouter()

@router.post("/my-processing")
async def my_processing(...):
    ...   # your subprocess call / library call / whatever doesn't fit a step
    return {...}

app = create_app(spec=SPEC, custom_routers=[router])
# Mounted under the same /api/v1 prefix, AFTER the framework's own routes.
# Real example: hutash-subs's `render` router does exactly this for its
# own ffmpeg-heavy caption burn-in, entirely outside the workflow system.
```

Call it from a widget or `action_button`, before or after `run_workflow`, never as a step inside it. Must it sit *between* two steps? No clean way exists today — run it first as its own request, hand the workflow the result as a `file`-typed input.

---

## Where to go from here

`application-format.md` — full schema, and (§3) hutash-subs annotated in
full, a complete built example bigger than any skeleton here. `widgets.md`
— every widget, every prop. `pipeline-format.md` — building a model
pipeline instead of an application.
