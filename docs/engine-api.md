# Engine API reference

The engine exposes a local HTTP API on `localhost:47990`. Studio, the OS shell, and the CLI are all clients of it — anything they can do, your own code can do too.

## Endpoints: The REST surface

| Method | Path | Description |
|---|---|---|
| GET | /health | Engine liveness and version |
| GET | /packages | List installed and available packages |
| POST | /packages/:id/install | Install a package; returns a job id |
| DELETE | /packages/:id | Uninstall a package and remove its files |
| POST | /packages/:id/start | Load a model into memory |
| POST | /packages/:id/stop | Unload a model |
| POST | /infer | Run inference on a loaded model |
| GET | /activity | Recent jobs with status and timing |
| GET | /activity/stream | Server-sent events for live job progress |
| GET | /hardware | Detected GPU, VRAM, RAM, and compatibility |
| GET | /settings | Read engine settings |
| PATCH | /settings | Update engine settings |

## Authentication: Bearer token

The API binds to localhost and requires a bearer token, so other processes on your machine cannot drive the engine unless you hand them the token. Read it from the engine settings or the `HUTASH_TOKEN` environment variable.

Request:

```
curl -H "Authorization: Bearer $HUTASH_TOKEN" \
     http://localhost:47990/health
```

Response:

```json
{
  "status": "ok",
  "version": "1.0.0",
  "models_loaded": 1
}
```

## Inference: Running a job

Inference is asynchronous: the call returns a job id, and progress arrives on the activity stream.

`POST /infer`:

```json
{
  "package": "kokoro",
  "task": "tts",
  "input": { "text": "Welcome to Hutash", "voice": "heart" },
  "out": "./welcome.wav"
}
```

`202 Accepted`:

```json
{
  "job_id": "j_8f2c41",
  "status": "queued",
  "position": 1
}
```
