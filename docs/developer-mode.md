# Developer Mode

Developer Mode is a Hutash OS setting for testing a package you're building — an application or a model pipeline — without publishing it anywhere first. Turn it on from Settings → Developer.

## What it gives you

Turning it on creates a developer workspace on disk, seeded with:

- A package-authoring reference — the field-by-field facts for both package types, the same ones the [format documentation](../format/index.html) is built from.
- A folder for applications you're working on, and a folder for model pipelines you're working on.

The workspace is created once and never overwritten — a doc or folder you've already edited stays as you left it.

## Loading a local package

Two pickers, one for each package type:

- **Load Application** — point it at a folder laid out like a `.hutash` application (see [the .hutash format](../format/index.html)).
- **Load Model Pipeline** — point it at a folder laid out like a `.hutashm` model pipeline.

Either one installs the folder through the same install path a published package goes through — not a separate preview mode. Once loaded, it behaves exactly like anything installed from the marketplace: it shows up in `hutash list` or `hutash apps list`, and you run it the same way.

## Safety: no silent collisions

Loading a local package whose id matches one already installed from the official catalogue is refused, not overwritten. This keeps a work-in-progress build from silently replacing a real install that happens to share its id.

## Removing a loaded package

Every loaded package appears in a list with a Remove button, which uninstalls it the same way removing any other installed package does.

## See also

- [Walkthrough](../format/walkthrough/index.html) — build a package by hand, then load it here to run it.
- [The .hutash format](../format/index.html) — what a package actually contains.
