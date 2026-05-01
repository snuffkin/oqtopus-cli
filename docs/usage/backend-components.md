# Managing Backend Components

OQTOPUS CLI manages three backend components:

- `engine`
- `tranqu`
- `gateway`

These names are installable component targets. Each target corresponds to a
separate source repository:

- `engine` [https://github.com/oqtopus-team/oqtopus-engine](https://github.com/oqtopus-team/oqtopus-engine)
- `tranqu` [https://github.com/oqtopus-team/tranqu-server](https://github.com/oqtopus-team/tranqu-server)
- `gateway` [https://github.com/oqtopus-team/device-gateway](https://github.com/oqtopus-team/device-gateway)

`install`, `update`, and `uninstall` operate on these component targets.

Installed releases are stored under the backend installation root recorded in
`.metadata`. Each component version has its own isolated directory and virtual
environment.

The installable components are not the same as the process targets used by
`start` and `stop`. In particular, `engine` is installed as one component, but
it provides multiple managed services such as `core`, `sse_engine`,
`mitigator`, `estimator`, and `combiner`.

## Install All Components

```bash
oqtopus backend install all
```

This installs `engine`, `tranqu`, and `gateway`.

Each component resolves its own latest stable version independently from its own
repository. Because the components are versioned independently,
`install all` does not accept a version argument.

The install order is:

1. `engine`
2. `tranqu`
3. `gateway`

If one component fails to install, the command stops. Components already
installed successfully remain bound in `.metadata`; rollback is not performed
automatically.

## Install One Component

Install the latest stable version:

```bash
oqtopus backend install engine
```

Install a specific version:

```bash
oqtopus backend install engine v1.2.3
```

The same form applies to `tranqu` and `gateway`.

## Engine And `sse_runtime`

Installing `engine` also builds the `sse_runtime` Docker image from the
installed engine release.

Docker must be available when running:

```bash
oqtopus backend install engine
```

or:

```bash
oqtopus backend install all
```

If Docker is unavailable, required `config/.env` values are missing, or the
Docker build fails, the engine install fails and the `engine_version` binding is
not updated.

## Update One Component

```bash
oqtopus backend update engine
```

`update` is equivalent to installing the latest stable version of the specified
component and updating the environment binding.

## Uninstall A Component Version

```bash
oqtopus backend uninstall engine v1.2.3
```

The CLI refuses to uninstall a version that is still referenced by a registered
environment.

## Prune Unused Releases

```bash
oqtopus backend prune
```

`prune` lists unreferenced installed releases and asks for confirmation before
deleting them.

To skip the confirmation prompt:

```bash
oqtopus backend prune --yes
```

## Configuration Files

`install`, `update`, `uninstall`, and `prune` do not modify files under
`config/`.

If a component version requires configuration changes, update the configuration
files manually.
