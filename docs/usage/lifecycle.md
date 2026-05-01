# Starting And Stopping Services

OQTOPUS CLI can start, stop, and inspect managed backend services from inside a
backend environment.

## Managed Services

The managed services are process targets for `start`, `stop`, and `status`.
They are different from the installable component targets used by `install`,
`update`, and `uninstall`.

The installable `engine` component is a set of backend microservices. After
`engine` is installed, OQTOPUS CLI can manage these service targets from that
installed release:

- `core`
- `sse_engine`
- `mitigator`
- `estimator`
- `combiner`

`tranqu` and `gateway` are installed as their own components and are also
managed as service targets:

- `tranqu`
- `gateway`

## Start All Services

```bash
oqtopus backend start all
```

The start order is:

1. `gateway`
2. `tranqu`
3. `mitigator`
4. `estimator`
5. `combiner`
6. `sse_engine`
7. `core`

If one service fails to start, the command stops and exits with an error.
Services already started by the same command are left running.

## Start One Service

```bash
oqtopus backend start gateway
```

If the service is already running, `start` fails instead of starting a duplicate
process.

## Check Process Status

```bash
oqtopus backend status
```

Example output:

```text
core: Stopped
sse_engine: Stopped
mitigator: Running (PID 12345)
estimator: Stopped
combiner: Running (PID 12348)
tranqu: Stopped
gateway: Stopped
```

If a PID file exists but the process is no longer alive, the service is treated
as stopped.

## Stop All Services

```bash
oqtopus backend stop all
```

The stop order is the reverse of the start order:

1. `core`
2. `sse_engine`
3. `combiner`
4. `estimator`
5. `mitigator`
6. `tranqu`
7. `gateway`

If one service fails to stop, the command continues attempting to stop the
remaining services and exits with an error afterward.

## Stop One Service

```bash
oqtopus backend stop gateway
```

If no PID file exists, the service is treated as already stopped.

`stop` sends `TERM` and waits up to 5 seconds. It does not send `KILL`
automatically.

## Process Output And Logs

Runtime stdout and stderr are redirected to `/dev/null`.

OQTOPUS CLI does not create application log files. Backend applications write
their own log files according to the `logging.yaml` files under `config/`.
