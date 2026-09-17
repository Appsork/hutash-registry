# Hutash MCP server

Hutash ships an MCP server (`hutash-mcp`) that exposes your locally installed models as tools. Any MCP-compatible client — your MCP client, an MCP-compatible CLI, or other agents — can generate speech, music, voice clones, and transcriptions through your local models.

## Setup: Connecting a client

The engine must be running and the models you want must already be installed.

### MCP-compatible CLI

One command registers the server:

```
<your-cli> mcp add hutash -- /path/to/hutash-mcp
```

### Your MCP client

Add the server to your MCP client's config file:

```json
{
  "mcpServers": {
    "hutash": {
      "command": "/path/to/hutash-mcp"
    }
  }
}
```

Restart the client after saving. The Hutash tools appear in its tool list.

## Engine tools: Always available

| Tool | Description |
|---|---|
| list_models | List installed models with status and capabilities |
| list_apps | List installed applications |
| show_model | Show a model's full detail and manifest |
| install | Install a model or app by ID |
| uninstall | Remove an installed model |
| start | Start a stopped model |
| stop | Stop a running model |
| get_status | Engine resource usage (RAM, VRAM, GPU) |
| get_activity | Recent install and lifecycle activity |

## Project tools: Writing into Studio

| Tool | Description |
|---|---|
| list_projects | List all Studio projects |
| generate_in_project | Generate content and register it as a Studio asset in a named project — the same mechanism Studio's own UI uses |

## Per-model tools: One tool per installed capability

Tool names follow the pattern `{model_id}_{capability}`. They appear automatically when a model is installed and disappear when it is removed. Each tool's input schema is built from the model's own manifest, so there is no hardcoded per-model knowledge.

| Tool | What it does |
|---|---|
| kokoro_tts | Generate speech with Kokoro |
| ace_step_1_5_music | Generate music with ACE-Step |
| chatterbox_clone | Clone a voice with Chatterbox |
| whisper_large_v3_stt | Transcribe with Whisper Large v3 |
| qwen3_1_7b_improve | Refine text with Qwen3 |

### Catch-all

`generate_with_local_model` runs inference against any installed model when the request doesn't match a specific per-model tool.

### Requirements

- Hutash OS installed
- The engine (hutashd) running
- Models installed before their tools appear
- An MCP server restart to pick up tools from a freshly installed model

## Vertical workflow tools: Subs and Podcast

Subs and Podcast are applications, not model pipelines. Once one is installed and running, each of its workflows is also available as an MCP tool — the same discoverability idea as the tools above, scoped to that app's own workflows. A tool's inputs and outputs match what its workflow declares, so an assistant sees the same fields a person filling in the app's own form would see.

### Subs

Transcribe and translate video subtitles locally using AI models.

| Tool | Description |
|---|---|
| transcribe | Transcribe — media_file (required), source_language. Returns: Subtitles (srt, vtt, txt); Raw Segments (json) |
| translate | Translate Subtitles — source_file (required), source_language, target_language (required). Returns: Translated Subtitles (srt, vtt, txt) |
| transcribe_translate | Transcribe + Translate — media_file (required), source_language, target_language (required). Returns: Original Subtitles (srt, vtt, txt); Translated Subtitles (srt, vtt, txt); Translated Segments (json) |
| render_styled | Render Styled Video — media_file (required), segments (required), style, platform, output_dir (required). Returns: Captioned Video (mp4); Styled Subtitles (ass) |

### Podcast

Turn podcast episodes into transcripts, show notes, and chapters.

| Tool | Description |
|---|---|
| transcribe | Transcribe Episode — media_file (required). Returns: Transcript (srt, vtt, txt); Raw Segments (json) |
| full_process | Full Episode Processing — media_file (required). Returns: Transcript (srt, vtt, txt); Show Notes (txt); Chapters (txt) |
| notes_only | Show Notes from Audio — media_file (required). Returns: Show Notes (txt); Transcript (srt, vtt, txt) |
| chapters_only | Chapters from Audio — media_file (required). Returns: Chapters (txt); Transcript (srt, vtt, txt) |
| chapters | Generate Chapters — transcript (required). Returns: Chapters (txt) |
| show_notes | Generate Show Notes — transcript (required). Returns: Show Notes (txt) |
