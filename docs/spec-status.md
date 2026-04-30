# OQTOPUS CLI Specification Status

This document tracks the current status of the OQTOPUS CLI specification.

## Decided

### Command structure

- The CLI entrypoint is `oqtopus`.
- `oqtopus init` is used to create environments.
- `oqtopus backend` is used to manage backend components.
- `oqtopus backend` must fail if it is executed outside a directory created by
  `oqtopus init <env_name> --template backend`.

### Supported platforms

- Linux: supported
- macOS: supported
- Windows: not supported for now

### Repository layout

- `docs/`: specifications and implementation planning documents
- `templates/`: environment templates used by `oqtopus init`
- `scripts/`: helper scripts such as `install.sh`
- `bin/oqtopus`: current shell-based CLI entrypoint
- `src/`: future Rust implementation

### Main specification documents

- `docs/oqtopus-cli.md`: product-level CLI specification
- `docs/implementation/copilot-implementation-brief.md`: implementation handoff for the shell PoC
- `docs/install-sh.md`: installer specification

### Backend environment structure

`oqtopus init <env_name> --template backend` creates:

```text
<env_name>/
  .metadata
  .env
  config/
  logs/
  pids/
```

- `config/` contains configuration files for each microservice.
- `logs/` contains one subdirectory for each managed service.
- `pids/` contains PID files named `<component>.pid`.

### Backend components

The backend components currently in scope are:

- `engine`
- `tranqu`
- `gateway`

### Install behavior

- `oqtopus backend install` updates the environment's `.metadata` binding.
- `oqtopus backend install` does not modify files under `<env_root>/config/`.
- If a component version requires configuration changes, the user must update
  configuration files manually.

### Uninstall behavior

- `oqtopus backend uninstall` must fail if the target version is referenced by
  any registered environment.
- `oqtopus backend uninstall` does not rewrite `.metadata`.

### Template retrieval

- `oqtopus init` always downloads templates from GitHub.
- `oqtopus init` does not use a local cache.
- If the network is unavailable or template download fails, `oqtopus init`
  exits with an error.
- Template retrieval uses the `main` branch.
- Template base URL:

   ```text
   https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/templates/<template_type>/
   ```

### Template contents

- Files committed under `templates/backend/` are copied into the root of the
  environment created by `oqtopus init`.
- `.metadata` is not stored as a literal template file.
- `.metadata` is generated dynamically by `oqtopus init`.
- Service configuration files are stored under `templates/backend/`.
- Required initial config files are whatever is present under
  `templates/backend/` at init time.
- The expected `templates/backend/config/` tree is defined in
  `docs/oqtopus-cli.md`.

### Installer behavior

- The installer is `scripts/install.sh`.
- Installation is intended to be triggered by:

   ```bash
   curl -LsSf https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/scripts/install.sh | sh
   ```

- `install.sh` supports:
  - `--bin-dir <path>`
  - `--version <tag>`
- Default install location: `~/.local/bin`
- If `--version` is omitted, install the latest available version.
- Latest version resolution only considers stable semantic version tags in
  `vX.Y.Z` format.
- Pre-release tags are excluded from automatic latest selection.
- If `--version` is explicitly specified, the installer uses the provided tag
  as-is and does not apply semver filtering.
- Latest version resolution should query the public GitHub tags API and select
  the newest stable semantic version tag.
- The installer should query:

   ```text
   https://api.github.com/repos/oqtopus-team/oqtopus-cli/tags?per_page=100
   ```
- For now, inspecting up to 100 tags is considered sufficient.
- If the tags API response contains 100 tags, the CLI should print a warning
  that additional tags may exist and latest version resolution may be
  incomplete.

- The installer downloads:

   ```text
   https://github.com/oqtopus-team/oqtopus-cli/archive/refs/tags/<version>.tar.gz
   ```

- The installer extracts `bin/oqtopus` from the archive contents.

### Error message guidance

- Error messages should be conventional and user-friendly.
- Error messages should explain what failed and why.
- If exact wording is unclear, it should be confirmed before finalizing.
- `docs/oqtopus-cli.md` contains representative error message examples for
  install, uninstall, start, and stop failures.

### Process lifecycle behavior

- If a PID file exists and the recorded process is alive, `start` fails.
- If a PID file exists and the recorded process is not alive, the PID file is
  treated as stale and removed.
- When `start` loads variables from `.env`, they are applied only to the
  launched `uv` process environment.
- These variables are not persisted as global or user-shell environment
  variables.
- `stop` sends `TERM` to the recorded PID.
- `stop` waits up to 5 seconds for the process to exit.
- If the process exits, the PID file is removed.
- If the process is still running after 5 seconds, `stop` fails.
- `stop` does not send `KILL` automatically.
- PID ownership or command matching is not required in the current design.

### Update semantics

- `oqtopus backend update <component>` is equivalent to
  `oqtopus backend install <component> <latest>`.
- `update` does not perform special processing beyond install plus metadata
  update.

### Prune safety

- `oqtopus backend prune` requires confirmation before deletion.
- The CLI should show the list of deletion targets before prompting.
- `oqtopus backend prune --yes` skips the interactive confirmation.
- PoC behavior and Rust behavior should match.
- The interactive prompt should follow a conventional `Proceed? [y/N]:`
  pattern.

### Runtime model

- All managed backend processes are run via `uv`.
- `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are launched
  from the installed `engine` release.
- `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are
  independent managed services.
- `tranqu` is launched from the installed `tranqu` release.
- `gateway` is launched from the installed `gateway` release.
- Exact `uv run` start commands are now defined in `docs/oqtopus-cli.md`.

## Undecided

## Deferred

### Rust internal structure

- Detailed Rust module layout under `src/`
- Shared crate boundaries
- Internal API organization

### OQTOPUS local cloud

- Full `cloud` template behavior
- `oqtopus cloud` command surface
- Cloud-specific configuration and lifecycle management

### Windows support

- Windows-specific path handling
- Windows process management
- Windows installer behavior

### Advanced UX

- Machine-readable output such as `--json`
- Richer diagnostics such as `oqtopus backend doctor`
- Config validation or migration helper commands
