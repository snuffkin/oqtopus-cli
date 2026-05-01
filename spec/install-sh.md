# install.sh Specification

This document defines the behavior of `scripts/install.sh`.

## 1. Purpose

`install.sh` installs the `oqtopus` command into a user-local binary directory.

The installer is intended for the shell-based distribution flow:

```bash
curl -LsSf https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/scripts/install.sh | sh
```

## 2. Supported Platforms

- Linux: supported
- macOS: supported
- Windows: not supported

## 3. Command Examples

Install the latest version:

```bash
curl -LsSf https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/scripts/install.sh | sh
```

Install a specific version:

```bash
curl -LsSf https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/scripts/install.sh | sh -s -- --version v1.0.0
```

Install to a custom directory:

```bash
curl -LsSf https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/scripts/install.sh | sh -s -- --bin-dir ~/.local/bin
```

## 4. Supported Options

- `--bin-dir <path>`
  - Installation target directory.
  - Default: `~/.local/bin`

- `--version <tag>`
  - Version of the `oqtopus` command to install.
  - Example: `v1.0.0`

## 5. Default Behavior

If `--version` is not specified, `install.sh` MUST install the latest
available version of `oqtopus`.

The latest version is resolved by querying the available Git tags of the
`oqtopus-cli` repository via the public GitHub tags API and selecting the
newest version tag in `vX.Y.Z` format.

The installer should query:

```text
https://api.github.com/repos/oqtopus-team/oqtopus-cli/tags?per_page=100
```

For now, inspecting up to 100 tags is considered sufficient.
If the API response contains 100 tags, `install.sh` SHOULD print a warning that
additional tags may exist and the latest version resolution may be incomplete.

Suggested warning:

```text
Warning: the GitHub tags API returned 100 tags.
Additional tags may exist, so latest version resolution may be incomplete.
```

Version comparison MUST follow semantic version ordering, not plain string
comparison.

Examples:

- `v1.2.10` is newer than `v1.2.9`
- `v1.10.0` is newer than `v1.9.0`

Pre-release tags such as `v1.0.0-rc1` are excluded from the default latest
selection unless explicitly supported in the future.

If `--version` is explicitly specified, the installer uses the provided tag
as-is and does not apply semver filtering.

## 6. Download Source

`install.sh` downloads the source archive of the requested version from:

```text
https://github.com/oqtopus-team/oqtopus-cli/archive/refs/tags/<version>.tar.gz
```

The installer extracts the archive and installs `bin/oqtopus` from the
archive contents.

## 7. Install Behavior

1. Resolve the version to install.
   - If `--version` is specified, use that version.
   - Otherwise resolve the latest available tag.
2. Download the corresponding `.tar.gz` archive.
3. Extract the archive into a temporary directory.
4. Copy `bin/oqtopus` from the extracted archive into the target binary
   directory.
5. Mark the installed file as executable.
6. Attempt to install shell completion files in standard user-local locations.
7. Print the installed version, target path, and completion setup result.

## 8. Path Behavior

- If `--bin-dir` is not specified, install to `~/.local/bin`.
- If the target directory does not exist, `install.sh` MUST create it.
- If the target directory is not on `PATH`, `install.sh` SHOULD print a
  guidance message.

## 9. Shell Completion Installation

`install.sh` SHOULD install shell completion files when possible.

Completion files are generated from the installed command:

```bash
oqtopus completion bash
oqtopus completion zsh
oqtopus completion fish
```

Default user-local completion destinations:

```text
bash:
  ~/.local/share/bash-completion/completions/oqtopus

zsh:
  ~/.local/share/zsh/site-functions/_oqtopus

fish:
  ~/.config/fish/completions/oqtopus.fish
```

The installer MAY create these directories if they do not exist. Failure to
install completion files MUST NOT fail the entire installation if the `oqtopus`
binary itself was installed successfully.

`install.sh` MUST NOT modify shell startup files such as `.bashrc`, `.zshrc`,
`.profile`, or `config.fish` automatically.

After installation, `install.sh` MUST print what completion files were installed
and any manual activation guidance needed for the user's shell.

## 10. Required Tools

`install.sh` MAY rely on common shell tools available on Linux and macOS.

At minimum, the installer expects:

- `sh`
- `curl`
- `tar`
- `mktemp`
- `chmod`
- `mkdir`

## 11. Error Behavior

`install.sh` MUST exit with an error if:

- `curl` is not available
- `tar` is not available
- the requested version does not exist
- the latest version cannot be resolved
- the archive download fails
- `bin/oqtopus` is not found in the extracted archive
- the target directory is not writable

Completion installation failures should be reported as warnings rather than
fatal errors.

## 12. Non-Goals

`install.sh` does not:

- run `oqtopus init`
- install backend components
- modify backend environment configuration files
- modify shell startup files automatically
- require root privileges
