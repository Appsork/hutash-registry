# hutash-registry

Hutash is a local AI creative studio — models and apps run on your own
machine, with no account and no cloud. This repository is its public
package catalogue: the desktop app reads the index here and installs from
it. Nothing is built or run in this repo.

| Path | What |
|---|---|
| `index.json` | The catalogue the app reads: every package, its modality, its version |
| `pipelines/*.hutash` | Model packages — one per model (voiceover, transcribe, translate, chat, image, music, voice cloning) |
| `apps/*.hutash` | Application packages |
| `ratings/` | Hutash Score per model, and the [methodology](ratings/METHODOLOGY.md) behind it |
| `releases/latest.json` | Update manifest the desktop app checks |

A `.hutash` file is a zip holding a `manifest.yaml` and the code to run the
model — not the weights. Those are fetched at install time from the source
each package pins, which is why a multi-gigabyte model is a few kilobytes
here.

- Package format spec: <https://hutash.com/docs/format/>
- Get the app: <https://hutash.com/download/>
- Hutash: <https://hutash.com>
