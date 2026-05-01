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
   - `.env`
   - `config/`
   - `logs/`
   - `pids/`
3. Resolve `install_root`.
4. Write absolute `env_root` and `install_root` to `.metadata`.
5. Register the environment path in the central registry.
6. Do not create a per-environment CLI binary.
   The `oqtopus` executable is expected to be installed in a directory on
   the user's `PATH`, so it can be run from anywhere.

Only the `backend` template is required for the PoC.
The `cloud` template should return a clear "not implemented" error.

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
oqtopus backend install <engine|tranqu|gateway> [version]
oqtopus backend uninstall <engine|tranqu|gateway> <version>
oqtopus backend update <engine|tranqu|gateway>
oqtopus backend prune
oqtopus backend start <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway>
oqtopus backend stop <core|sse_engine|mitigator|estimator|combiner|tranqu|gateway>
oqtopus backend status
oqtopus backend info
```

The script should parse subcommands as:

```text
oqtopus <init|backend> ...
```

## Component Repository Mapping

Use this mapping for downloads and latest-release lookups:

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
https://api.github.com/repos/<owner>/<repo>/releases/latest
```

Read `.tag_name` from the JSON response.

## Path Rules

### Data Directory

Resolve `install_root` as:

1. `$XDG_DATA_HOME/oqtopus/backend/releases` if `XDG_DATA_HOME` is set.
2. Otherwise `~/.local/share/oqtopus/backend/releases`.

### Config Directory

Resolve the registry path as:

1. `$XDG_CONFIG_HOME/oqtopus/backend/environments.json` if `XDG_CONFIG_HOME` is set.
2. Otherwise `~/.config/oqtopus/backend/environments.json`.

The registry file contains a JSON array of absolute backend environment paths.

## Metadata Format

`.metadata` is a simple `key=value` file.
It is not TOML, YAML, or JSON.

Required fields:

```text
template=backend
install_root=<absolute path>
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

Command:

```bash
oqtopus backend install <component> [version]
```

Required behavior:

1. Resolve version:
   - Use the provided version if present.
   - Otherwise fetch the latest GitHub release tag.
2. Create `<install_root>/<component>-<version>/`.
3. Download the release archive.
4. Extract it into the target directory with the top-level archive directory removed.
5. Run:

```bash
uv sync --frozen --no-dev --project <install_root>/<component>-<version>/
```

6. Update the component version binding in `.metadata`.

If the target directory already exists and contains `.venv`, the command may
skip download and run validation instead.

## Start Behavior

Command:

```bash
oqtopus backend start <component>
```

Required behavior:

1. Check `pids/<component>.pid`.
2. If the PID is alive, exit with an error.
3. If the PID file is stale, delete it.
4. Load valid `key=value` lines from `.env`.
5. Start the component in the background.
6. Write the process PID to `pids/<component>.pid`.
7. Write stdout/stderr to `logs/<component>.log`.

For PoC placeholder components:

- `core`
- `sse_engine`
- `mitigator`
- `estimator`
- `combiner`
- `gateway`

Start a long-running placeholder process such as:

```bash
sleep infinity
```

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
oqtopus backend stop <component>
```

Required behavior:

1. Read `pids/<component>.pid`.
2. If no PID file exists, report the component as stopped.
3. If the PID is alive, send `TERM`.
4. Wait briefly for shutdown.
5. If the process is still alive, return a clear error.
6. Remove the PID file only after the process has stopped or if it was stale.

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

## Info Behavior

Command:

```bash
oqtopus backend info
```

Print:

1. Environment metadata.
2. Installed releases grouped by component.
3. Python executable and version if available.
4. Expanded paths.

## Prune Behavior

Command:

```bash
oqtopus backend prune
```

Required behavior:

1. Load the registry.
2. Remove registry entries whose environment directory or `.metadata` no longer exists.
3. Collect active component bindings from remaining environments.
4. List directories under `install_root`.
5. Delete installed release directories that are not referenced by any active environment.

For safety, the PoC should print each deletion.

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
- `core` transitions from stopped to running to stopped.
- PID files are created and removed correctly.
- Logs are written under `demo/logs/`.
- Running `oqtopus backend ...` outside `demo/` fails validation.
- The same `oqtopus` executable is callable both inside and outside the
  environment because it is installed on `PATH`.

## Open Questions

These should be decided before the Rust implementation:

1. Should `oqtopus backend install` update `.metadata` automatically, or should binding be a separate command?
2. Should `uninstall` refuse to remove a version currently referenced by any registered environment?
3. Should `prune` require an explicit confirmation flag in Rust, such as `--yes`?
4. Should placeholder components remain available in Rust, or are they PoC-only?
5. Should `oqtopus init` download templates in the shell PoC, or create the backend template locally?
6. Should `gateway` eventually be managed like Python components via `uv`, or as a separate binary/process type?
