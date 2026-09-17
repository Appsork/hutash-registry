# Hutash CLI

The `hutash` command controls models, apps, and generation from the terminal. It's a thin HTTP client that talks to the engine (`hutashd`) on `localhost:47990` — the same relationship as `docker` to `dockerd`.

## Global flags and exit codes

Available on every command.

| Flag | Description |
|---|---|
| -h, --help | Help |
| --json | Output JSON (default when piped) |
| --token string | Engine API token |
| -v, --version | Version |

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Error |
| 2 | Not found |
| 3 | Engine unreachable |

## Generate: Running a capability

```
hutash run <model> <capability> [--key value ...] [--output path]
```

### Examples

```
hutash run kokoro tts --text "Hello world" --output hello.wav
hutash run ace-step-1.5 music --prompt "ambient piano" --output song.wav
hutash run chatterbox clone --ref sample.wav --text "Hello"
hutash run whisper-large-v3 stt --file recording.mp3
```

### Interactive

```
$ hutash run qwen3-1.7b improve
>>> Improve this: hello there
>>> /bye
```

### Save as a Studio asset

```
hutash run kokoro tts --text "Hello" --project "My Project"
```

## Models: Managing models

| Command | Description |
|---|---|
| hutash install \<id\> | Install a model |
| hutash uninstall \<id\> | Remove a model (venv + weights) |
| hutash start \<id\> | Start a stopped model |
| hutash stop \<id\> | Stop a running model |
| hutash restart \<id\> | Restart a model |
| hutash list | List installed models (all states) |
| hutash list --running | Show only running models |
| hutash ps | List running models (alias) |
| hutash health \<id\> | Health-check a running model |
| hutash inspect \<id\> | Full model detail (JSON) |
| hutash verify \<id\> | Check venv integrity |
| hutash repair \<id\> | Verify and repair a model's venv |
| hutash logs \<id\> | Tail a model's log (-n 50 default) |

Aliases: `pull` = `install`, `rm` = `uninstall`.

## Apps: Managing applications

| Command | Description |
|---|---|
| hutash apps list | List installed applications |
| hutash apps install \<id\> | Install an application |
| hutash apps uninstall \<id\> | Remove an application |
| hutash apps start \<id\> | Start an application |
| hutash apps stop \<id\> | Stop an application |

## Engine: Talking to the daemon

| Command | Description |
|---|---|
| hutash ping | Check the engine is reachable |
| hutash status | Show engine resources and host info |
| hutash version | Show CLI and daemon versions |
| hutash config | Show engine configuration |
| hutash config get \<key\> | Show one config value |
| hutash config set \<key\> \<value\> | Change one config value |
| hutash activity | Show recent activity |
| hutash activity -f | Stream activity continuously |

## Projects and environment: Studio projects, and how the CLI finds the engine

| Command | Description |
|---|---|
| hutash projects list | List Studio projects |

| Variable | Purpose |
|---|---|
| HUTASH_ENGINE | Engine address (default `http://localhost:47990`) |
| HUTASH_TOKEN | API token (fallback: `hutashd.json`) |
