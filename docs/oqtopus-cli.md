# OQTOPUS CLI Specification

This document defines the official behavior and interface of the
OQTOPUS Local Backend Environment CLI.
The CLI is implemented in **Rust** and distributed as a standalone executable
named `oqtopus`.

## 1. Overview

OQTOPUS CLI provides a single top-level command `oqtopus`:

- `oqtopus init`
  - Creates new local environments from templates.
- `oqtopus backend`
  - Manages backend components: `engine`, `tranqu`, and `gateway`.
  - Supports install / uninstall / update / prune.
  - Starts and stops backend processes.
  - Validates environment integrity before executing commands.

Current support target:

- Linux/macOS: supported
- Windows: not supported yet

## 2. Directory Structure (backend template)

`oqtopus init <env_name> --template backend` creates:

```text
<env_name>/
  .metadata       # environment-specific metadata
  .env            # environment variables
  config/         # configuration files for each microservice
  logs/           # per-service log directories
  pids/           # PID files such as core.pid
```

No backend code is installed at init time.

The `logs/` directory contains one subdirectory for each managed service, such
as `logs/core/`, `logs/sse_engine/`, and `logs/tranqu/`.

The `pids/` directory stores PID files using the `<component>.pid` naming
convention, such as `core.pid`, `sse_engine.pid`, and `tranqu.pid`.

## 3. .metadata Specification

`.metadata` is a simple `key=value` file (NOT TOML).

### 3.1 Required fields

`install_root` is resolved as follows:

1. Use `$XDG_DATA_HOME/oqtopus/backend/releases` if `XDG_DATA_HOME` is set.
2. Otherwise use `~/.local/share/oqtopus/backend/releases`.

```text
template=backend
install_root=<absolute path to the backend releases directory>
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

1. Creates the env directory.
2. Downloads the backend template from the official GitHub repository.
3. Copies the contents of `templates/backend/` into the root of the new environment directory.
4. Generates `.metadata` dynamically for the new environment.
5. Resolves the `install_root` path (respecting $XDG_DATA_HOME) and writes it to `.metadata`.
6. Does not place a per-environment CLI binary in the created directory.
   The `oqtopus` executable is expected to already be installed in a location
   on the user's `PATH`.
7. Does **NOT** install backend components.

The `templates/backend/` directory is treated as the source template root.
Its files are copied into `<env_name>/`, except for `.metadata`, which is
always generated dynamically by `oqtopus init`.

The expected configuration tree under `templates/backend/config/` is:

```text
templates/backend/config/
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

## 5. `oqtopus backend` Specification

### 5.1 Mandatory pre-execution checks

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

## 6. Installation Layout (XDG Data Directory)

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

## 7. Backend Installation Using `uv sync`

After extracting a release archive, the CLI MUST perform:
Executed inside the release directory:

```bash
uv sync --frozen --no-dev --project <install_root>/<component>-<version>/
```

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

## 8. Backend Commands

### 8.1 install

```bash
oqtopus backend install <engine|tranqu|gateway> [version]
```

Downloads and synchronizes the specified component to the shared installation root.

Execution Flow:

1. Version Resolution:

   - Uses the provided `version` (e.g., `v2.0.0`).
   - If omitted, queries the public GitHub tags API for the respective repository and selects the newest stable semantic version tag in `vX.Y.Z` format.
   - The tags API request SHOULD use `?per_page=100`.
   - If the tags API response contains 100 tags, the CLI SHOULD print a warning
     that additional tags may exist and latest version resolution may be incomplete.
   - Pre-release tags are excluded from automatic latest selection.

2. Directory Preparation:

   - Creates the target directory: `<install_root>/<component>-<version>/`.
   - If the directory already exists and contains a valid `.venv`, the CLI may skip the download and only verify the environment.

3. Retrieval & Extraction:

   - Downloads GitHub release archive:
     - engine: `https://github.com/oqtopus-team/oqtopus-engine/archive/refs/tags/<version>.tar.gz`
     - tranqu: `https://github.com/oqtopus-team/tranqu-server/archive/refs/tags/<version>.tar.gz`
     - gateway: `https://github.com/oqtopus-team/device-gateway/archive/refs/tags/<version>.tar.gz`
   - Extracts the contents directly into the target directory with `--strip-components=1`.

4. Environment Synchronization:

   - Executes `uv sync --frozen --no-dev --project <install_root>/<component>-<version>/` inside the target directory.
   - This ensures a deterministic, production-ready `.venv` is created using the Python version specified in the component's `pyproject.toml`.

5. Metadata Binding Update:

   - After successful installation, the CLI MUST update `.metadata`:
   - Existing bindings are replaced atomically.

6. Configuration Files:

   - The CLI MUST NOT modify files under `<env_root>/config/` during `install`.
   - If the installed version requires configuration changes, the user MUST
     update the configuration files manually.

```text
tranqu_version=v0.3.0
engine_version=v0.3.2
gateway_version=v0.2.1
```

### 8.2 uninstall

```bash
oqtopus backend uninstall <component> <version>
```

- Removes the release directory for the specified version.
- If the specified version is currently referenced by any registered
  environment, the command MUST fail and MUST NOT delete the directory.
- `uninstall` does not rewrite `.metadata`; it only operates on versions that
  are not currently referenced.

### 8.3 update

```bash
oqtopus backend update <component>
```

- Equivalent to `oqtopus backend install <component> <latest>`.
- Performs no special processing beyond install plus metadata update.

### 8.4 prune

```bash
oqtopus backend prune
```

- Lists unreferenced directories under `<install_root>` that will be deleted.
- Prompts for confirmation before deleting them.
- `oqtopus backend prune --yes` skips the confirmation prompt.
- Retains versions referenced by any `.metadata`.

Suggested interactive prompt:

```text
The following installed releases will be deleted:
  - /home/user/.local/share/oqtopus/backend/releases/engine-v1.2.0
  - /home/user/.local/share/oqtopus/backend/releases/tranqu-v0.4.1

Proceed? [y/N]:
```

### 8.5 start

Starts backend processes.

```bash
oqtopus backend start <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway>
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
   - If `.env` exists, the CLI MUST load all valid `key=value` entries (ignoring comments and empty lines)
   - and provide them only to the environment of the launched `uv` process.
   - These variables MUST NOT be persisted as global or user-shell environment variables.
4. Launching the Backend Component
   - All managed backend processes are launched via `uv`.
   - `core`, `sse_engine`, `mitigator`, `estimator`, and `combiner` are independent managed services launched from the installed `engine` release.
   - `tranqu` is launched from the installed `tranqu` release.
   - `gateway` is launched from the installed `gateway` release.
   - For `core`:

      ```bash
      uv run --project <install_root>/engine-<version> python -m oqtopus_engine_core.app \
          -c <env_root>/config/core/config.yaml \
          -l <env_root>/config/core/logging.yaml
      ```

   - For `sse_engine`:

      ```bash
      uv run --project <install_root>/engine-<version> python -m oqtopus_engine_core.app \
          -c <env_root>/config/sse_engine/config.yaml \
          -l <env_root>/config/sse_engine/logging.yaml
      ```

   - For `mitigator`:

      ```bash
      uv run --project <install_root>/engine-<version> python -m oqtopus_engine_mitigator.app \
          -c <env_root>/config/mitigator/config.yaml \
          -l <env_root>/config/mitigator/logging.yaml
      ```

   - For `estimator`:

      ```bash
      uv run --project <install_root>/engine-<version> python -m oqtopus_engine_estimator.app \
          -c <env_root>/config/estimator/config.yaml \
          -l <env_root>/config/estimator/logging.yaml
      ```

   - For `combiner`:

      ```bash
      uv run --project <install_root>/engine-<version> python -m oqtopus_engine_combiner.app \
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

### 8.6 stop

```bash
oqtopus backend stop <component>
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

### 8.7 status

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

### 8.8 info

```bash
oqtopus backend info
```

Prints:

1. Installed backend releases for each component
2. `.metadata` version bindings
3. Python environment info
4. Expanded paths

#### Example

```text
=== Environment (.metadata) ===
template = backend
engine_version = v2.1.0
tranqu_version = v0.3.3
gateway_version = v0.2.5

python_path = /usr/bin/python3
python_version = 3.11.7

install_root = /home/user/.local/share/oqtopus/backend/releases

=== Installed Backend Releases ===
engine: v0.3.3, v0.3.2
tranqu: v0.1.4
gateway: v0.2.5
```

## 9. Environment Registry (for prune operation)

To ensure that the `prune` command can identify all active environments without performing a full-disk scan, the CLI maintains a central registry of environment paths.

### 9.1 Registry File

- Path:
  - `$XDG_CONFIG_HOME/oqtopus/backend/environments.json` if `XDG_CONFIG_HOME` is set
  - Otherwise `~/.config/oqtopus/backend/environments.json`
- Format: A JSON file containing an array of absolute directory paths to active backend environments.

### 9.2 Registration Logic

- `oqtopus init`: Whenever a new environment is created with the `backend` template, its absolute path MUST be appended to this registry file.
- Automatic Detection: When any `oqtopus backend` command is executed within a directory, the CLI SHOULD verify if the current path is in the registry and add it if missing.

### 9.3 Prune Workflow

1. Load Registry: The CLI reads all paths from the `environments.json`.
2. Scan Environments: For each registered path:
   - Check if the directory and `.metadata` file still exist.
   - If they are missing, the path is removed from the registry (Self-healing).
   - If they exist, the CLI parses `.metadata` to collect all currently bound version strings (e.g., `engine_version`, `tranqu_version`, `gateway_version`).
3. Identify Garbage: The CLI lists all versioned directories in the `install_root`.
4. Deletion: Any directory in `install_root` that is NOT referenced by any active environment in the registry is safely deleted.

## 10. Template Retrieval

Templates are managed directly within the `oqtopus-cli` repository.

### 10.1 Repository Structure

Templates are stored in the `templates/` directory of the official repository:
`https://github.com/oqtopus-team/oqtopus-cli/tree/main/templates/`

### 10.2 Retrieval Logic

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

## 11. Rust Implementation Requirements

- CLI implemented using **clap**.
- Network operations via **reqwest**.
- `.metadata` loader provided by shared crate.
- Python/uv execution via subprocess.
- Process management for Linux/macOS.
- No Python dependency in the CLI itself.

The `scripts/install.sh` behavior is specified separately in
`docs/install-sh.md`.

## 12. Future Extensions

- oqtopus-cloud CLI
