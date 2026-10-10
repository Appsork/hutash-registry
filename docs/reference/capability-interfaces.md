# Capability interfaces

Every model that declares a capability serves the SAME inputs and outputs
for it, under the same names, so a workflow written against the capability
works with any model that declares it. A workflow step names a capability
and these parameter names; it never needs to know which model the user
selected.

A capability is named by its **short id** — `stt`, `tts`, `translate`,
`image`, `llm`, `audio-sep`, `music`, `voice-clone` — exactly
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
