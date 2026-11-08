# carina-plugin-wit

[WIT](https://component-model.bytecodealliance.org/design/wit.html) definitions for the interface between the [Carina](https://github.com/carina-rs/carina) host and its provider plugins.

Carina providers are WebAssembly components. This package, `carina:provider`, is the contract they are built against: what a provider exports (schemas, validation, read / create / update / delete) and what the host offers it in return.

## Contents

| File | What it defines |
|------|-----------------|
| [`wit/types.wit`](wit/types.wit) | Shared data types: values, resources, schemas, effects, and errors |
| [`wit/provider.wit`](wit/provider.wit) | The `provider` interface every provider exports |
| [`wit/diagnostics.wit`](wit/diagnostics.wit) | Typed diagnostic events and confidential protected-material registration |
| [`wit/world.wit`](wit/world.wit) | The worlds a provider can target, and host imports such as `aws-credentials` |

The worlds are:

- `carina-provider` — a provider with no network access
- `carina-provider-with-http` — a provider that calls HTTP APIs through `wasi:http`, limited to the hosts it declares
- `carina-provider-aws` — an HTTP provider that also receives AWS credentials resolved by the host

All three worlds import `diagnostics` and `protected-material`. Guest stdout and stderr are discarded by the host; diagnostic events bind through the host's redaction boundary. Credential and signing material must be registered before use.

## Using it

Provider authors do not depend on this repository directly. The Carina SDK (`carina-plugin-sdk` in the [carina](https://github.com/carina-rs/carina) repository) includes these definitions as a submodule, generates the bindings, and exposes them through `export_provider!`. A provider pins one Carina revision, and that revision fixes the WIT version it is built against.

See [Writing a Provider](https://carina-rs.dev/providers/writing/getting-started/) to build one.

## Changing the interface

A change here is a protocol change. It lands in this repository first, then in `carina` as a submodule bump together with a protocol version bump, and then in each provider when it moves to the new Carina revision.
