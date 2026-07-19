# wandr-wit

The portable **WIT contracts** for the [wandr](https://codeberg.org/harryzz/wandr)
runtime — the OS-agnostic boundary between a guest app and the host.

A guest app (any language, any UI framework) compiles once to a WASM component against
these contracts; `wandr-host` implements them and delegates only OS-specific bits to a
per-platform backend. The contracts are portable — only the backend is not.

| Path | Contents |
|---|---|
| `wit/` | the `wandr:*` packages — `ui-shell`, `device`, `chrome`, `assets`, `ime`, `crypto`, … |
| `proposals/` | draft WASI proposals wandr implements: `wasi-canvas`, `wasi-audio`, `wasi-input-handlers`, … |

## Consumers

- **[wandr-host](https://github.com/harryzz/wandr-host)** — implements them (host side)
- **[wandr](https://codeberg.org/harryzz/wandr)** — guest crates and apps import them

Both carry this repo as a submodule, so there is exactly one canonical copy of each
contract. Consumers may vendor *subset* copies in their own `wit/deps/`; subset-linking
is sound because the host provides supersets.

## History

Grafted with full history from the wandr monorepo (`wit/` and `proposals/`).
