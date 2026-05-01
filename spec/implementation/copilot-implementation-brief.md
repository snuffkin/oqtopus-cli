# OQTOPUS CLI Copilot Implementation Brief

This document is the implementation handoff for GitHub Copilot.
The product-level behavior is defined in `oqtopus-cli.md`.

## Goal

Build a shell-based Proof of Concept for the OQTOPUS local backend CLI.
The PoC should validate command behavior, metadata handling, process lifecycle,
and installation layout before the CLI is rewritten in Rust.
The command surface for both the PoC and the Rust implementation is a single
top-level CLI named `oqtopus`.

## Repository Layout

Use the following repository layout:

- `spec/`: specifications and implementation planning documents
- `README.md`: GitHub repository landing page; keep it lightweight with a
  brief overview and links
- `docs/index.md`: root of the user-facing documentation
- `docs/`: user-facing explanations, usage guides, and expanded documentation
- `templates/`: environment templates used by `oqtopus init`
- `scripts/`: helper scripts such as `install.sh`
- `bin/oqtopus`: shell-based CLI entrypoint for the current PoC
- `src/`: future Rust implementation

When adding user-facing explanations or usage documentation, update files under
`docs/`. Keep implementation notes, AI handoff material, and specification
updates under `spec/`.

## Implementation Phases

### Phase 1: Shell PoC

Implement one executable shell script:

- `bin/oqtopus`

The PoC should be portable across Linux and macOS.
Windows support can be deferred to the Rust implementation.

The scripts may use common POSIX utilities plus:

- `bash`
- `curl`
- `tar`
- `jq`
- `uv`
- `docker` when installing `engine`

If a required command is missing, print a clear error and exit non-zero.

### Phase 2: Rust CLI

After the PoC behavior is validated, reimplement the CLI in Rust using:

- `clap` for command parsing
- `reqwest` for network operations
- shared metadata parsing code
- subprocess execution for `uv`
- cross-platform process management

Do not start Phase 2 until Phase 1 behavior is stable.

## Phase 1 Scope

### `oqtopus init`

Command:

```bash
oqtopus init <env_name> --template backend
```

Required behavior:

1. Create `<env_name>/`.
2. Create:
   - `.metadata`
   - `config/`
   - `config/.env`
   - `logs/`
   - `pids/`
   - `sse_work/`
3. Resolve `install_root`.
4. Write `install_root`, `env_name`, and absolute `env_root` to `.metadata`.
5. Register the environment path in the central registry.
6. Do not create a per-environment CLI binary.
   The `oqtopus` executable is expected to be installed in a directory on
   the user's `PATH`, so it can be run from anywhere.
7. Create runtime directories dynamically. These are not copied from the
   template repository:
   - `pids/`
   - `sse_work/`
   - `logs/core/`
   - `logs/sse_engine/`
   - `logs/mitigator/`
   - `logs/estimator/`
   - `logs/combiner/`
   - `logs/tranqu/`
   - `logs/gateway/`

Before creating the environment directory, validate `env_name` with:

```text
^[a-z0-9][a-z0-9_.-]*$
```

The name must be Docker-safe because it is used to render Docker-related
configuration values. After downloading the template, replace `{{ env_name }}`
placeholders in `<env_name>/config/.env` with the validated environment name.

Only the `backend` template is required for the PoC.
The `cloud` template should return a clear "not implemented" error.

For v1.0.0, `oqtopus init` MUST download the backend template from GitHub using
the `templates/backend/` tree from the `main` branch. It MUST NOT create the
template from hard-coded local shell logic.

### `oqtopus backend`

All commands must first run environment validation:

1. `.metadata` exists in the current directory.
2. `template=backend`.
3. `env_root` in `.metadata` exactly matches the current absolute directory.
4. If the current directory is missing from the registry, add it.
5. If the current directory is not a backend environment created by
   `oqtopus init <env_name> --template backend`, exit with a clear error.

Required commands:

```bash
oqtopus init <env_name> --template backend
oqtopus help
oqtopus --help
oqtopus version
oqtopus --version
oqtopus completion <bash|zsh|fish>
oqtopus backend versions <engine|tranqu|gateway>
oqtopus backend install <engine|tranqu|gateway> [version]
oqtopus backend install all
oqtopus backend uninstall <engine|tranqu|gateway> <version>
oqtopus backend update <engine|tranqu|gateway>
oqtopus backend prune
oqtopus backend start <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
oqtopus backend stop <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
oqtopus backend restart <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
oqtopus backend status
oqtopus backend device-status show
oqtopus backend device-status active
oqtopus backend device-status inactive
oqtopus backend device-status maintenance
oqtopus backend info
```

The script should parse subcommands as:

```text
oqtopus <init|backend|completion|version|help> ...
```

## Help, Version, and Completion

The CLI must support help at the top level and for subcommands:

```bash
oqtopus help
oqtopus --help
oqtopus init help
oqtopus init --help
oqtopus backend help
oqtopus backend --help
oqtopus backend install help
oqtopus backend install --help
```

The same pattern applies to all backend subcommands. Help and version commands
must not require backend environment validation.

`oqtopus version` and `oqtopus --version` print the installed CLI version, for
example:

```text
oqtopus v1.0.0
```

`oqtopus backend info` remains the backend environment information command.

`oqtopus completion <bash|zsh|fish>` prints shell completion scripts to stdout.
Completion must cover commands, subcommands, flags, templates, and component
names, but must not complete version strings or make network requests.

Completion must include `device-status` under `oqtopus backend`, and `show`,
`active`, `inactive`, and `maintenance` under `oqtopus backend device-status`.
Completion for `start`, `stop`, and `restart` must include `all` in addition
to individual service names.
Completion for `install` must include `all` in addition to `engine`, `tranqu`,
and `gateway`.
Completion for `versions` must include `engine`, `tranqu`, and `gateway`.

## Component Repository Mapping

Use this mapping for downloads and latest-version lookups:

| Component | GitHub repository |
|-----------|-------------------|
| engine    | `oqtopus-team/oqtopus-engine` |
| tranqu    | `oqtopus-team/tranqu-server` |
| gateway   | `oqtopus-team/device-gateway` |

Release archive URL pattern:

```text
https://github.com/<owner>/<repo>/archive/refs/tags/<version>.tar.gz
```

Latest version lookup:

```text
https://api.github.com/repos/<owner>/<repo>/tags?per_page=100
```

Select the newest stable semantic version tag in `vX.Y.Z` format. Pre-release
tags are excluded from automatic latest selection. If exactly 100 tags are
returned, print a warning that additional tags may exist and latest version
resolution may be incomplete.

## Path Rules

### Data Directory

Resolve `install_root` as:

1. `$XDG_DATA_HOME/oqtopus/backend/releases` if `XDG_DATA_HOME` is set.
2. Otherwise `~/.local/share/oqtopus/backend/releases`.

### Config Directory

Do not maintain a central backend environment registry.
The CLI must not create or update `environments.json`.

## Metadata Format

`.metadata` is a simple `key=value` file.
It is not TOML, YAML, or JSON.

Required fields:

```text
template=backend
install_root=<absolute path>
env_name=<env_name>
env_root=<absolute path>
created_at=<ISO-8601 datetime>
```

Optional bindings:

```text
engine_version=<version>
tranqu_version=<version>
gateway_version=<version>
```

When updating a binding, preserve unrelated fields.
The update should avoid leaving a partially written `.metadata`.

## Install Behavior

## Versions Behavior

Command:

```bash
oqtopus backend versions <component>
```

Required behavior:

1. Does not run backend environment validation.
2. Supports `engine`, `tranqu`, and `gateway`.
3. Does not support `all`.
4. Queries the component repository tags API with `?per_page=100`.
5. Includes only stable semantic version tags in `vX.Y.Z` format.
6. Sorts versions by semantic version descending.
7. If the current directory is a valid backend environment, reads `.metadata`
   and `install_root` as optional context.
8. In a valid backend environment, marks the current version with `*`.
9. In a valid backend environment, marks locally available release directories
   with `(installed)`.
10. Includes current or locally installed versions even if they are not present
    in remote stable tags, and marks them with `(not in remote tags)`.
11. Prints:

```text
engine:
* v2.0.3 (installed)
  v2.0.2 (installed)
  v2.0.1
```

If no stable versions are found, exit non-zero with a clear error.

## Install Behavior

Command:

```bash
oqtopus backend install <component> [version]
oqtopus backend install all
```

Required behavior:

1. Resolve version:
   - Use the provided version if present.
   - Otherwise fetch GitHub tags and select the newest stable semantic version
     tag in `vX.Y.Z` format.
   - For `install all`, resolve the latest stable version independently for
     `engine`, `tranqu`, and `gateway`.
2. Create `<install_root>/<component>-<version>/`.
3. Download the release archive.
4. Extract it into the target directory with the top-level archive directory removed.
5. Run `uv sync`.

For `tranqu` and `gateway`:

```bash
uv sync --frozen --no-dev --project <install_root>/<component>-<version>/
```

For `engine`, the repository is a monorepo. Run `uv sync --frozen --no-dev` for
each engine service project:

```bash
uv sync --frozen --no-dev --project <install_root>/engine-<version>/core
uv sync --frozen --no-dev --project <install_root>/engine-<version>/combiner
uv sync --frozen --no-dev --project <install_root>/engine-<version>/estimator
uv sync --frozen --no-dev --project <install_root>/engine-<version>/mitigator
```

`sse_engine` is launched from the `core` project.

6. If the component is `engine`, build the `sse_runtime` Docker image from:

```text
<install_root>/engine-<version>/sse_runtime/Dockerfile
```

Load `SSE_CONTAINER_IMAGE` from `<env_root>/config/.env`. Use the current
user's UID and GID from `id -u` and `id -g` for Docker build arguments.
Run the equivalent of:

```bash
docker build <install_root>/engine-<version>/sse_runtime \
  -t <SSE_CONTAINER_IMAGE> \
  --build-arg UID=$(id -u) \
  --build-arg GID=$(id -g)
```

If Docker is unavailable, `id` is unavailable, the Dockerfile is missing,
`SSE_CONTAINER_IMAGE` is missing from `config/.env`, or the build fails,
`oqtopus backend install engine` must fail and must not update
`engine_version`.

7. Update the component version binding in `.metadata`.

If the target directory already exists and contains `.venv`, reuse it and update
the metadata binding. If the target directory exists but does not contain
`.venv`, treat it as an incomplete installation: remove that component-version
directory and download/extract/sync it again. A `--force` option is deferred to a
future release.

For `oqtopus backend install all`, install components in this order:

```text
engine
tranqu
gateway
```

`install all` does not accept a version argument. If installing any component
fails, stop immediately and return non-zero. Components already installed
successfully by the same command keep their metadata bindings; do not roll them
back automatically. The failed component must not update its metadata binding.

## Start Behavior

Command:

```bash
oqtopus backend start <component|all>
```

Required behavior:

1. Check `pids/<component>.pid`.
2. If the PID is alive, exit with an error.
3. If the PID file is stale, delete it.
4. Load valid `key=value` lines from `config/.env`.
5. Start the component in the background.
6. Write the process PID to `pids/<component>.pid`.
7. Redirect stdout/stderr to `/dev/null`. The CLI must not create log files
   itself; application log files are created according to `logging.yaml`.

For v1.0.0, placeholder processes are development/test-only and MUST NOT be the
default user-facing behavior. If the required component version binding is
missing from `.metadata`, or the installed release directory is missing, `start`
MUST fail with a clear error.

`core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are launched from
the installed `engine` release. Because `engine` is a monorepo, `core` and
`sse_engine` use the installed `core` uv project, while `mitigator`,
`estimator`, and `combiner` use their matching uv projects. `gateway` is
managed as a Python/uv component in the same style as `tranqu`.

`oqtopus backend start all` starts all managed services in this order:

```text
gateway
tranqu
mitigator
estimator
combiner
sse_engine
core
```

If starting any service fails, stop immediately and return non-zero. Services
already started by the same command are left running; do not roll them back
automatically.

For `tranqu`, use the installed version from `.metadata`:

```bash
uv run --project <install_root>/tranqu-<version> python -m tranqu_server.proto.service \
  -c <env_root>/config/tranqu/config.yaml \
  -l <env_root>/config/tranqu/logging.yaml
```

If `tranqu_version` is missing, print a clear error.

## Stop Behavior

Command:

```bash
oqtopus backend stop <component|all>
```

Required behavior:

1. Read `pids/<component>.pid`.
2. If no PID file exists, report the component as stopped.
3. If the PID is alive, send `TERM`.
4. Wait briefly for shutdown.
5. If the process is still alive, return a clear error.
6. Remove the PID file only after the process has stopped or if it was stale.

For `oqtopus backend stop all`, stop all managed services in reverse start
order:

```text
core
sse_engine
combiner
estimator
mitigator
tranqu
gateway
```

If stopping any service fails, continue attempting to stop remaining services
and exit non-zero after reporting the failure.

## Restart Behavior

Command:

```bash
oqtopus backend restart <component|all>
```

For a single service, restart stops the service and then starts it again. If
stop fails, restart fails and does not start the service.

For `restart all`, stop all services using the same behavior as `stop all`.
Only if every stop succeeds, start all services using the same behavior as
`start all`.

## Status Behavior

Command:

```bash
oqtopus backend status
```

Print only process status:

```text
core: Stopped
sse_engine: Stopped
mitigator: Running (PID 12345)
estimator: Stopped
combiner: Stopped
tranqu: Stopped
gateway: Stopped
```

## Device Status Behavior

Commands:

```bash
oqtopus backend device-status show
oqtopus backend device-status active
oqtopus backend device-status inactive
oqtopus backend device-status maintenance
```

Required behavior:

1. Run standard backend environment validation.
2. Operate on `<env_root>/config/gateway/device_status`.
3. If the file does not exist, fail with a clear error.
4. `show` prints the current file contents.
5. `active`, `inactive`, and `maintenance` write the matching value to the
   file.
6. The command does not require `gateway` to be running.
7. The command directly updates the local configuration file and does not call
   scripts from the installed `gateway` release.

## Info Behavior

Command:

```bash
oqtopus backend info
```

Print:

1. Environment metadata.
2. Component version bindings recorded in `.metadata`.
3. Expanded paths recorded in `.metadata`.

Do not print Python executable or Python version information. Managed services
run through component-specific `uv` environments, so a single process-level
Python path would be misleading.

Do not print installed release directories. `.metadata` is the source of truth
for the versions selected by the current environment.

## Decided Implementation Questions

- `oqtopus backend install` updates `.metadata` automatically.
- `oqtopus backend uninstall` removes the selected release directory without
  checking whether another environment references it.
- `oqtopus backend prune` is not provided in v1.0.0.
- The CLI does not maintain `environments.json`.
- Placeholder processes are development/test-only, not default v1.0.0 behavior.
- `oqtopus init` downloads templates from GitHub for v1.0.0.
- `gateway` is managed as a Python/uv component for v1.0.0.
- Runtime stdout/stderr is redirected to `/dev/null`.
- The CLI does not create log files itself; application log files are created
  according to `logging.yaml`.
- `templates/backend/config/.env` is distributed as the initial environment
  variable file. It must not contain secrets.
- `oqtopus init` creates `sse_work/` as an empty host-side work directory for
  `sse_runtime` containers. It is not copied from `templates/`.
- `oqtopus backend install engine` builds the `sse_runtime` Docker image from
  the installed engine release before updating `engine_version`.
- `oqtopus backend install all` installs `engine`, `tranqu`, and `gateway`
  using each component's independently resolved latest stable version.
- `oqtopus backend device-status` directly shows or updates
  `config/gateway/device_status`.
- `oqtopus backend start all`, `oqtopus backend stop all`, and
  `oqtopus backend restart all` operate on all managed services.
- `oqtopus version` / `oqtopus --version` report the CLI version.
- `oqtopus backend info` reports backend environment information.
- `oqtopus completion <bash|zsh|fish>` provides shell completion without
  version completion or network access.

## Acceptance Criteria

The PoC is complete when the following manual flow works:

```bash
bin/oqtopus init demo --template backend
cd demo
oqtopus backend status
oqtopus backend start core
oqtopus backend status
oqtopus backend stop core
oqtopus backend status
oqtopus backend info
```

Expected result:

- `demo/.metadata` contains absolute `env_root` and `install_root`.
- `demo/sse_work/` exists as an initially empty host-side work directory.
- `demo/logs/core/` and other managed service log directories exist.
- `core` transitions from stopped to running to stopped.
- PID files are created and removed correctly.
- The CLI does not create log files; backend applications create logs according
  to their `logging.yaml` configuration.
- Running `oqtopus backend ...` outside `demo/` fails validation.
- The same `oqtopus` executable is callable both inside and outside the
  environment because it is installed on `PATH`.

## Open Questions

No v1.0.0-blocking open questions are currently known. Rust-specific internal
structure remains deferred until the future Rust implementation phase.
