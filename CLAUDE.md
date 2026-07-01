# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## What this repo is

A library of **Intropy blueprints**. Each blueprint is a scaffold-style template
rendered into a working project by the `intropy` CLI (separate repo:
`integrio-intropy/intropy-cli`). The CLI is the only renderer; this repo
contains content, not code.

The model is intentionally narrow: one engine (Go `text/template` + sprig),
one manifest format, one source of truth per blueprint.

## Repository layout

```
<blueprint>/
  template.yaml          # required: the intropy.dev/v1 manifest
  skeleton/              # required: rendered into the user's --output
    <files…>             # `.tmpl` files are templated; everything else is copied
  README.md              # optional: author-facing — what this blueprint produces
  examples/              # optional: test fixtures for local renders
    minimal.yaml
    full.yaml
```

Rules:

- The CLI selects a blueprint via positional argument:
  `intropy int create hello-world -o ./out`.
- Only `<blueprint>/template.yaml` is parsed; only `<blueprint>/skeleton/` is
  walked by the renderer.
- Anything else at the blueprint root (README, examples, CHANGELOG, ADRs) is
  invisible to the renderer — it's for the blueprint author, not the
  scaffolded project.
- There is **no shared content between blueprints**. No repo-level
  `skeletons/` dir, no cross-blueprint includes. If two blueprints need the
  same file, duplicate it.

## Manifest schema (`<blueprint>/template.yaml`)

```yaml
apiVersion: intropy.dev/v1
kind: Template
metadata:
  name: <kebab-case>           # convention: match the directory name
  title: <Title Case>
  description: <one sentence>
  tags: [example, go, http]
  labels:                      # free-form; no key is load-bearing today
    intropy.io/template-level: example
spec:
  parameters:                  # raw JSON Schema; type must be "object"
    type: object
    required: [name]
    properties:
      name:
        type: string
        title: Service name
        description: Lowercase kebab-case identifier.
        pattern: "^[a-z][a-z0-9-]*$"
      port:
        type: integer
        default: 8080
  values:                      # optional; derived values rendered via Go template + sprig
    module: 'github.com/example/{{ .name }}'
```

Notes:

- **`spec.parameters` is plain JSON Schema.** Types: `string`, `boolean`,
  `integer`, `number`. Supported attributes: `title`, `description`, `pattern`,
  `enum`, `default`. The same schema validates CLI inputs and drives the
  Backstage form when a Template entity references this blueprint.
- **Parameter declaration order matters** for human display. Keep the YAML
  order intentional.
- **`spec.values` derives string values from parameters.** Each entry is a Go
  template (with sprig) rendered against the resolved parameters. Use for
  composite values needed in multiple skeleton files (e.g. a module path).
  Output is always a string. Don't chain entries — map iteration order
  is non-deterministic.
- **No `spec.steps`**, no `spec.owner`, no `nextSteps`. The model is
  intentionally narrow: a manifest declares parameters and the skeleton tree
  describes what gets written.

## Skeleton conventions (`<blueprint>/skeleton/`)

- **File contents are templated only if the filename ends in `.tmpl`.** The
  `.tmpl` suffix is stripped on output (`README.md.tmpl` → `README.md`).
- **Filenames and directory names are templated** with the same `{{ .param }}`
  syntax as file contents (e.g. `src/{{ .name }}.csproj`). The `.tmpl` suffix
  only governs whether *file contents* are rendered; path segments are always
  templated.
- **Use `{{ .paramName }}` to reference parameters and derived values.** The
  data context is a flat map containing parameters + `spec.values` entries.
- **Sprig is available.** `upper`, `lower`, `title`, `kebabcase`,
  `snakecase`, `replace`, `default`, `trim`, `len`, `get`, `dict`. Full list:
  https://masterminds.github.io/sprig/ .
- **Missing keys are a hard error.** The renderer runs with
  `missingkey=error`. Typos like `{{ .Name }}` (wrong case) fail the render
  with a clear message — they don't silently produce empty strings.
- **Sprig functions with side effects exist** (`env`, `now`, `uuidv4`,
  `getHostByName`). Don't use them — they break the "same inputs → same
  output" reproducibility guarantee.

Example skeleton file `skeleton/README.md.tmpl`:

```markdown
# {{ .name }}

Generated from the `{{ .name }}` blueprint.

Module: `{{ .module }}`
```

## Adding a new blueprint

1. Create the directory: `mkdir -p <name>/skeleton <name>/examples`.
2. Write `<name>/template.yaml` (manifest).
3. Write skeleton files under `<name>/skeleton/`. Suffix anything you want
   templated with `.tmpl`.
4. Write `<name>/README.md` describing what the blueprint produces and what
   parameters it takes (this is for humans browsing the repo, not for the
   scaffolded project).
5. Write at least one `<name>/examples/minimal.yaml` containing values that
   satisfy the required parameters.
6. Render it locally to confirm it works:

   ```bash
   intropy int create <name> -o /tmp/<name>-out \
     -f <name>/examples/minimal.yaml --version main
   ```

   Pass `--version main` while we don't have a release yet. Once releases
   exist, the default `--version` resolves to the latest tag.

7. Inspect `/tmp/<name>-out/` and confirm the rendered output matches what
   you expected.

## Releases and versioning

The CLI fetches blueprints by GitHub release tag:

- `intropy int create <name> -o ./out` → uses the latest GitHub release.
- `intropy int create <name> -o ./out --version v0.2.1` → uses that tag.
- `intropy int create <name> -o ./out --version main` → uses the default
  branch (works on any ref the GitHub tarball endpoint accepts).

To ship a new blueprint version, cut a GitHub release. There is no
intermediate index. The blueprint version is the release tag, not a commit
SHA — all blueprints in the repo share the same release cadence.

## What this repo is NOT

- **Not a Backstage scaffolder template repo.** Don't add
  `scaffolder.backstage.io/v1beta3` files. Don't add `spec.steps`,
  `intropy:workspace:template`, `publish:gitlab:merge-request`, or any
  scaffolder action references. Backstage talks to blueprints through a
  custom action that shells out to the `intropy` CLI; it does not render
  templates itself.
- **Not a multi-engine repo.** The renderer is Go `text/template` + sprig,
  full stop. One engine, one skeleton tree per blueprint.
- **Not a place for runtime code.** The CLI lives in
  `integrio-intropy/intropy-cli`. This repo ships content only.
