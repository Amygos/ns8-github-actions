# Copilot Instructions for ns8-github-actions

## Project overview

This repository ships reusable GitHub Actions workflows and composite
actions for the [NethServer 8](https://github.com/NethServer/ns8-kickstart)
ecosystem. The repository itself is the product: most changes affect
caller repositories through `workflow_call` contracts, GitHub context
handling, and assumptions about files that exist in the consuming
`ns8-*` repository.

## Build, test, and lint commands

This repository does not define repository-local build, lint, or test
commands. There is no `package.json`, `Makefile`, standalone test
suite, or local lint entrypoint in this repo.

Targeted test execution is delegated to caller repositories through the
reusable test workflows:

- `test-on-ubuntu-runner.yml`
- `test-on-digitalocean-infra.yml`

Those workflows accept `script`, `path`, and `args`, so "run a single
test" is implemented in the consuming module repository, not here.

## High-level architecture

### Reusable workflows

All top-level workflows are designed for `on: workflow_call` reuse.

- `publish-branch.yml` checks out the caller repository, optionally
  writes a base64-decoded `.netrc`, runs the caller-provided
  `build-images.sh`, and publishes the returned images to
  `ghcr.io/{owner}`. Branches `main` and `master` also get a `latest`
  tag.
- `module-info.yml` is the metadata hub. It derives owner, module name,
  tag, SHA, the main image name, and the full image list from GitHub
  context plus the `org.nethserver.images` container label. Its `images`
  output is JSON and is meant for downstream workflows.
- `scan-with-trivy.yml` has two scan paths: filesystem scans for the
  caller repo's `ui/` and `imageroot/`, and image scans for the JSON
  array passed in `inputs.images`. It can upload SARIF, update the
  dependency graph, generate CycloneDX SBOMs, and attach SBOMs to
  releases.
- `test-on-ubuntu-runner.yml` and `test-on-digitalocean-infra.yml`
  share the same caller-facing contract (`script`, `path`, `args`,
  `repo_ref`, `coremodules`, `corebranch`, `debug_shell`) but run in
  different environments. The DigitalOcean variant additionally
  provisions infrastructure from `NethServer/ns8-terraform-infra`,
  supports a `leader_nodes` matrix, and writes commit statuses around
  the test run.
- `build-apidoc.yml` and `clean-apidoc.yml` manage generated API docs.
  `build-apidoc.yml` uses the local composite action to populate
  `.apidoc/dst` and publishes that tree to an `apidoc-{ref}` branch with
  `git commit-tree`.

### Composite actions

- `.github/actions/build-apidoc` builds API docs inside a Node container
  managed by `buildah`. Its `id-rename.js` helper renames copied schema
  files from each schema's `$id`, so output naming follows schema
  metadata rather than source filenames.
- `.github/actions/delete-image` and
  `.github/actions/delete-untagged-versions` are GHCR maintenance
  wrappers around `gh api` package-version deletion endpoints.

## Key conventions

- Use `buildah`, not Docker, for image build/push logic and for the API
  docs containerized step.
- Module identity is derived from GitHub context, not workflow inputs:
  owner and repository name are lowercased, and module names strip the
  `ns8-` prefix.
- The main module image is always `ghcr.io/{owner}/{module}:{tag}`.
  Additional images come from the `org.nethserver.images` label on the
  primary image.
- `images` values are JSON arrays, not space-delimited strings.
  `module-info.yml` emits them with `jq`, and `scan-with-trivy.yml`
  consumes them with `fromJson`.
- Tag handling is cross-file behavior. `publish-branch.yml` pushes
  `latest` for both `main` and `master`, while `module-info.yml` maps
  `main` to `latest` when classifying releases. If tag semantics change,
  review both workflows together.
- The reusable test workflows do not own the tests; they provision the
  environment and invoke the caller repo's script, usually
  `test-module.sh`.
- The DigitalOcean test workflow namespaces infrastructure
  deterministically per ref by combining a sanitized ref token with the
  short commit SHA. Update workspace names, SSH hostnames, and status
  contexts together if that scheme changes.
- API doc generation only looks for `validate-input.json`,
  `validate-output.json`, and `validator-definitions.json` under
  `imageroot/`. Schemas whose `$id` uses the `urn:` protocol are
  skipped.
- Generated API docs are published with the bot identity
  `nethbot <nethbot@nethesis.it>` and a synthetic tree commit, not by
  editing a checked-out branch.
- `netrcb64` is a base64-encoded `.netrc` used during image builds, and
  `do_token` is required for DigitalOcean-backed integration tests.
