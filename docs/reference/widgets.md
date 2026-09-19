# Widget Registry

Every widget `hutash_vertical_ui` ships, what it does, and every prop it
reads. A vertical composes its whole UI from these by naming them in
`vertical.yaml` — it writes no React.

This file documents the CODE, not an intention. Each prop below is one a
widget actually reads, with the default it actually applies. If a prop
is not listed here, the widget ignores it.

Source of truth for the type names: `src/widgets/builtins.ts`.

---

## Quick reference

| Type | Purpose | Binds a value | Needs |
|---|---|---|---|
| `file_upload` | Drag-drop / click file picker | yes | workflow |
| `dropdown` | One-of-many selection | yes | workflow |
| `text_input` | Single or multi-line text | yes | workflow |
| `number_input` | Numeric field | yes | workflow |
| `slider` | Bounded numeric range | yes | workflow |
| `toggle` | On/off switch | yes | workflow |
| `model_selector` | Model picker filtered by capability | yes | workflow |
| `video_player` | HTML5 video + controls + styled caption overlay | no | media, workflow |
| `audio_player` | Waveform-led audio playback + transport | no | media |
| `output_player` | Plays a file the RUN produced | no | workflow |
| `source_switcher` | Chooses which cut the main player shows | no | media |
| `hanu_panel` | Hanu — context-aware chat that can run a workflow | no | workflow, media, navigation |
| `subtitle_list` | Editable timed-text list, auto-saving | no | media |
| `waveform_timeline` | Audio waveform with draggable cue regions | no | media |
| `media_meta` | Title, duration and date under a player | no | media, workflow, navigation |
| `style_picker` | Caption-style list; previews live on the player | yes | workflow, media |
| `section_header` | Title + optional action | no | — |
| `status_badge` | Coloured state pill | no | — |
| `progress_steps` | Step-by-step run progress | no | workflow |
| `project_grid` | Card grid of projects | no | workflow, navigation |
| `models_page` | Marketplace, grouped by capability | no | workflow, models |
| `setup_guard` | First-launch gate: install what the app needs | no | workflow, models |
| `settings_page` | Settings form from `settings:` | no | workflow |
| `action_button` | Primary action (run / navigate) | no | workflow |
| `export_controls` | Format picker, preview, save to the project | no | workflow |
| `toolbar` | Row of actions and separators | no | navigation |
| `sidebar` | Collapsing icon rail of destinations | no | navigation, autosave |
| `tabbed_panel` | Several panels of widgets in one column | no | — |
| `save_status` | Page-level save state | no | autosave |

"Needs" is the context a widget reads. A page carrying any media widget
gets `MediaProvider` automatically; `autosave` requires an open project.

`models` is `ModelStateContext` (`contexts/modelState.tsx`) — WHAT IS
INSTALLED, for the whole application, with one owner. It is also where
`workflow.models` comes from, so `settings_page`, `model_selector` and
`models_page` are reading ONE list even though two of them reach it
through the workflow context. That is the point: they used to hold three,
fetched at three different moments, and a model installed on the Models
page was still offered as "(not installed)" by a picker one tab away for
as long as the app stayed open. Never fetch `/models` in a widget.

---

## Control vocabulary

**These are not preferences. Pick by the SHAPE of the choice, not by
taste, so the same question always looks the same across every vertical.**

| The choice | Control | Why |
|---|---|---|
| Yes / no | `toggle` | One bit, one control. A two-option dropdown hides half the answer behind a click. |
| One of few (2–3) | card selection (`export_controls` format cards) | Few enough to show at once; seeing all the options IS the value. |
| One of many (4+) | `dropdown` | Too many to lay out without crowding the page. |
| An action | `action_button` (`variant`) | An action is a verb, not a value; it belongs on a button. |
| Several items for one action (a batch) | a checkbox per item, one `action_button` below | Each item is picked, the action is taken once; a toggle per item would read as a setting that takes effect on its own. |

**Yes/no → `toggle`.** Never a dropdown of Yes/No, never two radio
cards. The state is readable without opening anything.

**Several items for one action → checkboxes.** Added 2026-09-14 for the
first-launch gate's "Install selected": the user picks which capabilities to
install and then acts once. A `toggle` is a SETTING — flipping it changes
something by itself — so a row of toggles would promise installs that do not
happen until a button is pressed. A checkbox says "include this in what the
button does", which is exactly the shape. Items that are not optional are
shown checked and disabled rather than hidden, so the batch reads complete.

**One of few (2–3) → card selection.** The pattern
`export_controls` uses for SRT/VTT/TXT: a `role="radiogroup"` of cards,
all visible, the chosen one outlined. Use it when the options are few
AND the choice benefits from seeing them side by side.

There is now a shared primitive for it — `CardChoice`
(`src/widgets/cardChoice.tsx`) — added when `source_switcher` became the
second widget to need one, exactly as this section said it should be. It
is a PRIMITIVE, not a widget: no label, no state, no data source, so it
composes inside a widget that has those. `export_controls` still draws
its own, deliberately — its cards carry a format extension and a
promotion rule, and migrating the one widget that writes files is a
behaviour-preserving refactor worth doing on its own.

**One of many (4+) → `dropdown`.** Language pickers, model pickers,
format defaults. Below four options a dropdown wastes a click; above
three, cards crowd the page.

**Actions → `action_button` variants.**

| `variant` | For |
|---|---|
| `primary` (default) | The one thing this page is for. At most one per view. |
| `secondary` | A real alternative to the primary action. |
| `ghost` | Tertiary — present but not competing. |
| `danger` | Destructive and hard to undo. |

A page with two primary buttons has told the user nothing about which
one to press.

**There is no longer an exception here, and the story is worth keeping.**
`style_picker` used to claim one. The reasoning was that the table
assumes an option IS its label — "Hindi", "SRT", "whisper-small" are
fully conveyed by their names, so a list of names loses nothing, whereas
"Bold Pop" conveys nothing and the option must therefore render itself.
That reasoning was correct, and the conclusion drawn from it was not.

The option did not have to render itself. It had to be VISIBLE
somewhere, and the right somewhere was the video already on screen —
where the caption will actually sit, at the size it will be watched at.
Once the overlay previews the style live, the row is free to be a plain
name and the vocabulary applies unchanged.

The lesson generalises, so apply it before reaching for an exception: an
option whose label under-describes it does not need a richer CONTROL, it
needs the choice shown somewhere the user is already looking. Reach for
an exception only when there is no such surface, and change this table
first if you do.

---

## Input widgets

Every input widget takes `bind:` — the workflow input id it fills. The
value goes into the shared form state, which is how `action_button`
finds it without the page wiring anything together.

Widgets seed what they DISPLAY. A `<select>` shows its first option
whether or not the form holds a value, so a dropdown that displayed
"Same as source" while contributing nothing routed runs to the wrong
workflow. Every choice widget writes its displayed value into the form
on mount, and never overwrites a user's choice.

### `file_upload`

Drag-drop zone that is also a click target.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | `"Drop a file here"` | Zone text, and its accessible name |
| `accept` | string[] | — | MIME patterns, e.g. `[video/*, audio/*]` |
| `multiple` | boolean | `false` | Allow multi-select |

Binds a `File`. A run carrying one is sent as multipart automatically.

```yaml
- type: file_upload
  bind: media_file
  props:
    label: Drop your video or audio here
    accept: [video/*, audio/*]
```

### `dropdown`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `options` | `{value,label}[]` | — | Inline options |
| `options_from` | string | — | `workflow:<id>.inputs.<input>` or `api:<path>` |
| `prepend` | `{value,label}[]` | — | Fixed entries above whatever loads |
| `default` | string | — | Initial value |
| `filter_by` | `{field, from}` | — | Show only options whose `field` matches the input named by `from` |
| `describe_by` | string[] | — | Option fields to show after the label |

Options resolve from three sources so a vertical never restates a list
that already exists. A source that fails to load yields the prepended
entries alone — a picker offering less is usable; a page that refuses to
render is not.

An option keeps **every field its source declared**, not just `value` and
`label`. `GET /available-voices` answers with each voice's `language`,
`accent` and `gender` because a model's manifest declared them, and the
two props below are what a page does with them.

```yaml
- type: dropdown
  bind: target_language
  props:
    label: Subtitle language
    options_from: api:/available-languages
    prepend:
      - value: same
        label: Same as source (transcription only)
```

**`describe_by` — when the name is not the answer.** "Hindi" says
everything; "Fenrir" says nothing. Listed fields are appended after an em
dash, in the order given, and a field the option does not carry is
skipped rather than left as a dangling comma.

**`filter_by` — two questions whose answers are not independent.** The
dubbing form asks which language to dub into and which voice speaks it.
Offering every installed voice for every language is not untidiness, it
is a silent failure: a voice selects the phonemizer, so an English voice
given Hindi text returns a well-formed WAV of a fluent-sounding voice
saying no words. The run succeeds, nothing warns, and only a listener can
tell. The filter is the last place that is preventable before it becomes
audio.

```yaml
- type: dropdown
  bind: voice
  props:
    label: Voice
    options_from: api:/available-voices
    filter_by:
      field: language      # the field on each option
      from: target_language # the input holding the value to match
    describe_by: [accent, gender]
```

Two deliberate tolerances, both learned from what breaks otherwise:

- An option that does **not declare** the field is kept, never hidden. A
  manifest that predates the field should not vanish from the picker the
  moment a newer one beside it declares it.
- A held value the filter no longer offers is **replaced** with one it
  does. A `<select>` displays its first option whether or not the parent
  holds it, so leaving the stale value behind produces a form that looks
  correctly filled in while the run receives something the user never
  saw — the same invisible mismatch the seeding rule already guards.

### `text_input`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `placeholder` | string | — | Empty-state hint |
| `multiline` | boolean | `false` | Render a textarea |
| `max_length` | number | — | Character cap |
| `default` | string | — | Initial value |
| `autofill_from` | string | — | Seed from another input's filename |

`autofill_from: media_file` turns `my_holiday-clip.mp4` into
`my holiday clip`. It is an OFFER: it fills an empty, untouched field
only, so a typed name survives swapping the file.

### `number_input`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `min` / `max` | number | — | Bounds |
| `default` | number | — | Initial value |

Use when the exact figure matters more than its position in a range.
For a bounded value the user tunes by feel, use `slider`.

### `slider`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `min` | number | `0` | Lower bound |
| `max` | number | `100` | Upper bound |
| `step` | number | `1` | Increment |
| `default` | number | `min` | Initial value |

Shows its current value beside the track — a slider you cannot read a
number off is a guess.

### `toggle`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `default` | boolean | `false` | Initial state |

A `role="switch"` button. The control for every yes/no — see Control
vocabulary.

### `model_selector`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | — | Field label |
| `capability` | string | — | Filters to models declaring it (`stt`, `translate`) |

Lists models from the engine, disabling any not installed and marking
them so. Empty value means "Automatic" — the app resolves a model at run
time.

**Nothing installed for the capability yet**, this becomes one of two
fallbacks instead of a picker with nothing pickable in it:

- **No model at all declares the capability** — a single "No model
  installed — Install from Marketplace" button, navigating to the Models
  page (there is genuinely nothing to offer inline).
- **At least one uninstalled model exists** — one row per available
  model (name + an Install button), install triggered right there. Uses
  the SAME `startInstall`/`installs[id]` shared state the Models page
  renders from (`useModelState()`) — not a second install path, so a run
  started here shows on the Models page too, and vice versa. A row shows
  live progress (`InstallState.phase === "installing"`) or an error with
  Retry (`"error"`), the same three phases the Models page's own cards
  use. Falls back to the populated `<select>` automatically the moment
  the engine reports the capability has an installed model — no
  navigation, no remount.

---

## Media widgets

These share `MediaContext`: playback position and the segment list. That
is what lets a player, a transcript and a waveform cooperate without
knowing about each other — the page's YAML decides which are present.

### `video_player`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `show_subtitles` | boolean | `false` | Overlay the active cue on the frame |
| `subtitle_style` | string | `classic` | Caption style the overlay opens in |
| `style_input` | string | `"style_id"` | Workflow input holding the chosen style |
| `sources` | `{id,label,output?}[]` | `[]` | Other cuts of this project the player can show |
| `autoplay` | boolean | `false` | Play on load |
| `loop` | boolean | `false` | Loop playback |

Plays the media `File` from the form as an object URL — the file never
leaves the browser for playback.

**`sources` — other cuts of the same project.** A dubbing vertical's
deliverable is the mixed MP4 the run produced, and judging it means
watching it where the user was already watching, at the size it will be
watched:

```yaml
- type: video_player
  props:
    sources:
      - id: original
        label: Original
      - id: dubbed
        label: Dubbed
        output: dubbed_video
```

An entry naming an `output` is fetched from the run through the client
(bearer token in a header, never a query string) and played from an
object URL; an entry without one IS the uploaded file. Declaring no
`sources` is the normal case and changes nothing.

The player OWNS this list and publishes it to MediaContext;
`source_switcher` renders whatever is there. So the options are declared
once, and a switcher placed in a side panel needs no copy of them.

**This is not the case `output_player` refuses, and the difference is
the media rather than a preference.** That widget keeps a generated
voice-over off MediaContext because it is a different recording on its
own clock. A dubbed video is the same picture with a different audio
track — `replace_audio` runs `-c:v copy`, so the frames and their
timestamps are the original's, which are the timestamps the segments
were timed against. Sharing one clock is correct here and wrong there.
They diverge only past the last frame: the mix deliberately does not
truncate, so a dub running longer than the footage makes the file
longer, beyond anything there is to stay in sync with.

A named cut that cannot be fetched says so where the picture would be.
It does NOT fall back to the original — showing someone the source video
and calling it the dub is worse than showing them nothing.

THE CAPTION OVERLAY IS THE STYLE PREVIEW. Choosing a style in
`style_picker` swaps a class on the overlay and nothing else happens:
same words, same frame, same moment, different look, instantly. Nothing
is wired between the two widgets — the style id travels in MediaContext,
so a picker in a side panel restyles the captions on the real video at
the size they will be watched at.

The style is resolved in falling order of authority: the live choice in
MediaContext, then the bound workflow input named by `style_input`, then
`subtitle_style`. The input matters because `style_picker` usually lives
in a tab, and only the active tab is rendered — reading the input
directly is what puts a style chosen yesterday on the captions at first
paint, rather than only once the user opens the tab where they chose it.

Six styles are drawn: `bold_pop`, `word_center` (one word at a time),
`karaoke` (whole line, spoken word lit), `two_line`, `minimal` and
`classic`. An unknown id falls back to `classic` rather than vanishing.
A style whose mode is word-by-word falls back to the whole line when the
segments carry no word timings.

These are APPROXIMATIONS of what ffmpeg bakes in — CSS and libass do not
draw an outline identically, and they never will. What the preview DOES
promise is the geometry: the same size, the same wrap width, and so the
same number of lines.

**Everything is measured against the PICTURE, never the element.** The
video element is `object-fit: contain` inside a 16:9 stage, so the two
are the same rectangle only for a 16:9 video — a 9:16 phone clip is
pillarboxed, a 2.39:1 film letterboxed, and those bars belong to the
element and to no frame ffmpeg will ever burn into. `useVideoPictureBox`
measures the real picture from the element's own `videoWidth`/
`videoHeight`, and `.hv-video__frame` covers exactly it. Every
percentage the overlay uses — the caption band's bottom, the wrap
width, `word_center`'s `top: 50%` — then resolves against the frame
rather than the stage, with none of those rules changing.

**Font size is the renderer's own formula.** Every ASS style asks for
its size through `layout.scaled(base)` = `round(base * frameHeight /
1080)`, so a caption is a fixed fraction of frame height at any
resolution. `.hv-video__frame--measured` multiplies each style's
authored base (bold_pop's 84, karaoke's 56, …) by
`--caption-frame-scale`, the displayed picture height over 1080. A
1920-wide video shown at 960 gets exactly half the ASS size.

That is why this stopped being a guess: the old sizes were a fraction of
the STAGE width, correct for a 16:9 video and wrong for every other
shape — a 2.39:1 film previewed 34% too large and wrapped onto three
lines where the burn-in used two.

The `--caption-size-*` values in `tokens.css` remain, as the fallback
for the moment before a video reports its dimensions. They carry the
16:9 assumption and say so.

`captionSizes.test.ts` reads the vertical's own style modules and fails
if a base here parts company with the one there, so "keep them in step"
is checked rather than remembered.

**An ASS `Fontsize` is not a CSS `font-size`, and the gap is 12%.**
libass follows VSFilter: the number sets the text's ascent plus descent,
where CSS sets the em square. For a face whose ascent and descent overrun
its em — Arial's do — the same number draws SMALLER in libass than in a
browser, so a preview using the ASS number directly drew every caption a
tenth too large and wrapped it a word early. `LIBASS_FONTSIZE_TO_CSS` is
that conversion, MEASURED against real burns rather than derived from the
browser's own metrics (which are integers rounded per size, and the wrong
metric pair besides). `captionScale.test.ts` records the measurement.

**The wrap width and the band position are the RENDERER's, asked for.**
They were two hardcoded percentages — 80% wide, 16.7% up — and both were
the 16:9 figures. The renderer wraps at its layout's `max_width / width`
(72.9% on 16:9, 83.3% on 9:16 and 1:1) and puts the band at
`(height - subtitle_y) / height` (27.1% on a vertical preset). A caption
a tenth of the frame from where the preview showed it, on exactly the
videos these styles were designed for. `useCaptionLayout` reads them from
`GET /caption-layout`, which runs the same resolver the render pipeline
runs; `captionLayout.ts` says what is still not identical afterwards.

**A caption block is inline text, not a flex row.** The word spans used
to be flex items with a `gap`, which wraps GREEDILY between items and
leaves `text-wrap: balance` nothing to act on — balance being a property
of text. libass's `WrapStyle: 0` breaks lines evenly, so the two
disagreed by construction. As inline text with real spaces between the
spans, balance applies and the separator is a space glyph in the
caption's own font rather than a fraction of an em.

Measured end to end — captions burned with ffmpeg's `ass` filter, the
same strings measured in a real browser — agreement with the renderer's
line count went from 5/8 to 7/8 across two aspect ratios. The remaining
one is a caption whose text is within about 5% of the box width, where
the measurement's own ±1.5% decides the break. See
`hutash-subs/scripts/measure_caption_wrap.py` and
`hutash-os/scripts/measure-caption-wrap.mjs`.

### `audio_player`

Playback for media with nothing to look at — a podcast, a voice
recording, a music track. The waveform is the anchor of the panel and
the transport sits under it; there is no video element and so no black
rectangle where the content should be.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `show_waveform` | boolean | `true` | Draw the waveform above the transport |
| `show_volume` | boolean | `true` | Mute toggle + volume slider |
| `height` | number | `120` | Waveform height in px, passed through |
| `show_segments` | boolean | `true` | Lay transcript cues over the waveform |
| `draggable` | boolean | `false` | Allow dragging a cue edge on the waveform |

**The waveform is `waveform_timeline`, not a second drawing of one.**
This widget renders that one and owns only the `<audio>` element that
actually plays, attached to MediaContext so the transport, the waveform
and the subtitle list all read one clock. A copy would be a fork of the
catalogue inside the catalogue.

`draggable` defaults to `false` here and `true` there, and that is the
difference between the two: retiming a cue is a subtitle-editing
gesture, so place `waveform_timeline` directly when editing timing is
the point, and `audio_player` when playing is.

`show_waveform` used to default to `false` and drew a plain position
bar. Both are gone: an audio player whose purpose is to give audio
something to look at should not have to be asked for it.

### `output_player`

Plays a file the RUN produced — a generated voice-over, a dubbed video —
rather than the file the user uploaded.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `output` | string | first declared file output | Which workflow output to play |
| `label` | string | `"Result"` | Heading above the player |
| `media` | string | inferred | `audio` or `video`; inferred from the format otherwise |

**This is not a prop on `audio_player`, and the reason is the whole
widget.** That one owns the single `<audio>` attached to MediaContext so
the transport, the waveform and the subtitle list read ONE clock. A
generated voice-over is a different recording on a different clock:
attaching it there would highlight the transcript against the dub while
the video beside it plays the original, and whichever widget mounted
last would win. The ABSENCE of MediaContext is what this widget is for,
so it cannot be a flag on one built around it.

It has no design of its own — every class is the audio player's, so a
result plays the same way in every vertical.

**How the file is reached.** A workflow's file output carries `run_id`
and `file`, and the backend serves it at `/runs/{run_id}/files/{file}`
(`hutash_workflow`'s run endpoint). The widget FETCHES it through the
client, which carries the bearer token in a header, and plays it from an
object URL — a `src` could only have carried the token in a query
string, where it lands in history and logs.

A text output is never a candidate: without a `file` there is nothing to
fetch, so a workflow emitting an SRT alongside a voice-over plays the
voice-over. Before any run it says "Nothing generated yet." rather than
drawing a transport that does nothing, and a run whose files are gone —
they do not survive a restart — says so.

### `source_switcher`

Chooses which of `video_player`'s declared `sources` the main player is
showing.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | `"Showing"` | Legend above the cards |

**It declares nothing.** The options come from the player's own
`sources:` block, published into MediaContext by the widget that owns
them. Restating them here would put one fact in two places in a single
YAML file, and they would disagree the first time either moved.

The same shape `style_picker` already uses: a control in a side panel
changing what the video in the middle shows, with nothing wired between
the two. Place both and it works; place neither and the player shows the
uploaded file.

Card selection, per the control vocabulary — two options, both worth
seeing at once, the current one readable without opening anything. Fewer
than two cuts renders a message rather than a one-card control: before
the run there is nothing to switch to, and that is a normal state.

### `hanu_panel`

**Hanu**, the assistant: chat that can see what the app has open, and
can act on it. The widget is named for the assistant because the name is
what a user sees in the rail, and a type called `assistant_panel` beside
a label saying "Hanu" is one more thing for the next reader to reconcile.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | `"Hanu"` | Accessible name for the panel |
| `placeholder` | string | `"Ask about this project…"` | Composer hint |
| `empty_text` | string | see below | Shown before the first question |

**The context is READ, never passed in.** The open project comes from the
workflow inputs, the transcript size from `MediaContext`, the current
page from `NavigationContext`. A widget that had to be told what the user
was doing would be told wrongly the first time a page was added, and a
model grounded in the wrong project answers fluently and incorrectly with
nothing on screen to contradict it. A badge at the top names what it can
see for exactly that reason — `Viewing: V01 kokoro v2 · editor page · 9
segments` — because an assistant whose context is invisible is one the
user cannot reason about.

**The prompt is built on the BACKEND**, not here. The system prompt names
the workflows this app can run, and that list already exists server-side.
Assembling it in the widget would be a second copy of the app's own
capability list, drifting the first time a workflow was renamed. So the
widget sends facts and `POST /hanu/chat` turns them into
instructions.

**The action button is the point.** An assistant that can only say "you
could translate those subtitles" has moved the work, not done it. When
the model names a workflow the app really has, the reply renders a button
that starts it through the SAME `run` every other control uses — so a run
begun from chat is indistinguishable from one begun from a form. A named
workflow the app does NOT have is dropped server-side rather than
rendered: a button that cannot work is worse than no button, because it
looks like a feature.

Remembers the conversation for the SESSION, not across one. Every send
also ships the last `HANU_HISTORY_LIMIT` (12) messages as `history`,
which the backend folds into the system prompt so a follow-up like "make
it shorter" has an antecedent for "it" — see `hanu.tsx` and
`hanu.py`'s `build_system_prompt` for the bounded window and why it
stays bounded (the underlying capability is one `prompt`/`system` pair,
not a message array, so remembering means re-stating recent turns as
text on every call). Nothing is written to disk or to the project:
`messages` starts at `[]` again on a reload, same as before this
existed. This is a deliberate difference from Studio's own Hanu, which
is still memory-less. A failed turn is reported BESIDE the transcript,
never inside it — the model did not say it, and a transcript is a record
of what was said.

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

An overlay, in every vertical that has it: the assistant answers about
the work in front of you, so leaving that work to ask would be exactly
backwards — and an overlay never unmounts the editor or loses an edit.

### `media_meta`

What the media IS, under the player that plays it — title, run length,
creation date. Matters most in the case a player cannot help with: audio,
where there is no frame to recognise the file by.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `show_duration` | boolean | `true` | Show the run length |
| `show_date` | boolean | `true` | Show the creation date |
| `fallback_title` | string | — | Shown when no project is open |

The three facts come from three owners and none of them is this widget:
the name and date are the PROJECT's (same client call `project_grid`
makes), the duration is MediaContext's. Nothing is stored here, so
nothing here can disagree with the page around it.

A missing fact is OMITTED, not rendered as a dash or a zero. "0:00"
beside a file still decoding is a wrong answer; an absent row is an
honest one. With no title and no facts the widget renders nothing at all.

The date uses the reader's own locale. A date is read, not parsed.

### `subtitle_list`

Editable timed-text. Click a timestamp to seek, click text to edit.
Enter commits and moves down; Tab commits and jumps to the other field;
Escape cancels.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `editable` | boolean | `true` | Allow inline editing |
| `show_timestamps` | boolean | `true` | Show the time column |
| `click_to_seek` | boolean | `true` | Clicking a timestamp seeks |
| `follow_playback` | boolean | `false` | Auto-scroll to the playing cue |
| `auto_save` | boolean | `true` when editable | Persist edits |
| `save_path` | string | `transcripts/subtitles.srt` | Where in the project |
| `style_preview` | boolean | `true` | Draw cues in the chosen caption style |
| `style_input` | string | `style_id` | Workflow input holding the style |
| `save_endpoint` | string | — | A vertical route that serializes the segments itself |
| `working_path` | string | — | The file that route writes, read back before `save_path` |
| `platform_input` | string | `platform` | Workflow input sent with a `save_endpoint` save |
| `empty_action` | object | — | An `action_button` declaration shown under "No subtitles yet." |

**An empty list can offer the way out.** A project created while no model
was installed is saved with its media and no transcript; reopening it
showed "No subtitles yet." and nothing anywhere could produce one.
`empty_action` takes an ordinary `action_button` props block — usually
`run_workflow` with `run_on_project: true` — and renders it with that
widget, so it follows the same run and disabled rules as every other
button. It appears only while the list is empty.

**A vertical may own its own save format.** The catalogue's write path
takes TEXT: a widget encodes its content and the project stores the
bytes. That is right for SRT and VTT and wrong for anything that has to
be GENERATED — an ASS document in a caption style is drawn by a catalogue
of styles the vertical owns, and a copy of that here would be a second
design of the same looks.

So `save_endpoint` hands the route the segments themselves — words
intact, which is the whole reason — plus `style_id` and `platform` and
the SRT the widget encoded (as `subtitle_text`, for a route that wants
it). `{project_id}` in the path is substituted, so the vertical decides
its own route shape. `working_path` names the file to read back, and it
is tried BEFORE `save_path`.

Why it exists: SRT can say two things about a cue, when it starts and
when it ends. A project reopened from one lost its per-word timings and
its style, so every animated caption style collapsed to one static cue
per line — real exports, each showing a style's layout with none of its
motion. `parseAssWorkingFile` reads the format back (see `timedtext.ts`);
writing it stays with the vertical.

Fully backward compatible. With neither prop the widget saves text to
`save_path` exactly as before, and with them, a missing, unparseable or
foreign working file falls back to `save_path` — which is what opens
every project made before the format existed. An `onSave` from a React
host still outranks both.

**The cues are drawn in the chosen caption style.** A caption style is a
property of the SUBTITLES, not of the player showing them, so this list
and `video_player` resolve it the same way — through the one
`useCaptionStyleId` hook, three deep: the live choice in MediaContext,
then the bound workflow input (which is what a reopened project
restores), then the widget's own `subtitle_style`. Before this the list
rendered plain text whatever was picked, which made the style read as a
setting owned by the Style tab.

The LOOK only — weight, letter case, `classic`'s plate — via the shared
`.hv-caption--<id>` classes the overlay also wears. Not the size and not
the outline: both need a 16:9 frame with a picture behind them, and a
side panel is neither. The row being EDITED is never styled; typing into
an uppercased, outlined field is worse than typing into a plain one.

`style_preview: false` opts a list out.

**Auto-save is the widget's own behaviour.** Edits debounce for 2s, then
write to the open project, and the widget shows Saving… / Saved / Not
saved beside its own content. It does nothing at all when there is no
project — staying quiet beats inventing a destination.

The "Saved" confirmation is a FLASH, and it is owned by `useSavedFlash`
in `contexts/autosave.tsx` — never by a local `useState(false)` in the
widget. The obvious boolean has a hole that the two constants make
structural rather than unlucky: the debounce and the indicator are both
2000ms, so an edit made right after a save lands the next write at the
exact moment the flash is due to clear. Batched, `false` then `true`
leaves the boolean unchanged, so the `[saved]` effect never re-runs,
nothing re-arms the timeout, and "Saved" stays up for the rest of the
session — over an unsaved edit, and later over a failing save. The hook
flashes a monotonic id instead, so every success is a new value and
always re-arms. Both indicators (this widget and the toolbar's
`action: save`) go through it.

`follow_playback` never scrolls while a row is being edited: moving the
list out from under someone mid-sentence is worse than letting the
highlight drift.

A React host may pass `onSave(content, path)` to take over saving
entirely; the YAML path leaves it out.

### `waveform_timeline`

Audio waveform (wavesurfer.js) with each cue laid over it as a region.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `show_segments` | boolean | `true` | Draw cue regions |
| `draggable` | boolean | `true` | Resize a region to retime its cue |
| `click_to_seek` | boolean | `true` | Click seeks the video |
| `height` | number | `80` | Waveform height in px (wavesurfer's own unit) |

Dragging a region edge writes the new timing back to the shared
segments, so the subtitle list updates with it. Segments with **no text
get no region** — a gap is bare waveform, and drawing a region over it
would claim a cue is there.

Degrades to a placeholder wherever Web Audio or a canvas is unavailable
rather than throwing; that state is normal before a file is chosen.

---

### `style_picker`

A LIST OF NAMES. Clicking one restyles the caption overlay on the
player instantly; hovering previews without committing, and leaving the
row puts the chosen style back. `bind:` names the workflow input the
chosen style id is written to.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `styles_from` | string | `/styles` | Endpoint listing the styles (`api:` prefix optional) |
| `default` | string | first style | Style selected before the user picks |

No preview button, no preview card, no rendered clip, and nothing to get
back FROM. This widget used to be a gallery of cards that each rendered
a few seconds of the user's own footage through ffmpeg. The argument for
it was sound — a look cannot be read off its name — but it answered that
in the most expensive way available: minutes per style, a second video
element to play the result in, a "Back to video" button to escape it,
and only one style visible at a time. The overlay answers the same
objection for free. The name is not what conveys the look; the video
underneath it is.

So the **Control vocabulary** exception this widget used to claim is
WITHDRAWN. It is a radiogroup of options whose members are their labels,
because the preview no longer lives in the option — it lives on the
stage where the caption will actually sit.

Keyboard focus previews as hover does. Without it, tabbing the list
moves the selection ring and nothing else, and a keyboard user picks a
look they were never shown.

The chosen style is NOT cleared when this widget unmounts. It usually
lives in a tab, and only the active tab is rendered; clearing on unmount
would snap the captions back to the default the moment the user returned
to the transcript. The style belongs to the project, not to the panel
that set it.

The style is still rendered by ffmpeg — at EXPORT, once, on the style
the user settled on, which is the only time a burned-in render is worth
minutes of anyone's time. `POST /preview-style` still exists on the
vertical for other callers; this widget no longer calls it.

---

## Display and layout widgets

### `section_header`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `title` | string | `""` | Heading |
| `action_label` | string | — | Optional action; omit for a plain heading |
| plus any `action_button` action props | | | What the action does |

### `status_badge`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `status` | string | `"unknown"` | Mapped to a state |
| `label` | string | the status | Text shown |

States: `success`, `warning`, `error`, `unknown`. `installed`,
`complete`, `compatible`, `ready` → success; `running`, `degraded`,
`pending`, `progress` → warning; `failed`, `error`, `incompatible` →
error. **`unknown` is a real state** — something the engine has not
classified — not a placeholder to colour green.

### `progress_steps`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `visible_during` | string | — | `workflow_run` hides it until a run starts |

Collapses repeated events for one step onto that step's own row, so a
step updates in place instead of appending.

### `toolbar`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `items` | ToolbarItem[] | `[]` | Buttons and separators, in order |

Each item: `label`, `icon`, `tooltip`, `action`, `to`, or
`separator: true`. Supported actions: `navigate`, `run_workflow`, `save`.

`action: save` renders a stateful control — Saving… / Saved ✓ / Save —
that flushes pending work immediately. It disables itself when nothing
is pending (a Save button that writes an unchanged file teaches the user
that pressing it means nothing) and renders nothing when there is
nowhere to save.

### `sidebar`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `items` | SidebarItem[] | `[]` | Destinations, in order |
| `label` | string | `"Sections"` | Accessible name for the rail |
| `expanded` | boolean | `false` | Start pinned open |

Each item: `label`, `icon`, `action` (`navigate`, the default, or
`save`), and `to` for a navigate. `icon` names one of the rail's drawn
glyphs — `folder`, `marketplace`, `library`, `grid`, `gear`, `disk`,
`mic`, `doc`, `list`, `save_to_disk`, `chevron_left` — never an emoji:
an emoji renders differently on every platform and cannot take the
rail's colour, and a rail is the one place every icon must look like it
belongs to one set. An unknown name falls back to `grid` rather than
leaving a hole in the row.

**The set is Studio's.** The paths are copied verbatim from
`hutash-studio/src/ui/components/Sidebar/icons.tsx` (and
`ProjectSwitcher.tsx` for the folder), and so is the treatment: a 17px
glyph on a 24-unit viewBox, stroke 1.6 with round caps and joins,
colour inherited, centred in a fixed `--rail-icon-box` (1.125rem). The box is why a wide
glyph and a narrow one leave the labels lined up, and why the glyph is a
pixel smaller than the space it sits in. One product should not draw the
same idea two ways.

COLLAPSED is the resting state. It expands on hover, and the toggle at
its top pins it open; a rail's job is to give the content the width, and
a label is only needed while you are deciding where to go. The label is
the accessible name whether or not it is painted, so a collapsed rail is
not a row of mysteries to a screen reader.

It navigates; it does not own state. Every `to` is a page id the
navigation already resolves, so a mistyped one is the same broken link
it would be anywhere else. `action: save` reaches the same autosave
controller `save_status` reports on — the rail and the indicator can
never disagree about whether a save happened — and disables itself when
there is no project to save into.

**A page carrying a sidebar loses the top bar's page buttons**, keeping
only the breadcrumb. The rail already names those destinations, and two
controls answering one question an inch apart is not a convenience: it
is two designs, and they drift. Derived from the page, so a vertical
says where its rail goes once.

Give the column `left_width: auto` on the page. A fixed width clips the
labels when the rail expands instead of letting the content give up the
width for as long as they are on screen.

### `tabbed_panel`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `tabs` | Tab[] | `[]` | `{id, label, widgets}`, in order |
| `default_tab` | string | first tab | Tab shown on open |
| `label` | string | `"Panels"` | Accessible name for the tab row |

A CONTAINER: each tab holds widgets resolved through this same
catalogue, `customWidgets` overrides included. Renders nothing when it
declares no tabs.

Reach for it instead of one overlay per panel when the panels are about
the thing on screen. An overlay covers its own subject — a caption style
you cannot see against the video while you pick it is a style you pick
twice, and an export panel over the transcript hides what is being
exported. Tabs put the choice beside the subject. An overlay is still
right for a panel that is NOT about the page underneath it: Settings and
Models are the same wherever you opened them from.

ONLY THE ACTIVE TAB IS RENDERED. Not hidden — absent. An inactive tab
has no element in the DOM at all, so no stylesheet can put it back on
screen.

This is worth stating plainly because the obvious alternative was tried
first and shipped broken: `hidden` on every inactive panel, which is the
correct HTML and did not work. `hidden` hides through the User-Agent rule
`[hidden] { display: none }`, and this widget's own
`.hv-tabs__panel { display: flex }` beat it on ORIGIN — an author rule
wins over the UA sheet whatever the specificity. Every panel rendered at
once while the attribute sat there looking correct. Do not reintroduce
`hidden`, or a `display` rule, or a `.is-active` class: any of them can
be overridden by a stylesheet, and the failure is silent.

What unmounting costs was checked rather than assumed. A tab loses its
widgets' own LOCAL state — the export panel's chosen format and its
"Saved to…" confirmation, a style preview thumbnail. It does not lose the
work: a transcript's edits live in MediaContext at PAGE level, above this
widget, so they survive a tab switch, and a pending debounced save is
flushed on unmount by `useDebouncedSave`. If you add a tab whose state
must outlive a switch, lift that state to the page — do not make the
panel persist.

Pinned by `src/widgets/shell.test.tsx` in this package, which asserts
ABSENCE rather than any attribute or class. That distinction is the whole
lesson: the earlier test asserted the `hidden` attribute, jsdom does not
compute the cascade, and so the test passed for the entire time the
widget was visibly broken.

### Leaving a page with unsaved work

`confirm_leave:` on a page shows **exactly one** dialog, and it never
changes shape while it is up.

- Nothing unsaved → no dialog. The guard checks first and navigates
  silently. A prompt on a departure that loses nothing is a warning
  users learn to dismiss without reading, which costs the work it exists
  to protect.
- Unsaved work, project open → **Save and leave / Leave without saving /
  Cancel**. Save and leave navigates only if the write succeeded; the
  dialog stays up showing the error if it did not.
- Unsaved work, no project → the warning and **Leave / Cancel**. There
  is no third button because there is nowhere to put the work, and a
  button that quietly does nothing is worse than one that is absent.

**The three buttons are one row, always.** Same height
(`--control-height-lg`), equal gaps, and they shrink and ellipsize
together rather than wrapping — a dialog asking one question with three
answers reads as one question only while the answers are side by side.
`--accent` carries the one action that keeps the work; the two ways of
losing it sit on `--bg-3` and read as the same weight as each other,
because they are. Measured in Chromium, not asserted in jsdom: jsdom
computes no cascade, so a wrapped row or a taller button passes every
component assertion. The numbers are in `vertical-ui.css` beside the
rules that produced them.

Two things keep it to one dialog. The auto-save debounce is HELD while
it is open (`AutosaveContextValue.paused`) — the pending content stays
registered, so "Save and leave" still writes what is on screen, but no
timer fires underneath the question. And the dialog freezes its decision
when it opens, so a write already in flight when the click landed cannot
reshape it. Without both, a timer landing mid-decision turned three
buttons into two and rewrote the message, which reads as a second popup
appearing — worse than two dialogs, because the button the user was
reaching for moved.

### `save_status`

No props. Page-level save state, for a page that edits without a widget
reporting its own. Stands down automatically when a widget is already
reporting, so it never doubles up.

---

## Page-scale widgets

### `project_grid`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `opens` | string | `"editor"` | Page a card opens |
| `create_opens` | string | `"new_project"` | Page the create card opens |
| `create_button` | boolean | `true` | Show the create card |
| `show_thumbnail` | boolean | `true` | Show the thumbnail area |
| `show_status` | boolean | `true` | Show date and status |

Responsive by default: `repeat(auto-fill, minmax(18rem, 1fr))`. The
widget owns that, so no app declares columns.

Set `create_button: false` when a `section_header` already offers "New
Project" — two controls with the same label is the duplication to avoid.
The empty state defers its own action for the same reason.

Cards navigate with `{projectId, projectName}`, which is what gives the
editor a save destination.

**Right-click a card for "Remove from list."** No prop — every grid has
it, because a list you cannot take anything off is not a list. The action
calls `DELETE /projects/{id}`, which unlinks the `.hsprj` marker and
nothing else: the folder, the media and every export stay on disk. A
listing only shows folders that carry a marker, so removing it is what
makes the project disappear.

Deliberately no confirm dialog. Nothing is destroyed, and a prompt would
say otherwise — the menu carries "Files stay on disk" instead, which is
the fact the user actually needs. The menu is portalled to `<body>` so a
card's own overflow cannot clip it, and closes on outside click, Escape
or scroll.

### `models_page`

The marketplace, grouped by the capabilities the manifest declares. Its
card is Studio's marketplace card, deliberately: same anatomy (icon |
body | status | action), same badge treatment, same compatibility call.

**It opens in the WIDE overlay, and Settings opens in the standard one.**
`withGeneratedPages` gives the Models page `overlay_size: lg` (50rem) and
the Settings page the default `md` (35rem), so every vertical gets the
same two widths without declaring anything. The difference is deliberate
and is about what each page holds: a marketplace row carries an icon, a
name, a description, a wrapping meta line and an action side by side and
reads badly squeezed, while a settings form is label/control rows that
read worse the wider they get. Measured at `md` a card is 510px against
656px at `lg`.

A vertical that declares its own Models page — Subs and Dub both do, for
`capability_labels` — should keep `overlay_size: lg` on it so it matches
the generated one. Leaving it off is what made the same page 35rem in one
app and 50rem in another.

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `capability_labels` | object | `{}` | Human name per capability id |

**A picker's capability is a section too.** `GET /models` serves a group for
each capability `capabilities_needed` declares AND each capability a
`model_selector` in `settings:` names, so Hanu's Assistant picker
(`capability: llm`) always has options, and its "Open Models" lands on a page
with a section to install one. A language model's capability is `llm` — there
is no `chat` pseudo-capability. Mark `llm` `optional: true` under `setup:` when
the rest of the app works without a language model.

**Compatibility follows Studio exactly.** Compatible is a bare lit dot —
there is nothing to warn about, so it needs no words. Only `degraded`
and `incompatible` earn a pill, tinted from the status tokens at Studio's
own 10% fill / 30% border / full-strength text.

| State | Token |
|---|---|
| compatible | `--success` (dot + glow) |
| degraded | `--warning` (pill) |
| incompatible | `--error` (pill) |

The card reads these from `GET /models`:

| Field | Shown as |
|---|---|
| `compatibility` | the dot or the pill above |
| `compatibility_note` | its tooltip — the REASON, e.g. "Needs 8GB, your GPU has 3.6GB usable" |
| `hardware` | a pill: what the CATALOGUE says the model needs ("GPU (4GB+)") |
| `license` | a pill |
| `size_mb` | GB/MB, absent when unknown |
| `rating` | the score star, 0-10 |
| `languages` | the card's TITLE and a language fact in the meta row — see below |

**Named by language, not by package id.** A model's manifest declares
`languages` — ISO 639 codes, or `{source, target}` for translation — and the
engine's catalogue relays it. For a capability chosen by language (`stt`,
`translate`, `tts`, `voice-clone`) the card is titled "Hindi Translation" or
"English Transcription" (`modelTitle` in `src/languages.ts`), with the model's
own name beneath; the meta row adds "English → Hindi". A model declaring no
languages, or a capability not chosen by language (chat, image), keeps its
name as the title.

**Which model the app runs.** When two or more models for a capability are
INSTALLED and the vertical declares a `model_selector` setting for it, the
group shows a "Used by this app" chooser — Automatic plus each installed model
— that writes that setting, the one the run resolves from. It is
`ModelChooser` (`src/widgets/modelChooser.tsx`): radio cards for 2–3 options,
a dropdown for 4+, each option showing size, score, languages and a
compatibility warning.

`hardware` and `compatibility` are both shown on purpose: one is a claim
about the MODEL, the other a verdict about this MACHINE, and either alone
leaves the user unable to tell an unsupported model from an unsupported
computer. A tooltip repeating the badge's own word would be the same word
twice, so the note is what it carries when there is one.

**Score** renders a filled star in `--color-warning` plus the value to
one decimal, matching Studio's spec row. It renders NOTHING when there
is no score, where Studio shows an em dash: Studio's card is a fixed
spec grid whose Score column must hold something, this one is a flowing
meta row, and a dash there would be a fact the app does not have dressed
as one it does. The engine's `/catalogue` now relays index.json's own
`quality_score`, scaled to this card's 0-10 scale by the backend rather
than by the widget — which scale a particular backend publishes on is not
something a component should have to know.

Filtered to the vertical's `capabilities_needed` and grouped by them.
Install and remove proxy to the engine and then RE-READ the list: the
engine owns install state, and a local guess is wrong the moment an
install fails or another app installs the same model.

**The install lifecycle is Studio's, step for step.** An install is
ASYNCHRONOUS — `POST /packages/{id}/install` returns 202 the moment the
job is queued, and the download runs for minutes afterwards. So the POST
returning is not the install finishing, and nothing on the card is
inferred from it. Everything comes from the engine's own state machine,
polled through `/models/{id}/install-progress` (a proxy of
`GET /packages/{id}/status`, the same snapshot Studio polls):

| Engine state | The card |
|---|---|
| `queued`, `fetch_package`, `installing`, `starting`, `health_check` | a bar, "Installing 14%", the engine's own live line, and Cancel |
| `waiting_for_vram` | the same — the job is queued, not stalled |
| `ready` | "Finishing…" until the re-read confirms it |
| `failed` | the engine's REASON, a Retry and a Dismiss |
| `cancelled` / `removed` | nothing; the list's own answer takes over |

"Installed" is only ever the ENGINE's answer, read back from
`GET /models`. There is no installed phase in the widget's own state:
one would be a second copy of install state, and the copy is wrong the
moment an install fails.

A null percentage renders an indeterminate bar — never a fabricated
figure. A reported percentage never decreases, because a retried step
re-reports from its own zero and a bar that walks backwards reads as a
failure.

**Retry re-installs.** The engine resumes from the step that failed
rather than starting the download again, which is why the retry and the
install are one call. **Cancel reaches the engine** (`POST
/packages/{id}/cancel`): stopping the poll alone would leave the real
install running, and the model would reappear on the next read.

**An install already running is picked back up** when the page opens —
the engine keeps driving it across a page reload and across its own
restarts, so the page re-attaches rather than showing an untouched card
over a live download. A `cancelled` install is deliberately not resumed:
it is finished, by the user's own instruction.

**Remove asks first**, inline, and names what would be lost — the
capabilities from the engine's own non-destructive preview
(`DELETE /packages/{id}?confirm=false`, what Studio's confirm dialog
reads), rendered through this vertical's `capability_labels`. The engine
returns raw ids on purpose: naming them is the application's job, and
`vertical.yaml` is where the names are declared.

Nothing here mirrors a progress stream, stores install state, or names a
capability the app did not declare. The ownership map forbids all three.

### `setup_guard`

The first-launch gate. A vertical is a shell around models it does not
ship; with none installed every workflow fails at the engine, and the
failure arrives where the user can least act on it — mid-run, as a proxy
error naming a capability. This asks first, and does not open until the
answer is yes.

It takes no props. What it requires is DERIVED from manifest.yaml's
`capabilities_needed`, so an app is gated on what it already declared it
needs and cannot drift from a second list. `vertical.yaml` refines that,
never extends it:

```yaml
setup:
  required_capabilities:
    - capability: stt
      label: Speech Recognition
      recommended_model: whisper-tiny
    - capability: translate
      label: Translation
      optional: true
```

| Overlay key | Meaning |
|---|---|
| `label` | Human name for the capability. Defaults to the raw id |
| `description` | What it does, in a few words ("turn speech into text"). Defaults to the catalogue wording for known capabilities |
| `recommended_model` | Offered first. Falls back to the smallest listed |
| `optional` | Offered but never blocking — renders "Optional — skip for now." |

A card is headed by what its model does in which language — "Hindi
Translation" rather than "Translation" over "opus-mt-en-hi" — using the same
`modelTitle` as `models_page`, falling back to the capability's label when the
model declares no languages. What it does comes from the overlay's
`description`, else `capabilityDescription`'s plain defaults ("turn speech into
text").

An entry for a capability the manifest does not declare is DROPPED, not
honoured: gating an app on something it never said it needed is a gate
nobody can open.

**What counts as satisfied:** at least one INSTALLED model declaring the
capability. Not a configured one — settings can name a model that was
since removed — and not a catalogue entry, which is a thing you could
install rather than a thing you have.

**It is a gate, not a settings panel.** Settings is where you change a
choice already made; this is the reason there is nothing to change yet.

**Two steps, progressive disclosure.** A first launch should feel like a
minute of setup, not a configuration page, so decisions arrive one layer at a
time:

1. **Welcome** — the app's title, one sentence (`app.description`, which the
   backend fills from manifest.yaml's `description` when vertical.yaml does not
   restate it), and one primary button, "Set up <app>" (release labels such as
   "(Beta)" are dropped from the verb). No models, no choices.
2. **What to install** — one plain card per declared capability: a checkbox,
   what it does ("English Transcription — turn speech into text"), and the
   recommended model on one line ("Whisper Tiny · 717 MB · works on this
   machine"). Required capabilities are checked and locked; optional ones are
   unchecked with an "Optional" tag; installed ones read "Installed". One
   "Install selected" installs everything checked as a single batch, with the
   count and total size beside it. When nothing checked is left to install the
   button reads "Get Started", and the app opens only on that click — never on
   its own when an install lands, which pulled the screen away mid-read.

A required gap blocks on every launch; a later launch missing one goes straight
to step 2. With only optional gaps, the FIRST launch still walks both steps,
and finishing is written to the app's settings as `setup_completed` (the
settings file, not localStorage — a vertical's origin is its engine-assigned
port), so later launches open straight to the app. An unreadable settings file
counts as completed.

**The recommendation** (`pickModel`) never picks a model the engine calls
`incompatible` while another can run; then the vertical's `recommended_model`;
then the smallest model that is `compatible` outright.

**Other options.** A capability with more than one model has an "Other
options" link, collapsed by default. It reveals every candidate as the
Models page's own card (`ModelRow`, Studio's marketplace anatomy: size, score,
hardware, licence, compatibility) with `select` set, so the action slot reads
"Use this" / "Selected" instead of Add/Remove. Choosing one changes what
"Install selected" installs; a choice other than the recommendation is also
written to the capability's `model_selector` setting (one write for the whole
batch), so the run uses it. The recommendation itself pins nothing.

Two cases deliberately open the app rather than blocking it: no declared
capabilities (an app that works with nothing installed must not be locked
out of itself), and an unreachable engine (claiming models are missing
because we could not ask would lock someone out over a network blip).

Install progress is read from the engine's own install state machine —
the same source, and the same lifecycle, as `models_page`: the percentage
while it runs, "Finishing…" while the requirement list is re-read, and
the engine's own reason plus a Retry when it fails. Nothing is tracked
here; the ownership map forbids a second progress stream. A null
percentage renders an indeterminate bar rather than an invented figure.

The gate is satisfied when the ENGINE says the model is installed, never
when the install POST returns — that POST returns as soon as the job is
queued, so treating it as the answer put the button back to "Install"
seconds into a download with minutes to run, and reported a failure not
at all.

`VerticalApp` mounts the gate automatically. Placing `setup_guard` on a
page is for a vertical that wants it as a page of its own; it renders
nothing when there is nothing to install.

### `settings_page`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `title` | string | `"Preferences"` | Section heading |
| `settings` | schema | `vertical.yaml`'s `settings:` | Field schema override |

Saves per field. A single whole-form submit would send every field's
last-rendered value, so editing one setting could quietly revert
another.

Each field in the `settings:` schema declares a `type`: `model_selector`,
`dropdown`, `folder_picker`, `toggle`, or a plain text input (the default
for anything else).

- **`toggle`** — a checkbox bound to a boolean settings key. On its own it
  is just a yes/no setting; paired with `reveal_when` on another field it
  gates that field's visibility, mirroring hutash-studio's Settings →
  Features (Prompt Improve: off by default, a model picker appears once
  it's on).
- **`reveal_when`** (any field type) — names a sibling `toggle` field; this
  row is hidden until that toggle reads true. For an OPTIONAL capability
  (translate, an assistant model, …) this keeps its model picker out of
  the way until the feature is turned on. Never put `reveal_when` on a
  field for a REQUIRED capability — hutash-studio shipped exactly that for
  Transcription (`3e0f730`) and reverted it the same day (`893010d`): a
  toggle defaulting off hid a feature from every user who had the model
  installed, with nothing that ever turned it on.
- **`model_selector` with no installed model for its capability** renders
  a button — "No model installed — Install from Marketplace" — that
  navigates to the vertical's `models` page, instead of a `<select>` whose
  only options are disabled. A dropdown with nothing pickable in it answers
  the wrong question; a working link answers the real one.

```yaml
settings:
  translate_enabled:
    type: toggle
    label: Translation
    description: Show a translation model picker
    default: false
  translate_model:
    type: model_selector
    capability: translate
    label: Translation Model
    reveal_when: translate_enabled
```

---

## Action widgets

### `action_button`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `label` | string | `"Run"` | Button text |
| `running_label` | string | `label` | Text while a run is in flight |
| `variant` | string | `"primary"` | See Control vocabulary |
| `block` | boolean | `false` | Full-width, large — a dialog's commit |
| `action` | string | — | `run_workflow` or `navigate` |
| `requires` | string[] | `[]` | Input ids that must be filled |
| `workflow` | string | — | Workflow to run |
| `routes` | Route[] | — | `when:` conditions choosing a workflow |
| `create_project` | string | — | `${inputs.<id>}` — record a project first |
| `then_navigate_to` | string | — | Page to open when the run finishes |
| `navigate_params` | object | — | Params for it; `${inputs.<id>}` resolved |
| `run_on_project` | boolean | `false` | Run against the project open on this page, using its stored media |
| `to` / `params` | | — | `navigate` target |

Disabled while running and while any `requires` input is empty, so the
button is never a promise the run cannot keep. Inputs are filtered to
the chosen workflow's declared ones, so a form value the workflow has no
parameter for is never sent.

`run_on_project: true` sends the open project's id and drops every
`type: file` input from the request; the backend fills those from the
project's stored `media/` file. A File still held by the page could belong
to another project opened earlier, so it is never sent in this mode. The
button stays disabled until the open project HAS stored media.

```yaml
- type: action_button
  props:
    label: Generate Subtitles
    action: run_workflow
    requires: [media_file]
    workflow: transcribe-translate
    routes:
      - when: { target_language: same }
        workflow: transcribe
    create_project: ${inputs.project_name}
    then_navigate_to: editor
```

### `export_controls`

| Prop | Type | Default | Meaning |
|---|---|---|---|
| `formats` | `{value,label,output?,extension?,filename?}[]` | `[txt]` | Offered formats; `output` names the run output this one carries, `extension` the suffix to write it under, `filename` a stem suffix that keeps two text outputs apart |
| `filename` | string | output id | Fallback stem; the project name wins |
| `preview_lines` | number | `8` | Lines VISIBLE in the preview; the rest scrolls |
| `save_folder` | string | `"exports"` | Project folder the export is written into |
| `style_input` | string | `"style_id"` | Workflow input holding the caption style |
| `platform_input` | string | `"platform"` | Workflow input holding the platform |
| `styled_format` | string | `"mp4"` | Format promoted to first and preselected once a style is chosen |
| `output` | string | — | Which output to export |

**`preview_lines` sizes the box; it does not truncate the content.** It
used to slice the text before rendering, so the preview held five lines
and the rest never reached the DOM — a box with `overflow` and a
max-height over content that ended above the fold, which reads as a
scroll area that refuses to scroll and leaves no way to check what is
about to be exported without exporting it. The full text renders now and
the count sets the height, which is the only reading a scrollable
preview can honour.

Exports what is ON SCREEN — edited segments, not the raw run output —
**for the output those segments ARE**. The chosen format's own `output`
decides first, then the panel's `output`, then the workflow's `primary:`
output, then the first produced; a workflow emitting `original` before
`translated` would otherwise export the untranslated transcript.

The edits reach only an output declared `subtitle_editor`, or one whose
own value is already timed text. Handing them to every output is a bug
of the same family in the other direction: hutash-podcast's show notes
and chapters come out of the same run as the transcript, so asking for
the notes wrote the transcript into a file named after them, and the
notes could not be exported at all.

The download is named after the open project (`Interview.srt`).

ONE action, and it writes to disk. **This is a desktop app.** An export
lands in the open project's folder; nothing is handed to a browser to
drop in `~/Downloads`, separated from the project it belongs to. The
button says what it does: **Save** for a text format, **Render video**
for `mp4` (minutes of ffmpeg, so it does not pretend to be instant).
Neither says "download".

The word is gone from the catalogue's UI everywhere. It survives only
where something really is fetched over a network — a MODEL download in
`models_page` and `setup_guard`.

Afterwards the panel says **where the file went** and offers **Open
file** and **Open folder**. Those go through `POST
/projects/{id}/open` in `hutash_workflow`: a vertical's UI runs in an
iframe with no route to the desktop shell, so without that endpoint the
links could only have been decoration. A path the user cannot reach is
barely better than no message.

There is no separate "Save to project" and no "Copy to clipboard". A
clipboard button had no meaning on an MP4, so keeping it made the
panel's actions depend on which card was selected. With no project open
the save has nowhere to go and says so rather than silently falling back
to the browser.

`mp4` is not a serialization but a RENDER: it posts to the backend,
which burns the subtitles into the video's frames with ffmpeg, then
downloads the result.

**A format carrying `output` names WHAT IT CARRIES.** When that output is
a file the RUN already produced, it is kept as it stands:

```yaml
formats:
  - value: mp4
    label: Dubbed Video (MP4)
    output: dubbed_video
```

When it is TEXT, that text is what gets serialized — which is what lets
one panel offer every deliverable of a run instead of only the
transcript. `extension` separates the card's value from the file's
suffix (two cards cannot share a value, and "notes" is not an
extension), and `filename` keeps two outputs apart when they genuinely
share one:

```yaml
formats:
  - value: srt
    label: Transcript with timestamps (SRT)
    output: transcript
  - value: notes
    label: Show Notes (Markdown)
    output: notes
    extension: md
  - value: chapters
    label: Chapters (TXT)
    output: chapters
    extension: txt
    filename: chapters      # "Episode One chapters.txt"
```

Without that last line both text outputs would be written to
`Episode One.txt` and the second Save would silently overwrite the
first.

A dubbing vertical is the case that needed it. Its deliverable is the
mix ffmpeg already wrote: bytes, not text this widget could rebuild from
the segments on screen, and not something to burn subtitles into either
— that would be a different artefact wearing the same extension, and the
app has no burn-in step to make it with. Checked BEFORE the render
branch, so a declared `output` wins over what `mp4` means elsewhere.

The copy is server-side (`POST /runs/save-to-project`): the bytes are
already on the server and the project folder is on the same disk, so
sending a video down to the browser and posting it back would move it
twice across a boundary it never needed to cross. Both ends are resolved
and contained first — the source inside the named run, the destination
inside the project.

When `style_input` holds a caption style, that render is ANIMATED: the
style, the platform and the word-level segments go with the request and
the backend builds an ASS document from them. The style is read from the
workflow inputs rather than a prop, so picking one in `style_picker` and
pressing Download here are the same decision without either widget
knowing the other exists. The SRT still rides along, so a backend that
does not recognise the style falls back to the plain burn-in.

With a style chosen, `styled_format` leads the format list and starts
selected: the user has just watched their captions animate and the video
is what they came for. Reordered, not filtered — the SRT is still one
click away — and it follows the promotion only ONCE, so someone who
deliberately switched back does not have the choice taken away again.

---

## Remembering an input

`persist: true` on a bound widget stores its value against the OPEN
PROJECT and restores it when that project is opened again.

```yaml
- type: dropdown
  bind: platform
  props:
    options_from: api:/platforms
    persist: true
```

Opt-in, per input, and never global. Most inputs must NOT be remembered:
a form reopening pre-filled with the last run's file is an offer to redo
work, and one that runs against the wrong video if nobody looks. What
qualifies is a decision about the project rather than about the form —
a caption style, a target platform.

Values are stored in the project's `.hsprj` marker, so they travel with
the folder. Only strings, numbers and booleans persist; anything else is
skipped rather than serialized into something unreadable.

Two related page flags matter here:

| Flag | Meaning |
|---|---|
| `reset_inputs: false` | This overlay is a PANEL over live work, not a form. Without it the renderer infers "it binds something, so clear it on open" — which throws away the inputs the page underneath still needs. |
| (unset) | Inferred: an overlay that binds anything is treated as a form and opens empty. Right for a new-project dialog. |

A `persist: true` input is spared by the reset either way — it was
restored from disk, and clearing it would discard a choice the user
never revisited.

---

## Design values

**`tokens.css` owns every design value. No other file in this package
states one.** Not a hex, not a px, not a bare rem, not a duration.

This is enforced, not encouraged: `tests/unit/frontend/DesignValues.test.ts`
audits every `.css`, `.ts` and `.tsx` file in the package and fails on
any literal outside `tokens.css`. It found 252 the first time it ran.

| Kind | Use instead |
|---|---|
| `#ff0000`, `rgb()`, `white` | a colour token |
| `12px` | never — every dimension is rem so the UI scales with the root font size |
| `0.75rem` | a `--space-*`, `--size-*` or a semantic token |
| `0.15s` | `--transition-fast` / `--transition-base` |
| `0 0 0 2px …` | `--focus-ring`, `--selection-ring`, `--shadow-*` |
| `"Inter", sans-serif` | `--font-family` / `--font-mono` |

If no token fits, **add one to `tokens.css` in the same commit**. Name it
for what it measures, not for the widget that first needed it — the next
widget that wants a 2rem square should find one rather than adding a
second name for it.

Reading a token in JavaScript is a last resort and there is exactly one:
`waveform.tsx` paints to a canvas, which cannot take `var()`. It reads
the computed value and falls back to the element's own `color`, never to
a literal — a hardcoded fallback is a design value in hiding, and the
one it replaced was invisible on a light theme.

---

## Creating new widgets

**1. Check the catalogue first.** Twenty-nine widgets exist (the Quick
reference table above lists them all). A new one is justified when no
existing widget expresses the thing — not when an existing one needs a
prop. Adding a prop to `subtitle_list` is nearly always better than a
second list widget.

**2. Obey the control vocabulary.** A new control for a choice already
covered above is a second answer to a settled question. If you believe a
case is genuinely different, change this document first and say why.

**3. Read state from context, never from props threaded down.** A
widget's position on a page is declared in YAML, so the renderer cannot
know what it needs. Use `useWorkflow`, `useMedia`, `useOptionalNavigation`,
`useOptionalAutosave`. Use the `Optional` variants wherever the widget
should still render without that context.

**4. Own no state another widget owns.** Segments live in MediaContext;
form values live in WorkflowContext; install state lives in the engine.
A widget keeping its own copy is a second source of truth, and it will
disagree.

**5. Take configuration from `widget.props`, and give every prop a
default.** A vertical declaring nothing must get sensible behaviour.
Booleans that default ON read `props.x !== false`; booleans that default
OFF read `props.x === true`.

**6. Emit no px, and no hardcoded colours.** Every dimension in `rem`,
every colour from a token. A user's font-size preference has to scale the
whole UI.

**7. Degrade, never throw.** A widget that cannot do its job renders an
honest placeholder. Taking the page down denies the user everything else
on it — `waveform_timeline` without Web Audio is the worked example.

**8. Never invent a value you do not have.** A progress bar with no real
figure is indeterminate, not 0%. A model whose compatibility the engine
has not reported is `unknown`, not compatible. Silence beats a
confident wrong answer.

**9. Register it and export it.** Add to `src/widgets/builtins.ts`, and
export the component from `index.ts` so a host can use it directly.

**10. Document it here, with the props it actually reads.** A registry
that lists a prop the code ignores is worse than no registry.

**11. Test the failure paths.** The happy path is the easy half. Test
what happens with no data, no project, no network, and no permission —
those are the states users meet.
