# OQTOPUS CLI Specification

This document defines the official behavior and interface of the
OQTOPUS Local Backend Environment CLI.
For v1.0.0, the CLI is provided as a shell-based executable named `oqtopus`.
A Rust implementation is planned for a future release after the shell behavior
is validated.

## 1. Overview

OQTOPUS CLI provides a single top-level command `oqtopus`:

- `oqtopus init`
  - Creates new local environments from templates.
- `oqtopus backend`
  - Manages backend components: `engine`, `tranqu`, and `gateway`.
  - Supports versions / install / uninstall / update / prune.
  - Starts, stops, and restarts backend processes.
  - Validates environment integrity before executing commands.
- `oqtopus completion`
  - Prints shell completion scripts for supported shells.
- `oqtopus version`
  - Prints the installed OQTOPUS CLI version.
- `oqtopus help`
  - Prints help for the top-level command and subcommands.

Current support target:

- Linux/macOS: supported
- Windows: not supported yet

## 2. Directory Structure (backend template)

`oqtopus init <env_name> --template backend` creates:

```text
<env_name>/
  .metadata       # environment-specific metadata
  config/         # configuration files and environment variables
    .env          # environment variables for launched backend processes
  logs/           # per-service log directories such as logs/core/
  pids/           # PID files such as core.pid
  sse_work/       # host-side work directory for dynamically launched sse_runtime containers
```

No backend code is installed at init time.

The `logs/` directory contains one initially empty subdirectory for each managed
service, such as `logs/core/`, `logs/sse_engine/`, and `logs/tranqu/`.

The `pids/` directory stores PID files using the `<component>.pid` naming
convention, such as `core.pid`, `sse_engine.pid`, and `tranqu.pid`.

The `sse_work/` directory is an initially empty host-side working directory used
by dynamically launched `sse_runtime` Docker containers.
The default `SSE_HOST_WORK_PATH` in `config/.env` SHOULD point to this directory
relative to `env_root`.

## 3. .metadata Specification

`.metadata` is a simple `key=value` file (NOT TOML).

### 3.1 Required fields

`install_root` is resolved as follows:

1. Use `$XDG_DATA_HOME/oqtopus/backend/releases` if `XDG_DATA_HOME` is set.
2. Otherwise use `~/.local/share/oqtopus/backend/releases`.

```text
template=backend
install_root=<absolute path to the backend releases directory>
env_name=<environment directory name>
env_root=<absolute path to this environment>
created_at=<ISO datetime>
```

## 3.2 Optional fields (used when bound to versions)

```text
engine_version=v0.3.2
tranqu_version=v0.1.4
gateway_version=v0.2.1
```

## 4. `oqtopus init` Specification

### 4.1 Command

```text
oqtopus init <env_name> --template <template>
```

| template | Description                                       |
|----------|---------------------------------------------------|
| backend  | Local backend environment (engine/tranqu/gateway) |
| cloud    | Cloud client environment (future)                 |

### 4.2 Backend template behavior

`env_name` MUST be Docker-safe because it is used to derive Docker-related
configuration values such as `SSE_CONTAINER_IMAGE` and
`SSE_CONTAINER_NETWORK`.

Allowed `env_name` pattern:

```text
^[a-z0-9][a-z0-9_.-]*$
```

If `env_name` does not match this pattern, `oqtopus init` MUST fail before
creating the environment directory.

1. Creates the env directory.
2. Downloads the backend template from the official GitHub repository.
3. Copies the contents of `templates/backend/` into the root of the new environment directory.
4. Generates `.metadata` dynamically for the new environment.
5. Resolves the `install_root` path (respecting $XDG_DATA_HOME) and writes it
   to `.metadata`.
6. Writes `env_name=<env_name>` to `.metadata` between `install_root` and
   `env_root`.
7. Replaces `{{ env_name }}` placeholders in `<env_name>/config/.env` with the
   validated `env_name`.
8. Does not place a per-environment CLI binary in the created directory.
   The `oqtopus` executable is expected to already be installed in a location
   on the user's `PATH`.
9. Does **NOT** install backend components.
10. Creates runtime directories dynamically. These are not copied from the
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

The `templates/backend/` directory is treated as the source template root.
Its files are copied into `<env_name>/`, except for `.metadata`, which is
always generated dynamically by `oqtopus init`.

The expected configuration tree under `templates/backend/config/` is:

```text
templates/backend/config/
  .env
  core/
    config.yaml
    logging.yaml
  sse_engine/
    config.yaml
    logging.yaml
  mitigator/
    config.yaml
    logging.yaml
  estimator/
    config.yaml
    logging.yaml
  combiner/
    config.yaml
    logging.yaml
  tranqu/
    config.yaml
    logging.yaml
  gateway/
    config.yaml
    logging.yaml
```

`templates/backend/config/.env` is distributed as the initial environment
variable file for launched backend processes. It MUST NOT contain secrets.
The template may contain `{{ env_name }}` placeholders. `oqtopus init` replaces
those placeholders in `config/.env` with the validated environment name.

Runtime-only empty directories such as `pids/`, `logs/<component>/`, and
`sse_work/` are created by `oqtopus init`. They are not represented as template
files.

## 5. Help, Version, and Completion

### 5.1 Help behavior

The CLI MUST provide help at the top level and for subcommands.

Supported forms:

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

The same pattern applies to all backend subcommands. Help commands MUST NOT
require backend environment validation.

### 5.2 Version behavior

The CLI MUST provide:

```bash
oqtopus version
oqtopus --version
```

Both commands print the installed OQTOPUS CLI version and exit successfully
without requiring backend environment validation.

Suggested output:

```text
oqtopus v1.0.0
```

`oqtopus backend info` remains the command for backend environment metadata
such as component version bindings and expanded paths. `oqtopus version` is
only for the CLI version.

### 5.3 Shell completion behavior

The CLI MUST provide:

```bash
oqtopus completion <bash|zsh|fish>
```

The command prints the completion script for the requested shell to stdout and
exits successfully without requiring backend environment validation.

Completion MUST cover:

- Top-level commands: `init`, `backend`, `completion`, `version`, `help`
- Top-level flags: `--help`, `--version`
- `oqtopus init`: `--template`, `help`, `--help`
- Template names: `backend`, `cloud`
- `oqtopus backend`: `install`, `versions`, `uninstall`, `update`, `prune`,
  `start`, `stop`, `restart`, `status`, `device-status`, `info`, `help`,
  `--help`
- `oqtopus backend device-status`: `show`, `active`, `inactive`,
  `maintenance`, `help`, `--help`
- Install components: `engine`, `tranqu`, `gateway`, `all`
- Versions/update/uninstall components: `engine`, `tranqu`, `gateway`
- Start/stop/restart components: `core`, `sse_engine`, `mitigator`,
  `estimator`, `combiner`, `tranqu`, `gateway`, `all`

Version strings MUST NOT be completed in v1.0.0. Completion MUST NOT make
network requests.

## 6. `oqtopus backend` Specification

### 6.1 Mandatory pre-execution checks

Before running ANY backend command, the following validations MUST be executed:

1. `.metadata` MUST exist in the current directory.
2. `.metadata` MUST include:

   ```text
   template=backend
   ```

3. The current working directory MUST match the `env_root` field in `.metadata`.

   - If they differ, the CLI MUST exit with an error.
   - This prevents running `oqtopus backend` from the wrong directory.

4. Running `oqtopus backend ...` outside a directory created by
   `oqtopus init <env_name> --template backend` MUST fail with an error.
5. Optional version bindings (`engine_version`, `tranqu_version`, etc.) MUST NOT affect validation, but will be used later during `start`.

#### Error examples

Missing file:

```text
Error: .metadata not found.
This directory is not an OQTOPUS backend environment.
```

Template mismatch:

```text
Error: invalid environment template.
Found template='cloud', but 'oqtopus backend' requires template='backend'.
```

`env_root` mismatch:

```text
Error: Current directory does not match env_root.
env_root = /home/user/myenv
current  = /home/user/otherdir
```

Executed outside a backend environment:

```text
Error: oqtopus backend must be run inside a directory created by
oqtopus init <env_name> --template backend.
```

Install failure example:

```text
Error: failed to install component 'tranqu'.
Could not resolve a latest stable version from GitHub tags.
```

Uninstall failure example:

```text
Error: cannot uninstall 'engine' version 'v1.2.0'.
The version is still referenced by another environment.
```

Start failure example:

```text
Error: cannot start 'core'.
The component appears to be already running (PID 12345).
```

Stop failure example:

```text
Error: failed to stop 'gateway'.
The process did not exit within 5 seconds after TERM was sent.
```

## 7. Installation Layout (XDG Data Directory)

`oqtopus backend` manages three backend components:

- `engine`
- `tranqu`
- `gateway`

Installed releases for these components are stored under the `install_root`
defined in `.metadata`:

```text
<install_root>/<component>-<version>/
```

Each installed version is fully self-contained with its own `.venv`.

## 8. Backend Installation Using `uv sync`

After extracting a release archive, the CLI MUST synchronize the component's
Python environment with `uv`.

For `tranqu` and `gateway`, the CLI MUST perform:

```bash
uv sync --frozen --no-dev --project <install_root>/<component>-<version>/
```

For `engine`, the release is a monorepo. The CLI MUST run `uv sync` for each
engine service project:

```bash
uv sync --frozen --no-dev --project <install_root>/engine-<version>/core
uv sync --frozen --no-dev --project <install_root>/engine-<version>/combiner
uv sync --frozen --no-dev --project <install_root>/engine-<version>/estimator
uv sync --frozen --no-dev --project <install_root>/engine-<version>/mitigator
```

`sse_engine` is launched from the `core` engine project.

Behavior:

- If both `pyproject.toml` and `uv.lock` exist → `uv.lock` is used for strict reproducibility.
- No `pip install` is allowed unless explicitly documented as fallback.

### Guarantees

- If `uv.lock` is present, dependency resolution is pinned so repeated installs
  produce the same environment.
- Each installed release keeps its own `.venv`, so different versions can
  coexist under the shared `install_root`.
- Because each version is isolated in its own directory, rollback is done by
  selecting a different installed version rather than mutating a shared runtime.

## 9. Backend Commands

### 9.1 install

```bash
oqtopus backend install <engine|tranqu|gateway> [version]
oqtopus backend install all
```

Downloads and synchronizes the specified component to the shared installation root.
`all` installs all backend components using each component's latest stable
version.

Execution Flow:

1. Version Resolution:

   - Uses the provided `version` (e.g., `v2.0.0`).
   - If omitted, queries the public GitHub tags API for the respective repository and selects the newest stable semantic version tag in `vX.Y.Z` format.
   - `oqtopus backend install all` does not accept a version argument.
   - For `all`, the CLI resolves the latest stable version independently for
     `engine`, `tranqu`, and `gateway`.
   - The tags API request SHOULD use `?per_page=100`.
   - If the tags API response contains 100 tags, the CLI SHOULD print a warning
     that additional tags may exist and latest version resolution may be incomplete.
   - Suggested warning:

     ```text
     Warning: the GitHub tags API returned 100 tags.
     Additional tags may exist, so latest version resolution may be incomplete.
     ```

   - Pre-release tags are excluded from automatic latest selection.

2. Directory Preparation:

   - Creates the target directory: `<install_root>/<component>-<version>/`.
   - If the directory already exists and contains a valid `.venv`, the CLI
     reuses it and updates the metadata binding.
   - If the directory already exists but does not contain `.venv`, the CLI
     treats it as an incomplete installation, removes that
     component-version directory, and downloads/extracts/synchronizes it again.
   - A `--force` reinstall option is deferred to a future release.

3. Retrieval & Extraction:

   - Downloads GitHub release archive:
     - engine: `https://github.com/oqtopus-team/oqtopus-engine/archive/refs/tags/<version>.tar.gz`
     - tranqu: `https://github.com/oqtopus-team/tranqu-server/archive/refs/tags/<version>.tar.gz`
     - gateway: `https://github.com/oqtopus-team/device-gateway/archive/refs/tags/<version>.tar.gz`
   - Extracts the contents directly into the target directory with `--strip-components=1`.

4. Environment Synchronization:

   - For `tranqu` and `gateway`, executes `uv sync --frozen --no-dev --project <install_root>/<component>-<version>/` inside the target directory.
   - For `engine`, executes `uv sync --frozen --no-dev` for the `core`,
     `combiner`, `estimator`, and `mitigator` subprojects.
   - This ensures a deterministic, production-ready `.venv` is created using the Python version specified in the component's `pyproject.toml`.

5. SSE Runtime Docker Image Build:

   - When installing `engine`, the CLI MUST build the `sse_runtime` Docker
     image after `uv sync` succeeds.
   - The Dockerfile is expected at:

      ```text
      <install_root>/engine-<version>/sse_runtime/Dockerfile
      ```

   - The CLI MUST load `SSE_CONTAINER_IMAGE` from `<env_root>/config/.env`.
   - The CLI MUST use the current user's UID and GID from `id -u` and `id -g`
     for the Docker build arguments.
   - The build command is equivalent to:

      ```bash
      docker build <install_root>/engine-<version>/sse_runtime \
          -t <SSE_CONTAINER_IMAGE> \
          --build-arg UID=$(id -u) \
          --build-arg GID=$(id -g)
      ```

   - This is based on the existing backend setup `build-sse_runtime` behavior.
   - If Docker is not available, `id` is not available, the Dockerfile is
     missing, `SSE_CONTAINER_IMAGE` is missing from `config/.env`, or the
     Docker build fails, `oqtopus backend install engine` MUST fail and MUST
     NOT update the `engine_version` binding.
   - Installing `tranqu` or `gateway` does not build `sse_runtime`.

6. Metadata Binding Update:

   - After successful installation, the CLI MUST update `.metadata`:
   - Existing bindings are replaced atomically.

7. Configuration Files:

   - The CLI MUST NOT modify files under `<env_root>/config/` during `install`.
   - If the installed version requires configuration changes, the user MUST
     update the configuration files manually.

     ```text
     engine_version=v1.3.2
     tranqu_version=v1.3.0
     gateway_version=v1.2.1
     ```

8. `all` Target:

   - `oqtopus backend install all` installs:

      ```text
      engine
      tranqu
      gateway
      ```

   - The install order MUST be `engine`, then `tranqu`, then `gateway`.
   - Each component resolves its own latest stable version independently from
     its own repository.
   - `oqtopus backend install all <version>` MUST fail with a usage error.
   - If installing any component fails, the command MUST stop and return a
     non-zero exit status.
   - Components already installed successfully by the same command keep their
     metadata bindings; rollback is not performed automatically.
   - The failed component MUST NOT update its metadata binding.

### 9.2 versions

```bash
oqtopus backend versions <engine|tranqu|gateway>
```

Lists available stable versions for the specified backend component.

The command:

- does not require backend environment validation and can run outside a
  backend environment;
- queries the same GitHub tags API used by latest-version resolution;
- includes only stable semantic version tags in `vX.Y.Z` format;
- excludes pre-release tags such as `v1.0.0-rc1`;
- sorts versions by semantic version in descending order;
- when run from a valid backend environment, marks the current version from
  `.metadata` with `*`;
- when run from a valid backend environment, marks locally available release
  directories under `install_root` with `(installed)`;
- when a current or locally installed version is not present in remote stable
  tags, still includes it and marks it with `(not in remote tags)`;
- does not support `all`;
- does not complete version strings.

Example:

```text
engine:
* v2.0.3 (installed)
  v2.0.2 (installed)
  v2.0.1
```

If the GitHub tags API response contains 100 tags, the CLI SHOULD print:

```text
Warning: the GitHub tags API returned 100 tags.
Additional tags may exist, so the version list may be incomplete.
```

If no stable semantic version tags are found, the command MUST fail with a
clear error:

```text
Error: no stable versions found for engine.
```

### 9.3 uninstall

```bash
oqtopus backend uninstall <component> <version>
```

- Removes the release directory for the specified version.
- Does not check whether the specified version is referenced by the current
  environment or by any other environment.
- `uninstall` does not rewrite `.metadata`.
- If an environment still references a removed version, later commands such as
  `start` MUST fail with a clear "installed release directory not found" error
  until the version is installed again or the environment binding is updated.

### 9.4 update

```bash
oqtopus backend update <component>
```

- Equivalent to `oqtopus backend install <component> <latest>`.
- Performs no special processing beyond install plus metadata update.

### 9.5 start

Starts backend processes.

```bash
oqtopus backend start <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
```

Expected behavior:

1. `env_root` Validation
   - The current working directory MUST equal `env_root` defined in `.metadata`.
   - If not, the CLI MUST exit before performing any action.
2. Stale PID Handling
   - If `<env_root>/pids/<component>.pid` exists:
     - If the PID is still alive → the CLI MUST exit with an error.
     - If the PID does NOT exist (stale) → the CLI MUST delete the PID file and proceed.
3. Environment Variable Loading
   - If `<env_root>/config/.env` exists, the CLI MUST load all valid `key=value` entries (ignoring comments and empty lines)
   - and provide them only to the environment of the launched `uv` process.
   - These variables MUST NOT be persisted as global or user-shell environment variables.
4. Launching the Backend Component
   - All managed backend processes are launched via `uv`.
   - `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are independent managed services launched from the installed `engine` release.
   - `tranqu` is launched from the installed `tranqu` release.
   - `gateway` is managed as a Python/uv component and launched from the
     installed `gateway` release.
   - Placeholder processes are development/test-only. They MUST NOT be used as
     the default v1.0.0 user-facing behavior.
   - If the required version binding is missing from `.metadata`, or if the
     installed release directory is missing, `start` MUST fail with a clear
     error.
   - For `core`:

      ```bash
      uv run --project <install_root>/engine-<version>/core python -m oqtopus_engine_core.app \
          -c <env_root>/config/core/config.yaml \
          -l <env_root>/config/core/logging.yaml
      ```

   - For `sse_engine`:

      ```bash
      uv run --project <install_root>/engine-<version>/core python -m oqtopus_engine_core.app \
          -c <env_root>/config/sse_engine/config.yaml \
          -l <env_root>/config/sse_engine/logging.yaml
      ```

   - For `mitigator`:

      ```bash
      uv run --project <install_root>/engine-<version>/mitigator python -m oqtopus_engine_mitigator.app \
          -c <env_root>/config/mitigator/config.yaml \
          -l <env_root>/config/mitigator/logging.yaml
      ```

   - For `estimator`:

      ```bash
      uv run --project <install_root>/engine-<version>/estimator python -m oqtopus_engine_estimator.app \
          -c <env_root>/config/estimator/config.yaml \
          -l <env_root>/config/estimator/logging.yaml
      ```

   - For `combiner`:

      ```bash
      uv run --project <install_root>/engine-<version>/combiner python -m oqtopus_engine_combiner.app \
          -c <env_root>/config/combiner/config.yaml \
          -l <env_root>/config/combiner/logging.yaml
      ```

   - For `tranqu`:
     - The installed version MUST be read from `.metadata`: `tranqu_version=vX.Y.Z`
     - The CLI MUST launch the installed release using:

      ```bash
      uv run --project <install_root>/tranqu-<version> python -m tranqu_server.proto.service \
          -c <env_root>/config/tranqu/config.yaml \
          -l <env_root>/config/tranqu/logging.yaml
      ```

   - For `gateway`:

      ```bash
      uv run --project <install_root>/gateway-<version> python -m device_gateway.service \
          -c <env_root>/config/gateway/config.yaml \
          -l <env_root>/config/gateway/logging.yaml
      ```

5. Process Output
   - The CLI MUST redirect stdout and stderr for each launched process to
     `/dev/null`.
   - The CLI MUST NOT create log files itself.
   - Application log files are created by the backend applications according to
     their `logging.yaml` configuration.

6. `all` Target
   - `oqtopus backend start all` starts all managed services.
   - Start order MUST be:

      ```text
      gateway
      tranqu
      mitigator
      estimator
      combiner
      sse_engine
      core
      ```

   - The command MUST apply the same validation, stale PID handling,
     environment loading, process launch, PID writing, and stdout/stderr
     handling used by single-service `start`.
   - If starting any service fails, the command MUST stop and return a non-zero
     exit status. Services already started by the same command are left running;
     rollback is not performed automatically.

### 9.6 stop

```bash
oqtopus backend stop <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
```

Safely terminates the process.

- Reads the PID from the file in the local `pids/` directory.
- If no PID file exists, the component is treated as already stopped.
- If the PID file exists but the process is not alive, the PID file is treated
  as stale and removed.
- If the process is alive, the CLI sends `TERM` to the corresponding PID.
- The CLI waits up to 5 seconds for the process to exit.
- If the process exits within 5 seconds, the PID file is removed.
- If the process is still running after 5 seconds, the CLI MUST exit with an
  error.
- The CLI MUST NOT send `KILL` automatically.

For `oqtopus backend stop all`:

- The CLI stops all managed services.
- Stop order MUST be the reverse of start order:

   ```text
   core
   sse_engine
   combiner
   estimator
   mitigator
   tranqu
   gateway
   ```

- The command MUST apply the same PID handling and termination behavior used by
  single-service `stop`.
- If stopping any service fails, the command MUST continue attempting to stop
  the remaining services and exit non-zero after reporting the failure.

### 9.7 restart

```bash
oqtopus backend restart <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway|all>
```

Restarts managed backend services.

For a single service, `restart` is equivalent to:

1. `oqtopus backend stop <service>`
2. `oqtopus backend start <service>`

If `stop` fails, `restart` MUST fail and MUST NOT start the service.

For `oqtopus backend restart all`:

- The CLI first stops all managed services using the same order and behavior as
  `oqtopus backend stop all`.
- If any stop operation fails, the command exits non-zero and MUST NOT start
  services again.
- If all stop operations succeed, the CLI starts all managed services using
  the same order and behavior as `oqtopus backend start all`.

### 9.8 status

```bash
oqtopus backend status
```

Prints only **process status**:

```text
core: Stopped
sse_engine: Stopped
mitigator: Running (PID 12345)
estimator: Stopped
combiner: Running (PID 12348)
tranqu: Stopped
gateway: Stopped
```

If a PID file exists but the process is not alive, the component is treated as
stopped.

### 9.9 device-status

```bash
oqtopus backend device-status show
oqtopus backend device-status active
oqtopus backend device-status inactive
oqtopus backend device-status maintenance
```

Manages the local gateway device status file.

Expected behavior:

1. Environment Validation
   - The standard backend environment validation MUST run before executing any
     `device-status` command.
2. Target File
   - The command operates on:

      ```text
      <env_root>/config/gateway/device_status
      ```

   - If the file does not exist, the CLI MUST fail with a clear error.
3. `show`
   - Prints the current file contents.
4. Status Updates
   - `active` writes:

      ```text
      active
      ```

   - `inactive` writes:

      ```text
      inactive
      ```

   - `maintenance` writes:

      ```text
      maintenance
      ```

5. Constraints
   - Valid status values are only `active`, `inactive`, and `maintenance`.
   - The command directly updates the local configuration file and does not
     require `gateway` to be running.
   - The command does not call scripts from the installed `gateway` release.

### 9.10 info

```bash
oqtopus backend info
```

Prints:

1. `.metadata` version bindings
2. Expanded paths recorded in `.metadata`

`backend info` does not print Python executable or Python version information.
Managed services run through their component-specific `uv` environments, so a
single process-level Python path would be misleading.

`backend info` also does not print installed release directories. The source of
truth for the environment's selected component versions is `.metadata`.

#### Example

```text
=== Backend Info (.metadata) ===
template=backend
install_root=/home/user/.local/share/oqtopus/backend/releases
env_name=myenv
env_root=/home/user/myenv
created_at=2026-05-01T00:00:00Z
engine_version=v2.1.0
tranqu_version=v0.3.3
gateway_version=v0.2.5
```

## 10. Environment Registry

The CLI MUST NOT maintain a central backend environment registry.

- No `environments.json` file is created or updated.
- Each backend environment's `.metadata` is the source of truth for that
  environment only.
- The CLI does not scan or track other environments before deleting installed
  releases.
- `prune` is not provided in v1.0.0.

## 11. Template Retrieval

Templates are managed directly within the `oqtopus-cli` repository.

### 11.1 Repository Structure

Templates are stored in the `templates/` directory of the official repository:
`https://github.com/oqtopus-team/oqtopus-cli/tree/main/templates/`

### 11.2 Retrieval Logic

1. Default Source: `oqtopus init` downloads templates from the `main` branch.
   - URL Pattern: `https://raw.githubusercontent.com/oqtopus-team/oqtopus-cli/main/templates/<template_type>/`
2. `oqtopus init` MUST always download templates from GitHub and MUST NOT use
   a local cache.
3. The CLI fetches the files required to materialize the template under the
   new environment directory.
4. If template download fails, including when the network is unavailable, the
   CLI MUST print an error and exit.
5. `.metadata` is not downloaded as a literal template file. It is generated
   dynamically by `oqtopus init`.

## 12. Implementation Requirements

### 12.1 v1.0.0 shell implementation

- CLI provided as `bin/oqtopus`.
- Portable across Linux/macOS.
- May rely on common POSIX utilities plus `bash`, `curl`, `tar`, `jq`, and `uv`.
- `docker` is required when running `oqtopus backend install engine`, because
  the `sse_runtime` image is built during engine installation.
- `env_name` is used in Docker-related configuration values, so it must match
  `^[a-z0-9][a-z0-9_.-]*$`.
- No Windows support for v1.0.0.

### 12.2 Future Rust implementation

- CLI implemented using **clap**.
- Network operations via **reqwest**.
- `.metadata` loader provided by shared crate.
- Python/uv execution via subprocess.
- Process management for Linux/macOS.
- No Python dependency in the CLI itself.

The `scripts/install.sh` behavior is specified separately in
`install-sh.md`.

## 13. Future Extensions

- oqtopus-cloud CLI
