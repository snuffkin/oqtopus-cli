# oqtopus-cli

OQTOPUS CLI specification and implementation planning.
The CLI entrypoint is `oqtopus`, with subcommands such as `oqtopus init` and
`oqtopus backend`.

## Documents

- `spec/oqtopus-cli.md`: product-level CLI specification.
- `spec/implementation/copilot-implementation-brief.md`: implementation handoff for the shell PoC and later Rust rewrite.
- `spec/install-sh.md`: installer behavior for `scripts/install.sh`.
- `spec/spec-status.md`: decided items, open questions, and deferred topics.

## Repository Layout

- `spec/`: specifications and implementation planning documents
- `docs/`: documents for users
- `templates/`: environment templates used by `oqtopus init`
- `scripts/`: helper scripts such as `install.sh`
- `bin/oqtopus`: current shell-based CLI entrypoint
- `src/`: future Rust implementation
