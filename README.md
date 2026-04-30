# oqtopus-cli

OQTOPUS CLI specification and implementation planning.
The CLI entrypoint is `oqtopus`, with subcommands such as `oqtopus init` and
`oqtopus backend`.

## Documents

- `docs/oqtopus-cli.md`: product-level CLI specification.
- `docs/implementation/copilot-implementation-brief.md`: implementation handoff for the shell PoC and later Rust rewrite.
- `docs/install-sh.md`: installer behavior for `scripts/install.sh`.
- `docs/spec-status.md`: decided items, open questions, and deferred topics.

## Repository Layout

- `docs/`: specifications and implementation planning documents
- `templates/`: environment templates used by `oqtopus init`
- `scripts/`: helper scripts such as `install.sh`
- `bin/oqtopus`: current shell-based CLI entrypoint
- `src/`: future Rust implementation
