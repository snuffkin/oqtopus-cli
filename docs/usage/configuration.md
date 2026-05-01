# Configuration

This page will describe how to configure an OQTOPUS backend environment.

Configuration files are created under `config/` when you run:

```bash
oqtopus init <env_name> --template backend
```

The generated files are examples for the local backend environment. Review and
update them as needed before starting services.

## Configuration Directory

```text
config/
  .env
  core/
  sse_engine/
  mitigator/
  estimator/
  combiner/
  tranqu/
  gateway/
```

## Environment Variables

`config/.env` contains environment variables used when launching backend
processes.

These values are applied only to the launched process environment. They are not
persisted to your global shell environment.

`config/.env` is a configuration file and should not contain secrets.

## Service Configuration

Each service has its own configuration directory under `config/`.

Detailed configuration guidance will be added as the supported options are
documented.
