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

All three worlds import `diagnostics` and `protected-material`. The host rejects a legacy WIT package by inspecting the component type before linking or running start/info code; the typed incompatibility identifies the found package and the required `carina:provider@0.2.0` / protocol 25. Guest stdout and stderr are discarded by the host; diagnostic events bind through the host's redaction boundary. Only long-lived AWS session credentials and GitHub tokens require registration before adoption. Registration is idempotent by value and retains earlier generations. Per-request signing headers and query values are masked structurally in HTTP events, then pass through the session redactor; request preparation does not register them. Each request emits one completed HTTP event with headers, timings and status or transport code, including early failures.

The public AWS credential/configuration entry points are `AwsSessionMaterial`, `AwsSessionSource`, `AwsSessionFuture` and `AwsSdkParts`. Registration and acknowledged snapshots stay internal, with no snapshot credentials-provider or unsigned request-preparation API.

`AwsSdkParts::new(source, diagnostics)` takes an `AwsSessionSource` (SDK feature `aws`) and keeps its registering credentials provider and HTTP client private. Every credentials lookup fetches the current material, registers it before returning signing credentials, and preserves its optional expiry. Registration failures become typed credentials errors with their source chain; older generations remain protected. Its consuming `apply` method installs both together on one AWS SDK config builder; the SDK exposes neither component independently. Direct WASI AWS client constructors remain private. This guarantees paired installation on that builder, while later changes through AWS APIs remain under caller control.

Core owns event and registration budgets. Event budgets reset for each WASM export and each native provider dispatch through the same core call scope, with separate count/rate quotas for Error/Warn/Info and Debug/Trace. Only the first panic in a call is exempt from count/rate limits; later panics consume the ordinary quota. Panic context, message and location share a 256 KiB input cap. Over-cap guest text is masked before binding, retaining numeric location fields. The host emits one fixed suppression notice at call completion containing only the total count.

Level filtering happens at admission: invisible events consume no budget and add nothing to the suppression count. The default sink shows Error/Warn/Info; Debug/Trace require the existing `CARINA_WASI_HTTP_TRACE=1` host opt-in, which is never forwarded to guests. Custom sinks explicitly observe all levels and use the same separate quotas. Sink and validated schema context survive fallback calls and native custom-type validation. The host selects a sink on a consumed `ProviderSessionBuilder` before constructing and sharing the session; session clones cannot change it. Session construction and call-budget/presentation plumbing are exposed by `provider-host` (`guest-runtime` includes it for the native plugin host); WASM SDK guests enable neither feature. Enabling `provider-host` on wasm32 is a compile error, including through transitive dependencies. The in-repo manifest check remains an additional lint; trusted native in-process providers accept host feature unification. Native calls share per-call budgeting and the count-only end-of-call notice. Detached native tasks and other threads share a bounded session budget, whose suppressed count is delivered as one fixed host notice when the last diagnostic owner drops. Emission handles are weak; native OS output is outside this channel.

## Using it

Provider authors do not depend on this repository directly. The Carina SDK (`carina-plugin-sdk` in the [carina](https://github.com/carina-rs/carina) repository) includes these definitions as a submodule, generates the bindings, and exposes them through `export_provider!`. A provider pins one Carina revision, and that revision fixes the WIT version it is built against.

See [Writing a Provider](https://carina-rs.dev/providers/writing/getting-started/) to build one.

## Changing the interface

A change here is a protocol change. It lands in this repository first, then in `carina` as a submodule bump together with a protocol version bump, and then in each provider when it moves to the new Carina revision.
