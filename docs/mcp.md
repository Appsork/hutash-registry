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
