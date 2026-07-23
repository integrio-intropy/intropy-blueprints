# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## What this repo is

A library of **Intropy templates**. Each template is a scaffold
rendered into a working project by the `intropy` CLI (separate repo:
`integrio-intropy/intropy-cli`). The CLI is the only renderer; this repo
contains content, not code.

The model is intentionally narrow: one engine (Go `text/template` + sprig),
one manifest format, one source of truth per template.

## Repository layout

```
<template>/
  template.yaml          # required: the intropy.dev/v1 manifest
  skeleton/              # required: rendered into the user's --output
    <files…>             # `.tmpl` files are templated; everything else is copied
  README.md              # optional: author-facing — what this template produces
  examples/              # optional: test fixtures for local renders
    minimal.yaml
    full.yaml
```

Rules:

- The CLI selects a template via positional argument:
  `intropy int create hello-world -o ./out`.
- Only `<template>/template.yaml` is parsed; only `<template>/skeleton/` is
  walked by the renderer.
- Anything else at the template root (README, examples, CHANGELOG, ADRs) is
  invisible to the renderer — it's for the template author, not the
  scaffolded project.
- There is **no shared content between templates**. No repo-level
  `skeletons/` dir, no cross-template includes. If two templates need the
  same file, duplicate it.

## Manifest schema (`<template>/template.yaml`)

```yaml
apiVersion: intropy.dev/v1
kind: Template
metadata:
  name: <kebab-case>           # convention: match the directory name
  title: <Title Case>
  description: <one sentence>
  tags: [example, go, http]
  labels:                      # free-form, except the intropy.dev/* keys below
    intropy.io/template-level: example
    # intropy.dev/data-flow: in|out|both|internal — copied into scaffold.json
    # intropy.dev/block-kind: <kind> — marks a block template (see "Block templates")
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
  Backstage form when a Template entity references this template.
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

## Skeleton conventions (`<template>/skeleton/`)

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

Generated from the `{{ .name }}` template.

Module: `{{ .module }}`
```

## Block templates

A **block template** scaffolds one component of an Intropy integration system
and declares which topology block kind it is via the label
`intropy.dev/block-kind: extractor | loader |
transactional-integration`. When that label is present, `intropy int create`
hoists the template's **well-known edge parameters** into a structured
`topology` section of the scaffolded project's `.intropy/scaffold.json`.
`intropy system create` later joins those sections across sibling components
into a wired SystemHost (same topic name ⇒ shared topic, provided + consumed
API name ⇒ shared API, same connector name ⇒ shared connector).

The well-known parameter names per kind (validated by the CLI at create time;
DNS-1123 unless noted). A block declares only its edge *shape* — which external
system it reaches and which topics/APIs it touches. The **transport** (how the
external system is reached; the Dapr binding type) is *not* a block parameter: it
is resolved once, at `intropy system create`, which prompts for any connector
whose transport is still unset:

| Kind | Required | Optional |
|------|----------|----------|
| `extractor` | `sourceSystem`, `publishesTopic` | `connectorName` (defaults to `sourceSystem`), `publishesContract` (PascalCase; defaults to PascalCase of topic) |
| `loader` | `subscribesTopic`, `targetSystem` | `connectorName` (defaults to `targetSystem`) |
| `transactional-integration` | `providesApi`, `targetSystem` | `providesContract` (defaults to PascalCase of API), `connectorName` |

All kinds may also declare `consumesApis` (comma-separated).

Further block template conventions:

- **`name` is the DNS-1123 component name**, not a PascalCase project name.
  It becomes the folder name, the csproj name (`skeleton/{{ .name }}.csproj.tmpl`),
  and the Dapr app-id — the SystemHost resolves component projects by the
  convention `../<name>/<name>.csproj`, so all three must match. Derive the
  PascalCase form as a `projectName` value for `RootNamespace`.
- The skeleton is **flat** (csproj at the skeleton root, no `src/`) so the
  rendered project satisfies that convention.
- Don't declare a Contracts `ProjectReference` in the csproj —
  `intropy system create` inserts it when it generates the system's shared
  Contracts project.
- The pub/sub component name is a **system-level decision**: workers read it
  from the runtime config the SystemHost injects (`INTROPY__CONFIG`), never
  hard-code it in the skeleton.

## The `AGENTS.md` convention

Every skeleton ships an `AGENTS.md.tmpl` at its root. `AGENTS.md` (per the
[agents.md](https://agents.md/) standard, auto-loaded by most coding agents)
is the scaffolded project's **manifest for agents**: the facts about *this*
component that no skill can know. It is not a tutorial — generic framework
how-to lives in the Intropy skills collection (`intropy skills collection add
--name intropy --ref harbor.intropy.io/skills/index:latest`), which
`int create` offers to install into `.agents/skills/`.

Each skeleton also ships a one-line `CLAUDE.md` containing exactly
`@AGENTS.md` — Claude Code doesn't auto-load AGENTS.md, so this import gives
it the same context. Keep it one line; never put content in it.

Rules for authoring `AGENTS.md.tmpl`:

- **Facts only, no teaching.** State what the component is, its component /
  topic / binding names (with rootPaths and ports), app id, and the key-file
  map. Do not restate framework conventions (naming-sync rules, builder
  usage, DI patterns) — those belong to the skills and duplicating them here
  drifts.
- **One canonical run path.** If the skeleton ships a `Taskfile.yml`,
  `task run` is canonical: `AGENTS.md` points at it and briefly says what it
  does; it never duplicates the underlying `dapr run` command. If there is no
  Taskfile, the raw `dapr run` command lives in `AGENTS.md` (and `README.md`
  must agree with it — same ports, same flags).
- **Project-specific deviations are facts.** Deliberate departures from
  framework defaults (e.g. "idempotency omitted in this sample; add
  `.WithIdempotency(...)` in …") belong in a short Development notes section.
- **One skills pointer.** End with a single "Framework guidance" line
  pointing at the skills collection — no per-skill routing table.
- **Section structure:** title + one-liner, Project overview, Important
  files, Build and run, optional Development notes / Testing, Framework
  guidance.

The facts in `AGENTS.md` must match the skeleton (component YAML `metadata.name`
and rootPaths, `Constants.cs` values, Taskfile vars, `.http` ports). When you
change one, change the others in the same commit.

## Adding a new template

1. Create the directory: `mkdir -p <name>/skeleton <name>/examples`.
2. Write `<name>/template.yaml` (manifest).
3. Write skeleton files under `<name>/skeleton/`. Suffix anything you want
   templated with `.tmpl`.
4. Write `<name>/README.md` describing what the template produces and what
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

The CLI fetches templates by GitHub release tag:

- `intropy int create <name> -o ./out` → uses the latest GitHub release.
- `intropy int create <name> -o ./out --version v0.2.1` → uses that tag.
- `intropy int create <name> -o ./out --version main` → uses the default
  branch (works on any ref the GitHub tarball endpoint accepts).

To ship a new template version, cut a GitHub release. There is no
intermediate index. The template version is the release tag, not a commit
SHA — all templates in the repo share the same release cadence.

## What this repo is NOT

- **Not a Backstage scaffolder template repo.** Don't add
  `scaffolder.backstage.io/v1beta3` files. Don't add `spec.steps`,
  `intropy:workspace:template`, `publish:gitlab:merge-request`, or any
  scaffolder action references. Backstage talks to templates through a
  custom action that shells out to the `intropy` CLI; it does not render
  templates itself.
- **Not a multi-engine repo.** The renderer is Go `text/template` + sprig,
  full stop. One engine, one skeleton tree per template.
- **Not a place for runtime code.** The CLI lives in
  `integrio-intropy/intropy-cli`. This repo ships content only.
