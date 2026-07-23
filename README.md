# intropy-templates

A library of **Intropy templates** — scaffolds rendered into
working projects by the `intropy` CLI (separate repo:
`integrio-intropy/intropy-cli`). This repo contains content, not code: the CLI
is the only renderer.

One engine: Go `text/template` + [sprig](https://masterminds.github.io/sprig/).
One manifest format. One skeleton tree per template.

## Templates

- **`hello-world/`** — minimal example template that exercises the manifest
  and skeleton conventions.
- **`transactional/`** — a transactional integration component template.

Block templates — scaffold one component of an integration system and record
its edge intent in `.intropy/scaffold.json` so `intropy system create` can wire
them into a SystemHost (see the "Block templates" section of `CLAUDE.md`):

- **`block-extractor/`** — reads from an external system and publishes a topic.
- **`block-loader/`** — subscribes to a topic and loads into an external system.
- **`system-host/`** — the Aspire SystemHost that declares the system's
  components and edges; scaffolded by `intropy system create`.

The transactional-integration block kind is carried by the `transactional`
template above (via its `intropy.dev/block-kind` label) rather than a separate
thin template.

## Template layout

```
<template>/
  template.yaml          required: the intropy.dev/v1 manifest
  skeleton/              required: rendered into the user's --output
    <files…>             `.tmpl` files are templated; everything else is copied
  README.md              optional: author-facing notes
  examples/              optional: test fixtures for local renders
```

Only `template.yaml` and `skeleton/` are seen by the renderer. Anything else at
the template root is for the author, not the scaffolded project.

## Rendering locally

```bash
intropy int create hello-world -o /tmp/hello-out \
    -f hello-world/examples/minimal.yaml --version main
```

Pass `--version main` while there is no release yet. Once releases exist, the
default `--version` resolves to the latest tag.

## Conventions

See [`CLAUDE.md`](./CLAUDE.md) for the full manifest schema and skeleton
conventions. In short:

- File contents are templated only when the filename ends in `.tmpl` (the
  suffix is stripped on output). Path segments are always templated.
- Reference parameters and derived values with `{{ .paramName }}`.
- The renderer runs with `missingkey=error`, so a typo like `{{ .Name }}`
  fails the render instead of producing an empty string.
