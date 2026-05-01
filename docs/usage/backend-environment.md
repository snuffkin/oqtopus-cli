# Backend Environment

An OQTOPUS backend environment is a local directory created by:

```bash
oqtopus init <env_name> --template backend
```

Backend commands must be run from the root of this environment.

## Directory Layout

```text
<env_name>/
  .metadata
  config/
    .env
    core/
    sse_engine/
    mitigator/
    estimator/
    combiner/
    tranqu/
    gateway/
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

## `.metadata`

`.metadata` records environment-specific information such as:

- the environment template;
- the absolute environment path;
- the shared backend installation root;
- the installed component versions bound to this environment.

Do not move an environment directory after creating it. Backend commands check
that the current directory matches the `env_root` recorded in `.metadata`.

## `config/`

`config/` contains backend configuration files.

`config/.env` contains environment variables used when launching backend
processes. These variables are applied only to the launched process environment;
they are not written to your global shell environment.

`config/.env` is a configuration file and should not contain secrets.

## `logs/`

`logs/` contains one directory for each managed service.

OQTOPUS CLI creates the directories, but it does not create application log
files. Log files are created by the backend applications according to their
`logging.yaml` configuration.

## `pids/`

`pids/` stores PID files for managed services.

The CLI uses these files to detect running services and to stop them safely.
Stale PID files are removed automatically when the recorded process no longer
exists.

## `sse_work/`

`sse_work/` is the host-side working directory for dynamically launched
`sse_runtime` Docker containers.

The default `SSE_HOST_WORK_PATH` in `config/.env` points to this directory
relative to the environment root.
