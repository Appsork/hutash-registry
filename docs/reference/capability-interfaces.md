# Capability interfaces

Every model that declares a capability serves the SAME inputs and outputs
for it, under the same names, so a workflow written against the capability
works with any model that declares it. A workflow step names a capability
and these parameter names; it never needs to know which model the user
selected.

A capability is named by its **short id** — `stt`, `tts`, `translate`,
`image`, `video`, `upscale`, `llm`, `audio-sep`, `music`, `voice-clone` — exactly
as a package declares it in `manifest.yaml` and exactly as
`GET /catalogue?capability=<id>` matches it. There is no other spelling of
a capability id and no alias for one.

`required` inputs must be sent; `optional` ones may be left out. A manifest
may declare an optional name below under `controls:` rather than `inputs:`
(for example `language`, `voice`, `width`): what is standard is the NAME, and
the engine sends both kinds the same way, as one named field.

A workflow step passes every parameter by name under `params:` — never as a
bare `input:` — and the workflow runner refuses, when an app starts and
again on each run, a step that names a parameter its model does not declare
or leaves out a required one.

## stt

```yaml
inputs:
  file:     {type: audio, required: true}
  language: {type: string, required: false}
outputs:
  text:     string
  segments: array
```

## tts

```yaml
inputs:
  text:  {type: string, required: true}
  voice: {type: string, required: false}
outputs:
  audio: wav bytes
```

## translate

```yaml
inputs:
  text:        {type: string, required: true}
  source_lang: {type: string, required: false}
  target_lang: {type: string, required: false}
outputs:
  text: string
```

## image

```yaml
inputs:
  prompt:          {type: string, required: true}
  negative_prompt: {type: string, required: false}
  width:           {type: int,    required: false}
  height:          {type: int,    required: false}
outputs:
  image: png bytes
```

## llm

```yaml
inputs:
  prompt:     {type: string, required: true}
  max_tokens: {type: int,    required: false}
outputs:
  text: string
```

## audio-sep

```yaml
inputs:
  file: {type: audio, required: true}
outputs:
  vocals:       wav
  instrumental: wav
```

## music

```yaml
inputs:
  prompt:   {type: string, required: true}
  duration: {type: float,  required: false}
outputs:
  audio: wav bytes
```

## voice-clone

```yaml
inputs:
  text:            {type: string, required: true}
  reference_audio: {type: file,   required: false}
outputs:
  audio: wav bytes
```

## video

```yaml
inputs:
  prompt:     {type: string, required: true}
  width:      {type: int,    required: false}
  height:     {type: int,    required: false}
  num_frames: {type: int,    required: false}
  num_steps:  {type: int,    required: false}
outputs:
  video: mp4 bytes
```

Text to video. `num_frames` is a count of frames, not seconds; how many a
second of playback takes is the model's own frame rate (Wan 2.1 writes 15 a
second). A model may round `width`, `height` and `num_frames` to the nearest
values it supports and says so in its own description — Wan 2.1, for example,
takes sides in multiples of 16 and frame counts of the form 4k + 1. A model may
declare further controls of its own (a seed, a guidance scale); only the names
above are standard.

The output is declared `type: mp4`, which makes the inference server answer with
raw `video/mp4` bytes rather than JSON, and which Studio's schema builder turns
into a `video` output rendered in a player.

## upscale

```yaml
inputs:
  image: {type: image_file, required: true}
  scale: {type: int,        required: false}   # 2 or 4
outputs:
  image: png bytes
```

Image super-resolution. `scale` is 2 or 4 and defaults to 4; any other value is
refused, not rounded. The result is `scale` times the input in each direction,
as a PNG. The input is named `image` rather than `file` because it is the one
image a model is handed, the same way `reference_audio` names the one audio clip
a voice-clone model is handed.

The capability's modality is `upscale`. Studio has no Upscale feature yet, so
such a model is reached by applications and workflows, not by a Studio page.

## inpaint — proposed, not yet standard

**Nothing declares this capability and nothing enforces it.** No package, no
workflow and no application uses `inpaint`, and the interface below is a
starting point for discussion, not a contract. It is recorded here so the name
is reserved and the shape is argued about once. Do not write a workflow against
it until a package declares it and this heading loses the word "proposed".

```yaml
# PROPOSED — not implemented
inputs:
  image:  {type: image_file, required: true}
  mask:   {type: image_file, required: true}    # white = repaint, black = keep
  prompt: {type: string,     required: true}
  width:  {type: int,        required: false}
  height: {type: int,        required: false}
  num_steps: {type: int,     required: false}
outputs:
  image: png bytes
```

Open questions: whether the mask is a separate upload or an alpha channel on
`image`; whether `negative_prompt` joins the standard names (the `image`
capability already has it); and whether the output keeps the input's size or
the model's.
