[![add-on registry](https://img.shields.io/badge/DDEV-Add--on_Registry-blue)](https://addons.ddev.com)
[![tests](https://github.com/zone1987/ddev-claude/actions/workflows/tests.yml/badge.svg?branch=main)](https://github.com/zone1987/ddev-claude/actions/workflows/tests.yml?query=branch%3Amain)
[![last commit](https://img.shields.io/github/last-commit/zone1987/ddev-claude)](https://github.com/zone1987/ddev-claude/commits)
[![release](https://img.shields.io/github/v/release/zone1987/ddev-claude)](https://github.com/zone1987/ddev-claude/releases/latest)

# DDEV Claude

## Overview

This add-on makes [Claude Code](https://docs.claude.com/en/docs/claude-code) available **inside** your [DDEV](https://ddev.com/) project's web container. After installing it, `ddev claude` runs Claude Code right next to your project's code and PHP tooling — no host installation needed.

It pulls a tiny prebuilt single-binary image (`docker.io/zone1987/claude-code`) via one `COPY --from=...` layer, so install is a fast copy instead of an `npm install`. The image is rebuilt daily and only republished when a new Claude Code version ships on npm.

## Installation

```bash
ddev add-on get zone1987/ddev-claude
ddev restart
```

## Authentication

Claude Code uses a long-lived OAuth token (valid for one year).

1. On your **host**, generate a token: `claude setup-token`
2. Export it in your shell (`~/.zshrc`): `export CLAUDE_CODE_OAUTH_TOKEN="<your-token>"`, then `source ~/.zshrc`
3. Apply it: `ddev restart`

The add-on creates a gitignored `.ddev/config.token.local.yaml` that forwards `CLAUDE_CODE_OAUTH_TOKEN` into the container. You can also just log in interactively the first time you run `ddev claude` — the login persists in DDEV's global cache volume and survives restarts.

## Usage

| Command | Description |
| ------- | ----------- |
| `ddev claude` | Start Claude Code in the web container |
| `ddev claude --version` | Print the installed Claude Code version |
| `ddev claude -p "…"` | Run Claude Code with arguments (passed through) |
| `ddev ssh` then `claude` | Open a container shell and run Claude manually |

## Configuration

All installed files carry a `#ddev-generated` marker, so `ddev add-on get` can update them later — unless you've edited them, in which case your version is kept.

* **Plugins:** edit the `PLUGINS` / `MARKETPLACES` arrays in `.ddev/config.claude.yaml`. By default the official plugins `context7`, `php-lsp`, `code-simplifier`, `code-review` and `security-guidance` are installed automatically on `post-start`.
* **Shared vs. per-project login:** auth/state is shared across all projects by default (`/mnt/ddev-global-cache/claude-code/shared`). For per-project isolation, replace `shared` with `${DDEV_PROJECT:-default}` in `.ddev/config.claude.yaml`.
* **Pin a Claude version:** change `:latest` to `:<version>` in `.ddev/web-build/Dockerfile.claude`, then `ddev restart`.

## Removal

```bash
ddev add-on remove claude
```

Removes the add-on files and the generated `config.token.local.yaml`. The persisted login in the global cache volume is left untouched.

## Credits

**Contributed and maintained by [@zone1987](https://github.com/zone1987)**
