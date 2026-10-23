> **This is a fork of [oxc-project/tsrs](https://github.com/oxc-project/tsrs).**
> It adds native AST transformer plugins declared in `tsconfig.json`, with the `plugins[].transform`
> syntax of [ttsc](https://ttsc.dev/) for TypeScript-Go. Everything else is upstream `tsrs` at its latest release.
> The upstream README follows the fork section.

## Fork: transformer plugins

A plugin is a shared library (`.dylib` / `.so`) that runs inside the emit pipeline. It reads the AST, queries the type
checker and returns a rewritten source file:

```jsonc
{
  "compilerOptions": {
    "plugins": [
      // Before the TypeScript transforms (type erasure, import elision, decorators, class fields…)
      { "transform": "./plugins/libmy_plugin.dylib" },
      // After the TypeScript and downlevel transforms, before the module transform
      { "transform": "${configDir}/plugins/libmy_plugin.dylib", "after": true },
      // On the .d.ts output
      { "transform": "./plugins/libmy_plugin.dylib", "afterDeclarations": true }
    ]
  }
}
```

`transform` is a library path, resolved from the tsconfig's directory. Install from npm (Linux x64 and arm64, macOS
arm64); the package provides the `tsrs` command, and its version is the upstream `tsrs` release it is built from:

```sh
npm i -D tsrs-with-transformers
npx tsrs -p tsconfig.json
```

Or build from source:

```sh
cargo build --release -p tsrs_cli
target/release/tsrs -p tsconfig.json
```

To write a plugin, build a Rust `cdylib` that depends only on
[`crates/tsrs_plugin_api`](crates/tsrs_plugin_api/src/lib.rs). That crate is the C ABI, and it has no dependencies.
The plugin exports `tsrs_plugin_transform`. [`examples/transform_plugin`](examples/transform_plugin/src/lib.rs) is a
complete example: it rewrites calls, injects an import and queries the checker. Plugins run on macOS and Linux.

What the fork changes, kept small so it rebases cleanly:

| where | what |
| --- | --- |
| `crates/tsrs_plugin_api` (new) | the C ABI and safe `Api` wrapper that plugins build against |
| `crates/tsrs_plugin_host` (new) | the library loader and the host side of the ABI, with its tests and an end-to-end CLI test |
| `examples/transform_plugin` (new) | an example plugin, loaded by that end-to-end test |
| `crates/tsrs_core/src/compileroptions.rs` | `PluginImport` keeps `transform`, `after` and `afterDeclarations` |
| `crates/tsrs_tsoptions/src/{parsinghelpers,gojson}.rs` | read and write those three fields |
| `crates/tsrs_compiler/src/emitter.rs`, `Cargo.toml` | three one-line calls into `tsrs_plugin_host` |
| `tools/lint/unsafe_sites.tsv` | generated rows for the new crates' `unsafe` blocks (in releases that have the file) |
| `tools/fork` (new) | the rebase helper and the npm packaging (`tsrs-with-transformers`, `@tsrs-with-transformers/compiler-*`) |

The plugin API numbers syntax kinds on its own, append-only scale, and the host translates them by name. So when
upstream inserts or reorders kinds, existing plugin binaries keep working.

`main` is the newest upstream release tag plus one fork commit, so it never carries unreleased upstream code. To move
it to a new release: `git fetch upstream --tags && node tools/fork/rebase.mjs <release tag>`. It regenerates
`Cargo.lock` and `unsafe_sites.tsv` itself and stops on any other conflict. Then run `cargo test -p tsrs_plugin_host`.
`tsrs_plugin_host`'s kind test names any new kind to append to
`crates/tsrs_plugin_api/src/kind.rs`.

---

# `tsrs`

[![npm](https://img.shields.io/npm/v/tsrs)](https://www.npmjs.com/package/tsrs)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](https://github.com/oxc-project/tsrs/blob/main/LICENSE)

`tsrs` is a drop-in replacement for the TypeScript 7 compiler and language server, written in Rust.
Our goal is 100% conformance, with much lower memory use and faster type checking. [Read the announcement](https://blog.cloudflare.com/typescript-rust-port/).

## Get started

```sh
pnpm add -D tsrs
pnpm exec tsrs --noEmit --fast
```

`--fast` enables performance optimizations, including adaptive checker counts and additional caches.
See [fast mode](#fast-mode) for details.

You can try `tsrs` today and give us feedback on performance, memory usage, and conformance.
Try checking and emitting code in the [browser playground](https://tsrs-playground.void.app).

<a href="https://github.com/oxc-project/tsrs/blob/main/bench/RESULTS.md">
  <img alt="tsrs --fast, tsrs and TypeScript 7.1 beta: check time and peak memory by project" src="https://raw.githubusercontent.com/oxc-project/tsrs/main/bench/benchmark.svg">
</a>

## Conformance

We aim to release a stable version against TypeScript 7.1 with 100% conformance.
See [compatibility and evidence](https://github.com/oxc-project/tsrs/blob/main/docs/COMPATIBILITY.md)
for supported features and known gaps.

Default mode preserves the pinned TypeScript compiler's checker behavior, including known diagnostic differences
when checker count or file order changes. This sensitivity is inherited from TypeScript itself. See
[differences from pinned TypeScript](https://github.com/oxc-project/tsrs/blob/main/docs/DEFAULTS.md) for the remaining
runtime differences and unsupported features.

## Fast mode

The following defaults and opt-in preset are available in **0.20.0** and later.

By default, tsrs follows the pinned TypeScript compiler's checker settings: four checkers (capped by file count),
Go's file assignment and checker history. Enable the fast preset with `--fast` or
`TSRS_FAST=1`:

```sh
pnpm exec tsrs --noEmit --fast
# Or enable the same preset through the environment:
TSRS_FAST=1 pnpm exec tsrs --noEmit
```

Fast mode adapts checker counts to the machine and project, groups related work together, and enables additional
caches and memory optimizations. It keeps `strict`, `skipLibCheck`, `noCheck` and other checking options as configured.

See [performance options](https://github.com/oxc-project/tsrs/blob/main/docs/USAGE.md#performance-options)
for configuration, behavior and incremental-cache details.

## Extra features

[`@tsrs/wasm`](https://github.com/oxc-project/tsrs/blob/main/npm/tsrs-wasm/README.md) provides WebAssembly support
targeting `wasm32-wasip1` (WASI Preview 1).

## Learn more

- [CLI usage and memory tuning](https://github.com/oxc-project/tsrs/blob/main/docs/USAGE.md)
- [Editor setup](https://github.com/oxc-project/tsrs/blob/main/docs/LSP.md#editor-setup)
- [Node API](https://github.com/oxc-project/tsrs/blob/main/docs/NODE_API.md)
- [WebAssembly for Node and browsers](https://github.com/oxc-project/tsrs/blob/main/npm/tsrs-wasm/README.md)
- [Benchmarks and methodology](https://github.com/oxc-project/tsrs/blob/main/bench/README.md)
- [Release notes](https://github.com/oxc-project/tsrs/blob/main/docs/STATUS.md)
- [Engineering notes](https://github.com/oxc-project/tsrs/blob/main/notes/README.md)

## Contributing

The [issue tracker](https://github.com/oxc-project/tsrs/issues) is open for bug reports. Please include a minimal reproduction.
Pull requests remain closed because we don't have the bandwidth to review them.

See the [contributing guide](https://github.com/oxc-project/tsrs/blob/main/CONTRIBUTING.md)
for building from source and running the suites, the [Code of Conduct](https://github.com/oxc-project/tsrs/blob/main/CODE_OF_CONDUCT.md), and
[security policy](https://github.com/oxc-project/tsrs/blob/main/SECURITY.md) for reporting vulnerabilities.

## Acknowledgments

We thank the following organizations and contributors:

<p>
  <a href="https://depot.dev">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/oxc-project/tsrs/main/assets/sponsors/depot-dark.svg">
      <img alt="Depot" src="https://raw.githubusercontent.com/oxc-project/tsrs/main/assets/sponsors/depot-light.svg" height="32">
    </picture>
  </a>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://www.owner.com">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/oxc-project/tsrs/main/assets/sponsors/owner-dark.svg">
      <img alt="Owner.com" src="https://raw.githubusercontent.com/oxc-project/tsrs/main/assets/sponsors/owner-light.svg" height="32">
    </picture>
  </a>
</p>

- [Depot CI](https://depot.dev) for sponsoring our continuous integration.
- [Owner.com](https://www.owner.com) for sponsoring the initial work.
- [Max Schwenk](https://github.com/maschwenk) for the port.
- [@walkerdb](https://github.com/walkerdb) for improvements to the project.
- [@JiangWeixian](https://github.com/JiangWeixian) for transferring the `tsrs` npm package name.

We extend special thanks to the [TypeScript team](https://www.typescriptlang.org/) for their support.

## License

`tsrs` is a derivative of TypeScript (Copyright Microsoft Corporation), licensed under [Apache-2.0](https://github.com/oxc-project/tsrs/blob/main/LICENSE).
The bundled `lib.*.d.ts` files are TypeScript's. See [NOTICE](https://github.com/oxc-project/tsrs/blob/main/NOTICE) for attribution.
Not affiliated with or endorsed by Microsoft.
