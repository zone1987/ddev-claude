[![add-on registry](https://img.shields.io/badge/DDEV-Add--on_Registry-blue)](https://addons.ddev.com)
[![tests](https://github.com/zone1987/ddev-claude/actions/workflows/tests.yml/badge.svg?branch=main)](https://github.com/zone1987/ddev-claude/actions/workflows/tests.yml?query=branch%3Amain)
[![build-image](https://github.com/zone1987/ddev-claude/actions/workflows/build-image.yml/badge.svg?branch=main)](https://github.com/zone1987/ddev-claude/actions/workflows/build-image.yml?query=branch%3Amain)
[![last commit](https://img.shields.io/github/last-commit/zone1987/ddev-claude)](https://github.com/zone1987/ddev-claude/commits)
[![release](https://img.shields.io/github/v/release/zone1987/ddev-claude)](https://github.com/zone1987/ddev-claude/releases/latest)

# DDEV Claude <!-- omit in toc -->

* [What is DDEV Claude?](#what-is-ddev-claude)
* [Installation](#installation)
* [Authentication](#authentication)
* [Usage](#usage)
* [How it works](#how-it-works)
* [The Claude Code image](#the-claude-code-image)
* [Configuration & customization](#configuration--customization)
* [Removal](#removal)
* [Resources](#resources)
* [Credits](#credits)

## What is DDEV Claude?

This [DDEV](https://docs.ddev.com) add-on makes [Claude Code](https://docs.claude.com/en/docs/claude-code) available **inside** your DDEV web container. After installing it, `ddev claude` starts Claude Code right next to your project's code, PHP runtime, and tooling — no host installation required.

The add-on does **not** install Claude per project at build time. Instead it pulls a tiny, prebuilt, single-binary image (`docker.io/zone1987/claude-code`) via a single `COPY --from=...` layer, so installation is a fast copy rather than an `npm install`. That image is rebuilt automatically every day and only republished when a new Claude Code version is available on npm.

## Installation

```bash
ddev add-on get zone1987/ddev-claude
ddev restart
```

To install a specific branch or pull request while testing:

```bash
ddev add-on get https://github.com/zone1987/ddev-claude/tarball/main
```

## Authentication

Claude Code authenticates with a long-lived **OAuth token** (valid for one year).

1. On your **host machine** (where Claude Code is installed) generate a token:

   ```bash
   claude setup-token
   ```

2. Export it in your shell so it is available to DDEV. Add this to your `~/.zshrc` (or `~/.bashrc`):

   ```bash
   export CLAUDE_CODE_OAUTH_TOKEN="<your-token>"
   ```

   Then reload it:

   ```bash
   source ~/.zshrc
   ```

3. Restart the project so the token is passed into the container:

   ```bash
   ddev restart
   ```

The add-on creates a `config.token.local.yaml` in your project's `.ddev` directory that forwards `CLAUDE_CODE_OAUTH_TOKEN` from your host into the web container. This file matches DDEV's default `.gitignore` pattern (`/config.*.local.y*ml`), so it is **never committed**.

> [!NOTE]
> Alternatively you can log in interactively the first time you run `ddev claude` (no token needed). Your login is persisted in DDEV's global cache volume (see [How it works](#how-it-works)) and survives restarts.

## Usage

Start Claude Code inside the web container:

```bash
ddev claude
```

Pass any Claude Code arguments through the wrapper:

```bash
ddev claude --version
ddev claude -p "explain this project"
```

Or open a shell first and run it manually:

```bash
ddev ssh
claude
```

If you use Claude Code plugins, cache them once after install:

```text
/reload-plugins
```

## How it works

* **`web-build/Dockerfile.claude`** — appended to DDEV's web image build. A single `COPY --from=docker.io/zone1987/claude-code:latest` line copies the `claude` binary into `/usr/local/bin/claude`.
* **`commands/web/claude`** — the `ddev claude` wrapper command. It runs `claude "$@"` inside the web container.
* **`config.claude.yaml`** — a `post-start` hook that:
  * Persists Claude's auth and state in DDEV's **global cache volume** (`/mnt/ddev-global-cache/claude-code/shared`). This survives `ddev restart`, `ddev rebuild`, `ddev poweroff` and even `ddev delete`, stays out of Mutagen sync, and never lands in git. The login is **shared across all your projects** by default.
  * Installs the official Claude Code plugins (`context7`, `php-lsp`, `code-simplifier`, `code-review`, `security-guidance`) from the `claude-plugins-official` marketplace, idempotently.
* **`config.token.local.yaml`** — created on install (gitignored), forwards your OAuth token into the container.

> [!TIP]
> Want per-project logins instead of one shared login? In `config.claude.yaml`, change the `shared` path segment to `${DDEV_PROJECT:-default}`.

## The Claude Code image

The image source lives in this repo under [`claude-code/Dockerfile`](claude-code/Dockerfile) — a two-stage build that installs `@anthropic-ai/claude-code` via npm and extracts the single self-contained binary onto a slim Debian carrier (matching the glibc of DDEV's webserver image).

[`.github/workflows/build-image.yml`](.github/workflows/build-image.yml) builds and publishes it to Docker Hub:

* Runs daily at **06:00 UTC** (plus on manual dispatch and on pushes to `claude-code/**`).
* Resolves the latest version with `npm view @anthropic-ai/claude-code version`.
* Skips the build if that version's tag already exists on Docker Hub (no upstream release webhook exists, so this is a poll-and-gate).
* Builds multi-arch (`linux/amd64`, `linux/arm64`) and pushes `:latest` and `:<version>` tags.

> [!IMPORTANT]
> The workflow requires two repository secrets so it can push to Docker Hub:
> * `DOCKERHUB_USERNAME` — `zone1987`
> * `DOCKERHUB_TOKEN` — a Docker Hub access token with Read/Write/Delete

## Configuration & customization

All installed files contain a `#ddev-generated` marker. As long as you don't modify them, `ddev add-on get` can update them on a later install. If you edit a file, DDEV will leave your version in place.

* **Change which plugins are installed:** edit the `PLUGINS`/`MARKETPLACES` arrays in `config.claude.yaml`.
* **Pin a specific Claude version:** change `:latest` to `:<version>` in `web-build/Dockerfile.claude` and `ddev restart`.

## Removal

```bash
ddev add-on remove claude
```

This removes the add-on files and deletes the generated `config.token.local.yaml` (only if it still contains the `#ddev-generated` marker). Your persisted login in the global cache volume is left untouched.

## Resources

* [Claude Code documentation](https://docs.claude.com/en/docs/claude-code)
* [DDEV Documentation for Add-ons](https://docs.ddev.com/en/stable/users/extend/additional-services/)
* [DDEV Add-on Registry](https://addons.ddev.com/)
* Image inspiration: [avhulst/ddev-claude-image](https://github.com/avhulst/ddev-claude-image)

## Credits

**Contributed and maintained by [@zone1987](https://github.com/zone1987)**
