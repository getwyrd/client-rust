# Contributing to client-rust

We LOVE your input! We want to make contributing to this project as easy and transparent as possible, whether it's:

- Reporting a bug
- Submitting a fix
- Proposing a new feature
- Discussing the current state of the code
- Adding tests, examples, or documentation

## Submitting Pull Requests

Pull Requests are the best way to propose changes to the codebase. We actively welcome your Pull Requests.

If you are contributing to an open source project for the first time, please read the [detailed how-to guide](https://github.com/firstcontributions/first-contributions).

It is important to note that we enforce the [Developer Certificate of Origin](https://developercertificate.org/) (DCO) on Pull Requests: every commit must carry a `Signed-off-by` line. Use `git commit -s` and [read more about how the check works](https://github.com/apps/dco).

What CI requires of every PR:

- Builds without warnings — `RUSTFLAGS=-Dwarnings`, and both rustfmt and clippy are enforced (`make check` runs all of it).
- Unit tests pass (`make unit-test`), and integration tests pass against a real TiKV cluster.
- Committed generated code is in sync: if you change anything under `proto/`, run `make generate` and commit the result — CI fails on drift.
- A DCO sign-off on every commit (see above).

Reviews and merges are handled by the TiKV community bot: a PR merges once reviewers and approvers from [`OWNERS`](OWNERS) have added the `lgtm` and `approved` labels. At least one review is always required. If any of this is difficult for you, don't worry about it and ask on the PR.

A few conventions:

- Title your PR and commits as `component: short description` — for example `transaction: resolve orphaned locks whose primary was never written` or `store: make grpc_max_decoding_message_size configurable`. Use `*:` for changes that span components.
- Please follow PingCAP's [Rust style guide](https://pingcap.github.io/style-guide/rust/).
- Code PRs should include new tests or test cases.

## Building and testing

The repository pins a stable Rust toolchain in [`rust-toolchain.toml`](rust-toolchain.toml); rustup picks it up automatically, so a plain `cargo build` just works. The minimum supported Rust version is the `rust-version` field in [`Cargo.toml`](Cargo.toml).

We use [nextest](https://nexte.st) as the test runner:

```
cargo install cargo-nextest --locked
```

The `Makefile` wraps the common workflows:

- `make check` — regenerates protos, type-checks all targets, and runs rustfmt + clippy with warnings as errors. Run this before pushing.
- `make unit-test` — unit tests, no cluster needed.
- `make generate` — regenerates `src/generated/` from `proto/` (commit the output).
- `make doc` — builds the API documentation.

Integration tests need a running TiKV cluster. The easiest way is [TiUp](https://github.com/pingcap/tiup) (>= 1.5); `make tiup` starts a local 3-node playground with the repository's config, then:

```
make integration-test        # or integration-test-txn / integration-test-raw
```

`PD_ADDRS` (default `127.0.0.1:2379`) tells the tests where PD is. You probably won't need integration tests for your first few PRs.

## Reporting issues

We use GitHub to track public bugs and issues. Report an issue by [opening a new issue](https://github.com/tikv/client-rust/issues/new).

**Great Bug Reports** tend to have:

- A quick summary and/or background
- Steps to reproduce
  - Be specific!
  - Give sample code if you can
- What you expected would happen
- What actually happens
- Notes (possibly including why you think this might be happening, or stuff you tried that didn't work)

If you tend to use Chinese to propose issues, [asktug](https://asktug.com/) is also a good choice.

## Getting help

If you need help, either to find something to work on or with any technical problem, the easiest way to get it is via [internals.tidb.io](https://internals.tidb.io), the forum for TiDB developers.

You can also ask in Slack — we monitor the #client-rust channel on the [tikv-wg slack](https://tikv.org/chat). Or ask directly on GitHub issues and PRs.
