# Juju API Contract, Generation and SDK Plan

Status: proposed direction, based on the initial design discussion.

## Goal

Make Juju's existing API an explicitly defined, independently consumable contract. Generate wire types, low-level clients and server bindings from that contract, and provide shared handwritten client workflows in an SDK.

Retain the existing wire protocol and facade behaviour. This is not a proposal to replace JSON-RPC with HTTP endpoints, redesign facades or introduce new facade versions merely to adopt code generation.

Make each step independently testable and deployable. Server migration, client migration, SDK workflow extraction and CLI repository extraction are separate changes, not a single coordinated switchover.

## Repository Responsibilities

### This Repository

- Authoritative, expressive definitions of facade versions and wire payloads.
- Shared definitions of the RPC protocol and watcher lifecycle contracts.
- Generators and generated wire types and low-level clients for Go and TypeScript/JavaScript, with Python as a possible additional client target.
- Go server interfaces and dispatch bindings. Juju and JIMM are Go servers, so other language targets do not need generated server stubs.
- Shared protocol definitions and Go client runtime abstractions independent of controller internals. Language-specific client runtimes may remain in their existing client packages.
- Handwritten SDK operations, compatibility helpers and custom clients where useful.
- Temporary reflection exports and import tooling during migration.
- Contract validation, compatibility checks and generator tests.

Start with the SDK in this repository, in distinct packages from generated bindings. Separate modules or repositories can follow if ownership or release cadence warrants them; they are not prerequisites.

### Juju Controller Repository

Retain facade implementations, authorisation, domain services, workers and controller-specific dependency injection. Import generated server bindings and wire types, adapting them to internal types where necessary.

Do not make this repository import Juju controller internals. A transitional adapter in Juju or a consumer may bridge existing connection and parameter types without introducing that reverse dependency.

### TypeScript Client (`juju/js-libjuju`)

Retain `@canonical/jujulib` as the dashboard's TypeScript client package. Keep its browser WebSocket transport, login, bakery authentication, connection lifecycle and public API while migrating facade types and calls to generated outputs from this repository. Do not create a parallel dashboard client or replace `js-libjuju`'s runtime as part of the initial migration.

Its current Juju schema snapshots, importer and generator overrides are useful migration evidence and implementation material, but must not remain a competing long-term source of truth. Adapt or retire them as the authored contract becomes authoritative.

### Consumer Repositories

- CLI: commands, prompts, output, local store, current controller/model and credential persistence policy.
- Terraform provider: planning, reconciliation, Terraform state and resource lifecycle semantics.
- JIMM: authorisation policy, controller selection, proxying and bookkeeping.
- Juju Dashboard: UI workflows, Redux integration and dashboard-specific state.

Consumers supply explicit connection settings and credentials. The SDK must not implicitly read the CLI store. Shared connection and authentication behaviour can live in the SDK without owning consumer persistence policy.

## API Definitions

Identify each operation by facade name, version and method. Define request and response payloads using JSON Schema or an equivalent language-neutral schema. Choose the precise operation format through a small working example; evaluate OpenRPC against Juju's actual protocol rather than assuming it is a direct fit.

Keep descriptions, constraints and semantics alongside the operations and fields they describe. Do not require authors to assemble the contract from reflection data and separate semantic overlays.

The contract needs to express:

- Field presence, nullability, empty values, defaults and meaningful bounds.
- RPC errors, per-item errors and partial success.
- Side effects, ordering, completion and safe retry behaviour.
- Method availability across facade versions.
- Watcher creation, IDs, initial events, blocking Next calls, Stop, cancellation and disconnect cleanup, according to each watcher's actual behaviour.

Define shared protocol rules once: connection and login, controller/model scope, facade discovery, request IDs, object IDs, envelopes and errors.

Do not put Go pointers or integer implementation widths into the language-neutral contract solely to reproduce existing declarations. Numeric bounds belong in the contract where observable; Go binding choices belong in generation policy or binding-specific settings. Use adapters when existing signatures differ.

Generation must preserve actual JSON behaviour, including custom encodings. Neither a reflected schema nor ordinary struct shape alone proves equivalence. Do not introduce stricter runtime validation merely because a schema can express it: existing accepted inputs and server validation behaviour must be preserved.

## Generated and Handwritten Client Surface

Expose generated facade clients directly under an explicitly low-level namespace. Add handwritten operations only where there is real reusable logic.

Illustrative API, not a committed Go design:

```go
// Shared SDK workflow.
result, err := client.Deploy(ctx, options)

// Direct facade RPC with generated wire types.
response, err := client.RPC.Application.Deploy(ctx, request)
```

Do not create layers that each mirror every generated method. Keep conversions and exceptional compatibility handling in focused helpers or selected SDK operations. Routine facade negotiation may be generated or handled by the shared runtime when the definitions establish equivalent calls.

Workflows such as deployment can compose generated calls and custom transfer clients. Their contracts must define cancellation, partial success, returned resource identities and retry safety. Separate acceptance from readiness; callers must be able to choose whether to wait. Never blindly retry a partially completed workflow or promise rollback that the server does not support.

Server-side correctness and policy enforcement remain authoritative. Client validation is convenience, not a security or consistency boundary.

## Compatibility

Compatibility is an ongoing requirement for the CLI, Terraform, JIMM and dashboard, not a temporary bridge between major Juju releases.

- Preserve existing clients talking to migrated servers.
- Preserve new clients talking to supported older controllers, including the required Juju 4 CLI to Juju 3.6 server compatibility.
- Cover ongoing differences between 4.0, 4.1, 4.2 and subsequent releases.
- Preserve CLI syntax, output, exit codes and local-store behaviour when the client implementation or repository changes.
- Preserve dashboard connection, authentication, facade discovery and unsupported-facade behaviour while migrating generated TypeScript clients.
- Treat Go import and type migration as explicit source changes; wire compatibility does not imply Go source compatibility.

Discover supported facade versions and select mutually supported contracts. Keep version-specific bindings available underneath any negotiated surface. Do not silently rebind an operation to a new incompatible shape when generating a newer library.

Use handwritten adaptations only where versions differ meaningfully. Select based on advertised API support where sufficient, not scattered release-number checks. A new capability, such as SSH proxy support, must return a clear unsupported-feature error on older controllers unless an equivalent fallback actually exists. Not every capability is fully described by facade discovery.

The definition repository may retain more versions than any one server serves. The client support matrix and server registrations are separate selections. Specify the complete supported controller range before claiming compatibility.

## Reflection Import During Transition

Use Juju's current reflection/schema machinery as the structural baseline. Export every required facade version, not only the latest, and include Admin and relevant agent facades. Capture pinned revisions from Juju 3.6 and the relevant 4.x branches.

Commit deterministic reflection exports with source revision and exporter provenance. These are review evidence, not the authored API contract.

For subsequent updates:

1. Re-export from a recorded Juju revision.
2. Diff against the committed baseline.
3. Manually update the expressive definition and review behaviour changes.
4. Regenerate and run relevant compatibility checks.
5. Commit the updated evidence and definition together.

Flag changes within an existing facade version for investigation. Absence from a newer export must not delete contracts needed by supported older clients or controllers. Preserve historical versions in the authoritative definitions.

Track ownership per facade version. Before migration, reflection supplies structural evidence; after migration, changes are definition-first. Do not allow a later import to overwrite an owned contract silently.

Delete reflection snapshots and the importer once all required contracts are captured and API development is definition-first. Retain historical contracts for the client support range, independent of deleting migration tooling.

## Incremental Server Migration

Add generated registration/dispatch support alongside existing reflection. Select exactly one path for each facade name/version; avoid ambiguous fallback that could expose unintended methods or mask registration errors.

Generated server interfaces will normally correspond to facade versions. One existing implementation may satisfy several interfaces. Do not require another handwritten implementation layer unless an adapter has a concrete purpose.

Keep existing construction, authorisation, connection ownership and object caching/lifecycle machinery where practical. Account for pre-login Admin dispatch and watcher/object IDs, not only ordinary authenticated calls.

For each migration, define the unchanged contract, generate bindings, adapt the existing implementation, switch registration and test. Unmigrated versions keep using reflection. Remove legacy dispatch only when nothing depends on it.

Registering generated interfaces with existing reflection is a possible probe, not a mandatory extra phase or a substitute for generated dispatch.

## Incremental Consumer Migration

Generated clients must initially be able to share existing authenticated RPC connections through a small transport interface and consumer-side adapters. Migration must not require duplicate connections or authentication stacks.

For TypeScript, generated facade clients must fit `js-libjuju`'s existing transport and connection model. Preserve its login-time facade negotiation: intersect client-registered versions with versions advertised by the server, then select a supported version. Keep its browser transport and authentication runtime rather than duplicating them in the new repository.

Move one facade or workflow in one consumer at a time. Server and client migrations can happen independently because the wire contract does not change.

Preserve existing validation, conversions, error handling and version fallbacks explicitly. Move genuinely reusable behaviour into the SDK, while leaving consumer-specific policy and state in its owner. Remove old wrappers as their callers disappear, rather than maintaining permanent duplicate client APIs.

Replacing uses of Juju's API client packages does not by itself remove every dependency on the Juju module. Track connection code, transfer clients and other shared imports separately.

## HTTP, Streams and Extensions

Keep HTTP upload/download handlers, log streams and upgraded connections custom initially. They still need documented contracts and compatibility tests, but do not have to enter the initial facade definition language.

Watchers are in scope because they use facade RPC. Their generated calls and handwritten lifecycle helpers must preserve the actual protocol.

Support separately owned extensions such as the JIMM facade using the same generators and connection runtime. JIMM-specific definitions may remain owned by JIMM while using the shared definition format and generators. Establish early how its generated Go server bindings and TypeScript facade client are published and consumed. The dashboard currently hand-rolls JIMM facade methods and types; replace those with generated bindings. Separately generated clients must share a connection without hard-coding every extension into Juju or making Juju implement it.

## Delivery Sequence

1. Establish repository structure, ownership rules and the compatibility matrix.
2. Capture and commit deterministic reflection baselines from pinned releases.
3. Select a small Juju facade version with simple payloads and no watcher lifecycle. Author its expressive contract and the minimum shared protocol definition.
4. Generate wire types, a low-level client and Go server interface/dispatch bindings. Prove the generated client against an unchanged Juju controller, and prove an existing client against Juju's generated server binding.
5. Add generated TypeScript output to `js-libjuju` while preserving its transport and public connection API. Exercise facade negotiation against supported Juju versions.
6. Early cross-project proof: generate Go server bindings for one Juju facade and one JIMM facade, then call both through the TypeScript client from the dashboard. Replace one dashboard-owned JIMM facade method/type in this proof. This verifies separately owned extensions and the actual dashboard path; it does not require migrating all JIMM facades.
7. Migrate one CLI, Terraform or JIMM Go client call, preserving its current handwritten behaviour.
8. Exercise multiple facade versions and materially different wire shapes against supported controllers, then add a watcher case.
9. Expand facade coverage and independently migrate the CLI, Terraform, JIMM and dashboard. Add Python client generation when a consumer and validation target are identified.
10. Extract shared workflows, using deployment as a substantive early example.
11. Move the CLI into its own repository when ready; retain its local store.
12. Make Juju shell tests use a pinned built CLI, as JIMM already does. Support testing unreleased CLI/controller changes and released compatibility pairs.
13. Remove legacy wrappers, dispatch and import tooling after their respective migration exit criteria are met.

Steps overlap where independent; CLI extraction is not required to prove the API generation approach. Include Juju's non-CLI API consumers, such as agents and workers, before declaring its old client layer removable.

## Verification and Exit Criteria

The first milestone is one unchanged RPC working with generated code on either end independently. It must not depend on a facade redesign or SDK workflow rewrite.

Verify:

- Deterministic generation and clean regeneration checks.
- Wire equivalence for populated, omitted, null and empty representations, custom encodings, unknown fields and malformed requests.
- RPC failures, per-item failures, partial success and unsupported operations.
- Mixed generated/legacy registration and shared client connection behaviour.
- Existing client against generated server, and generated client against existing servers in the declared support matrix.
- Authentication, cancellation, concurrent calls, watcher lifecycle and cleanup for the slices that exercise them.
- Consumer-level behaviour for each migrated command, resource or JIMM path.

Use focused tests per migration and broader compatibility gates for shared runtime changes. Full CLI extraction additionally requires shell-test coverage and a defined cross-repository build/release process.

## Open Decisions

- Exact definition syntax and suitability of OpenRPC for Juju's envelopes.
- Generated package layout and policy for sharing types across facade versions.
- Negotiated client surface versus explicit version access when shapes differ.
- Transport/runtime extraction boundary and treatment of existing custom codecs.
- JIMM definition ownership and publication of its generated TypeScript client.
- Whether to add Python client generation and which consumer will validate it.
- First facade, first consumer and exact controller support matrix.
- Module versioning, release cadence and cross-repository CI ownership.

## Existing Implementation Anchors

- [Facade registry](../juju/apiserver/facade/registry.go)
- [Server method lookup, construction and dispatch](../juju/apiserver/root.go)
- [Supported reflected method shapes](../juju/rpc/rpcreflect/type.go)
- [Application facade registrations](../juju/apiserver/facades/client/application/register.go)
- [Existing client schema export](../juju/apiserver/facades/schema.json)
- [Existing agent schema export](../juju/apiserver/facades/agent-schema.json)
- [Reflection schema generator](../juju/generate/schemagen/gen/generator.go)
- [Current application client](../juju/api/client/application/client.go)
- [Terraform model operations](../terraform-provider-juju/internal/juju/models.go)
- [JIMM model client wrappers](../jimm/internal/jujuclient/modelmanager.go)
- [js-libjuju connection and facade negotiation](../js-libjuju/api/client.ts)
- [js-libjuju facade generator](../js-libjuju/generator/generator.ts)
- [js-libjuju Juju schema history](../js-libjuju/generator/schema/schema-history.json)
- [js-libjuju generator overrides](../js-libjuju/generator/overrides.ts)
- [Dashboard-owned JIMM facade methods](../juju-dashboard/src/juju/jimm/JIMMV4.ts)