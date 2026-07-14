# intropy-blueprints

A library of **Intropy blueprints** — scaffold-style templates rendered into
working projects by the `intropy` CLI (separate repo:
`integrio-intropy/intropy-cli`). This repo contains content, not code: the CLI
is the only renderer.

One engine: Go `text/template` + [sprig](https://masterminds.github.io/sprig/).
One manifest format. One skeleton tree per blueprint.

## Blueprints

- **`hello-world/`** — minimal example blueprint that exercises the manifest
  and skeleton conventions.
- **`transactional/`** — a transactional integration component blueprint.

## Blueprint layout

```
<blueprint>/
  template.yaml          required: the intropy.dev/v1 manifest
  skeleton/              required: rendered into the user's --output
    <files…>             `.tmpl` files are templated; everything else is copied
  manifests/             optional: deployment-manifest template
    template.yaml          same intropy.dev/v1 schema as the root manifest
    skeleton/              rendered by `intropy manifests create` into deploy/
  README.md              optional: author-facing notes
  examples/              optional: test fixtures for local renders
```

Only `template.yaml`, `skeleton/`, and `manifests/` are seen by the renderer.
Anything else at the blueprint root is for the author, not the scaffolded
project.

## Rendering locally

```bash
intropy int create hello-world -o /tmp/hello-out \
    -f hello-world/examples/minimal.yaml --version main
```

Pass `--version main` while there is no release yet. Once releases exist, the
default `--version` resolves to the latest tag.

## Deployment manifests

`intropy manifests create`, run from inside a scaffolded project, walks up to
`.intropy/scaffold.json`, re-fetches the blueprint at the pinned release tag,
and renders `<blueprint>/manifests/skeleton/` into `<project>/deploy/` — a
kustomize `base/` + `overlays/{dev,prod}` tree.

Scaffold values that match declared manifest parameters (like `appId`) are
seeded automatically, so the command runs zero-prompt; the full scaffold value
map is also available to skeleton files under the reserved `.scaffold` key.
Validate the output with:

```bash
kustomize build deploy/overlays/dev
```

Known gap: the storage-binding components are placeholders
(`bindings.localstorage` on the sidecar's ephemeral filesystem) — replace them
with the integration's real transport, keeping the component names.

Projects scaffolded from a release older than the first one containing
`manifests/` get an actionable error; pass `--version <newer tag>` to render
manifests from a newer blueprint release.

## Conventions

See [`CLAUDE.md`](./CLAUDE.md) for the full manifest schema and skeleton
conventions. In short:

- File contents are templated only when the filename ends in `.tmpl` (the
  suffix is stripped on output). Path segments are always templated.
- Reference parameters and derived values with `{{ .paramName }}`.
- The renderer runs with `missingkey=error`, so a typo like `{{ .Name }}`
  fails the render instead of producing an empty string.
