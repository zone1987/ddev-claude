# DDEV Claude — Developer Guide

This document is for **maintainers and contributors** of the `ddev-claude` add-on. If you just want to
*use* the add-on, see [`README.md`](README.md) instead.

* [Repository layout](#repository-layout)
* [The Claude Code image](#the-claude-code-image)
* [Local testing](#local-testing)
* [Contributing](#contributing)
* [Releases](#releases)

## Repository layout

| Path | Purpose |
| --- | --- |
| `install.yaml` | Add-on manifest: which files DDEV installs into a project's `.ddev` directory. |
| `web-build/Dockerfile.claude` | Appended to DDEV's web image build; copies the `claude` binary in via `COPY --from=...`. |
| `commands/web/claude` | The `ddev claude` wrapper command (runs `claude "$@"` in the web container). |
| `config.claude.yaml` | `post-start` hook: persists auth in the global cache volume and installs Claude Code plugins. |
| `claude-plugins.example.json` | Template users copy to `.ddev/claude-plugins.json` to override marketplaces/plugins. |
| `claude-code/Dockerfile` | Source for the prebuilt `docker.io/zone1987/claude-code` image (see below). |
| `.github/workflows/build-image.yml` | Builds & publishes the image to Docker Hub. |
| `.github/workflows/tests.yml` | Runs the bats test suite on every push/PR. |
| `.github/scripts/update-checker.sh` | Helper for checking upstream add-on template updates. |
| `tests/test.bats` | Integration tests (installs the add-on into a real DDEV project). |

## The Claude Code image

The image source lives in this repo under [`claude-code/Dockerfile`](claude-code/Dockerfile) — a two-stage
build that installs `@anthropic-ai/claude-code` via npm and extracts the single self-contained binary onto a
slim Debian carrier (matching the glibc of DDEV's webserver image).

[`.github/workflows/build-image.yml`](.github/workflows/build-image.yml) builds and publishes it to Docker Hub:

* Runs daily at **06:00 UTC** (plus on manual dispatch and on pushes to `claude-code/**`).
* Resolves the latest version with `npm view @anthropic-ai/claude-code version`.
* Skips the build if that version's tag already exists on Docker Hub (no upstream release webhook exists, so this is a poll-and-gate).
* Builds multi-arch (`linux/amd64`, `linux/arm64`) and pushes `:latest` and `:<version>` tags.

> [!IMPORTANT]
> The workflow requires two repository secrets so it can push to Docker Hub:
> * `DOCKERHUB_USERNAME` — `zone1987`
> * `DOCKERHUB_TOKEN` — a Docker Hub access token with Read/Write/Delete

To build the image manually for testing:

```bash
docker build -t docker.io/zone1987/claude-code:dev claude-code/
```

## Local testing

The add-on is tested with [bats-core](https://bats-core.readthedocs.io/). The suite in
[`tests/test.bats`](tests/test.bats) installs the add-on into a throwaway DDEV project and asserts that the
`claude` binary and the `ddev claude` wrapper both work.

**Prerequisites:** [DDEV](https://docs.ddev.com), Docker, and the bats libraries:

```bash
brew install bats-core
brew tap bats-core/bats-core
brew install bats-assert bats-file bats-support
```

**Install your local working copy into a project** (the fastest feedback loop while developing):

```bash
ddev add-on get /path/to/ddev-claude
ddev restart
```

**Run the test suite** from the add-on root:

```bash
bats ./tests/test.bats
```

Useful variants:

```bash
# Skip the "install from release" test (only test the local copy)
bats ./tests/test.bats --filter-tags '!release'

# Verbose debugging output
bats ./tests/test.bats --show-output-of-passing-tests --verbose-run --print-output-on-failure
```

## Contributing

1. Branch off `main` and make your change.
2. Run the tests locally (`bats ./tests/test.bats --filter-tags '!release'`).
3. Open a pull request. The [`tests`](.github/workflows/tests.yml) workflow runs automatically.

To install and try out a specific branch or PR without merging:

```bash
# The main branch
ddev add-on get https://github.com/zone1987/ddev-claude/tarball/main

# Any other branch
ddev add-on get https://github.com/zone1987/ddev-claude/tarball/<branch>

# A pull request
ddev add-on get https://github.com/zone1987/ddev-claude/tarball/refs/pull/<PR_NUMBER>/head
```

Installed files carry a `#ddev-generated` marker so `ddev add-on get` can update them later. Keep that marker
on any file the add-on installs — removing it makes DDEV treat the file as user-owned and stop updating it.

## Releases

Create a GitHub release / tag (e.g. `v1.2.3`) to publish a new add-on version. Users install the latest
release with `ddev add-on get zone1987/ddev-claude`; the `release`-tagged bats test exercises that path.

Note that the **add-on version is independent of the Claude Code image version**. The image is rebuilt and
re-tagged automatically by [`build-image.yml`](.github/workflows/build-image.yml) whenever npm publishes a new
Claude Code version — you don't cut an add-on release for that. Only release the add-on when the add-on's own
files (commands, hooks, Dockerfile glue, docs) change.
