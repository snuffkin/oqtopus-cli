# OQTOPUS CLI Specification Status

This document tracks the current status of the OQTOPUS CLI specification.

## Decided

### Command structure

- The CLI entrypoint is `oqtopus`.
- v1.0.0 is provided as a shell-based CLI.
- A Rust implementation is planned for a future release, but is not part of
  the immediate v1.0.0 implementation.
- `oqtopus init` is used to create environments.
- `oqtopus backend` is used to manage backend components.
- `oqtopus backend versions` lists available stable component versions from
  remote GitHub tags without requiring backend environment validation.
- `oqtopus backend device-status` is used to show or update the local gateway
  device status file.
- `oqtopus completion <bash|zsh|fish>` prints shell completion scripts.
- `oqtopus version` and `oqtopus --version` print the installed CLI version.
- `oqtopus help` and `--help` are supported at the top level and for
  subcommands.
- `oqtopus backend` must fail if it is executed outside a directory created by
  `oqtopus init <env_name> --template backend`.

### Supported platforms

- Linux: supported
- macOS: supported
- Windows: not supported for now

### Repository layout

- `spec/`: specifications and implementation planning documents
- `README.md`: GitHub repository landing page; keep it lightweight with a
  brief overview and links.
- `docs/index.md`: root of the user-facing documentation.
- `docs/`: user-facing explanations, usage guides, and expanded documentation
  should be updated here.
- `templates/`: environment templates used by `oqtopus init`
- `scripts/`: helper scripts such as `install.sh`
- `bin/oqtopus`: current shell-based CLI entrypoint
- `src/`: future Rust implementation

User-facing documentation should be expanded under `docs/`, not in
`README.md`. Implementation notes, AI handoff material, and specification
updates should continue to live under `spec/`.

### Main specification documents

- `spec/oqtopus-cli.md`: product-level CLI specification
- `spec/implementation/copilot-implementation-brief.md`: implementation handoff for the shell PoC
- `spec/install-sh.md`: installer specification

### Backend environment structure

`oqtopus init <env_name> --template backend` creates:

```text
<env_name>/
  .metadata
  config/
    .env
  logs/
    core/
    sse_engine/
    mitigator/
    estimator/
    combiner/
    tranqu/
    gateway/
  pids/
  sse_work/
```

- `config/` contains configuration files for each microservice and
  environment variables used when launching backend processes.
- `.metadata` contains `env_name=<env_name>` between `install_root` and
  `env_root`.
- `env_name` must match `^[a-z0-9][a-z0-9_.-]*$` because it is used in
  Docker-related configuration values.
- `oqtopus init` replaces `{{ env_name }}` placeholders in `config/.env` with
  the validated environment name.
- `config/.env` contains environment variables for launched backend processes.
- `logs/` contains one initially empty subdirectory for each managed service,
  using `logs/<component>/`.
- `pids/` contains PID files named `<component>.pid`.
- `sse_work/` is an initially empty host-side working directory for dynamically
  launched `sse_runtime` Docker containers.
- The default `SSE_HOST_WORK_PATH` in `config/.env` points to `sse_work`
  relative to `env_root`.
- Runtime-only empty directories such as `pids/`, `logs/<component>/`, and
  `sse_work/` are created by `oqtopus init`; they are not represented as
  template files.

### Backend components

The backend components currently in scope are:

- `engine`
- `tranqu`
- `gateway`

### Install behavior

- `oqtopus backend versions <engine|tranqu|gateway>` lists available stable
  versions for a component.
- `versions` uses GitHub tags, includes only `vX.Y.Z`, sorts by semantic
  version descending, and does not support `all`.
- When `versions` is run from a valid backend environment, it also marks the
  current `.metadata` binding with `*` and locally available release
  directories with `(installed)`.
- `oqtopus backend install all` installs `engine`, `tranqu`, and `gateway`.
- `install all` resolves the latest stable version independently for each
  component.
- `install all` does not accept a version argument.
- `install all` runs in the order `engine`, `tranqu`, `gateway`.
- If `install all` fails partway through, the command stops and leaves already
  installed component bindings in place; rollback is not performed
  automatically.
- `oqtopus backend install` updates the environment's `.metadata` binding.
- `oqtopus backend install` does not modify files under `<env_root>/config/`.
- If a component version requires configuration changes, the user must update
  configuration files manually.
- Latest version resolution uses the GitHub tags API and selects the newest
  stable semantic version tag in `vX.Y.Z` format.
- If the target release directory already exists and contains `.venv`, it is
  reused. For `engine`, the install is considered reusable only when the
  required service subprojects have `.venv` directories.
- If the target release directory exists without `.venv`, it is treated as an
  incomplete installation and recreated.
- A `--force` reinstall option is deferred to a future release.
- Installing `engine` also builds the `sse_runtime` Docker image from
  `<install_root>/engine-<version>/sse_runtime/Dockerfile`.
- Installing `engine` runs `uv sync --frozen --no-dev` for the monorepo
  subprojects `core`, `combiner`, `estimator`, and `mitigator`.
- `sse_engine` runs from the installed `engine` `core` project.
- The `sse_runtime` image build uses `SSE_CONTAINER_IMAGE` from
  `<env_root>/config/.env`.
- The `sse_runtime` image build passes the current user's UID and GID from
  `id -u` and `id -g` as Docker build arguments.
- If Docker is unavailable, the Dockerfile is missing, required variables are
  missing, or the Docker build fails, `oqtopus backend install engine` fails
  without updating `engine_version`.

### Uninstall behavior

- `oqtopus backend uninstall` removes the target release directory without
  checking whether the version is referenced by the current environment or
  another environment.
- `oqtopus backend uninstall` does not rewrite `.metadata`.

### Template retrieval

- `oqtopus init` always downloads templates from GitHub.
- `oqtopus init` does not use a local cache.
- For v1.0.0, the shell CLI also downloads templates from GitHub; it does not
  create the backend template from hard-coded local shell logic.
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
- `templates/backend/config/.env` is distributed as the initial environment
  variable file and must not contain secrets.
- `templates/backend/config/.env` may contain `{{ env_name }}` placeholders
  that are rendered during `oqtopus init`.
- Runtime-only empty directories such as `pids/`, `logs/<component>/`, and
  `sse_work/` are created by `oqtopus init`, not copied from `templates/`.
- The expected `templates/backend/config/` tree is defined in
  `spec/oqtopus-cli.md`.

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
- The installer should place shell completion files in standard user-local
  locations when possible.
- The installer must not modify shell startup files automatically.
- Completion installation failures are warnings, not fatal install errors.

### Help, version, and completion behavior

- Help commands are available as both `help` subcommands and `--help` flags.
- Help and version commands do not require backend environment validation.
- `oqtopus version` and `oqtopus --version` print the CLI version.
- `oqtopus backend info` remains the backend environment information command.
- Completion covers commands, subcommands, flags, templates, and component
  names.
- Version strings are not completed in v1.0.0.
- Completion must not make network requests.

### Error message guidance

- Error messages should be conventional and user-friendly.
- Error messages should explain what failed and why.
- If exact wording is unclear, it should be confirmed before finalizing.
- `spec/oqtopus-cli.md` contains representative error message examples for
  install, uninstall, start, and stop failures.

### Process lifecycle behavior

- `oqtopus backend start all` starts all managed services.
- `oqtopus backend stop all` stops all managed services.
- `oqtopus backend restart all` restarts all managed services.
- Start order is `gateway`, `tranqu`, `mitigator`, `estimator`, `combiner`,
  `sse_engine`, `core`.
- Stop order is the reverse of start order.
- If `start all` fails partway through, the command stops and leaves already
  started services running; rollback is not performed automatically.
- If `stop all` fails for one service, the command continues attempting to stop
  remaining services and exits non-zero.
- `restart <service>` stops the service and starts it again only if stop
  succeeds.
- `restart all` stops all services first; if every stop succeeds, it starts all
  services using the normal start order.
- If a PID file exists and the recorded process is alive, `start` fails.
- If a PID file exists and the recorded process is not alive, the PID file is
  treated as stale and removed.
- When `start` loads variables from `config/.env`, they are applied only to the
  launched `uv` process environment.
- These variables are not persisted as global or user-shell environment
  variables.
- `stop` sends `TERM` to the recorded PID.
- `stop` waits up to 5 seconds for the process to exit.
- If the process exits, the PID file is removed.
- If the process is still running after 5 seconds, `stop` fails.
- `stop` does not send `KILL` automatically.
- PID ownership or command matching is not required in the current design.
- Runtime stdout/stderr is redirected to `/dev/null` by default.
- `oqtopus backend start <service> --foreground` keeps stdout/stderr attached
  to the terminal for a single service and waits for the service process to
  exit.
- `--foreground` is not supported with `start all`.
- The CLI does not create log files itself; application log files are created
  by backend applications according to their `logging.yaml` configuration.
- Placeholder processes are development/test-only and are not the default
  v1.0.0 user-facing behavior.
- If a required version binding or installed release directory is missing,
  `start` fails with a clear error.

### Update semantics

- `oqtopus backend update <component>` is equivalent to
  `oqtopus backend install <component> <latest>`.
- `update` does not perform special processing beyond install plus metadata
  update.

### Prune

- `oqtopus backend prune` is not provided in v1.0.0.
- The CLI does not maintain `environments.json`.

### Runtime model

- All managed backend processes are run via `uv`.
- `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are launched
  from the installed `engine` release.
- `core` and `sse_engine` use the `engine` `core` uv project; `mitigator`,
  `estimator`, and `combiner` use their matching engine uv projects.
- `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are
  independent managed services.
- `tranqu` is launched from the installed `tranqu` release.
- `gateway` is managed as a Python/uv component and launched from the installed
  `gateway` release.
- Exact `uv run` start commands are now defined in `spec/oqtopus-cli.md`.

### Device status behavior

- `oqtopus backend device-status show` prints
  `<env_root>/config/gateway/device_status`.
- `oqtopus backend device-status active` writes `active`.
- `oqtopus backend device-status inactive` writes `inactive`.
- `oqtopus backend device-status maintenance` writes `maintenance`.
- Valid device status values are only `active`, `inactive`, and `maintenance`.
- Device status commands run after standard backend environment validation.
- Device status commands do not require `gateway` to be running.
- Device status commands directly update the local configuration file and do
  not call scripts from the installed `gateway` release.

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
