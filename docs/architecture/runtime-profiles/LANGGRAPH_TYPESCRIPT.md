# TypeScript / LangGraph Runtime Profile

**Profile ID:** `RUNTIME-LANGGRAPH-TS`  
**Profile Version:** `1.0.0`  
**Core Specification:** `AI-HARNESS-ARCH 2.0.0`  
**Language:** TypeScript / JavaScript  
**Runtime Framework:** LangGraph

This profile maps the runtime-neutral [AI Workflow Harness Architecture Specification](../AI_HARNESS_ARCHITECTURE.md) onto a TypeScript implementation using LangGraph.

The profile is implementation guidance, not a replacement for the core specification. LangGraph-native features MAY be used wherever they preserve the core harness semantics. Runtime convenience MUST NOT weaken graph contracts, state ownership, tool-effect restrictions, or authorization boundaries.

## 1. Concept mapping

| Harness concept | TypeScript / LangGraph mapping |
| --- | --- |
| Graph | `StateGraph` or another compiled/invocable LangGraph workflow unit |
| Node | LangGraph node implemented by deterministic code, configured model invocation, tool adapter, or subgraph call |
| Graph state | Explicit LangGraph state schema/channels owned by the graph |
| Public graph contract | Runtime-validated input/output schemas surrounding graph invocation |
| Subgraph | Compiled graph invoked/composed by a parent through an explicit input/output mapping |
| Orchestrator Graph | LangGraph workflow responsible for sequencing, routing, fan-out/fan-in, and child invocation |
| Task Graph | LangGraph workflow with explicitly authorized task capabilities |
| Context Graph | LangGraph workflow restricted to `pure` and `read` domain capabilities |
| Tool | Guarded tool wrapper or deterministic capability adapter preserving the core `ToolEffect` metadata |
| Functional Core | Ordinary pure TypeScript modules with no LangGraph dependency |
| Execution context | Trusted runtime/configuration data passed separately from untrusted graph input/state where possible |
| Checkpoint/recovery | LangGraph checkpointer using stable thread identity and deliberate checkpoint namespace behavior |
| Pause/resume | LangGraph interrupt/resume facilities with authenticated external resumption |
| Artifact persistence | Separate scoped artifact storage; not ordinary graph state and not assumed to be the LangGraph checkpointer |
| Model configuration | Configuration-resolved provider/model instances created during graph/node assembly or controlled lazy initialization |
| Observation | LangGraph/LangSmith, Langfuse, OpenTelemetry, or custom instrumentation that preserves state/privacy rules |

A `Task Graph` in this architecture is not synonymous with any LangGraph helper named `task`. Graph categories are semantic categories defined by the core specification.

## 2. Dependency boundaries

LangGraph-specific execution types MAY appear inside workflow implementation and runtime-integration modules.

They SHOULD NOT leak into:

- public domain contracts,
- artifacts intended for cross-stack consumption,
- Functional Core functions,
- reusable tool result contracts.

Public contracts SHOULD be represented as ordinary serializable values validated by the repository's schema mechanism.

The Functional Core MUST remain independent of LangGraph, provider SDKs, live clients, and effectful infrastructure.

## 3. Graph construction

A typical assembly flow SHOULD:

1. validate configuration,
2. resolve provider/model configuration,
3. instantiate configured model clients,
4. construct authorized tools and effect guards,
5. construct nodes and subgraphs,
6. validate parent/child contract compatibility,
7. configure checkpointing and runtime context when required,
8. compile the LangGraph graph.

Models remain configuration concerns. A profile implementation MAY use model factories or provider helpers, but it MUST NOT require an additional conceptual Model Layer merely because construction is centralized.

Model/client instances, credentials, sockets, functions, and other live objects MUST NOT be persisted as graph state.

## 4. Graph-category enforcement

### Orchestrator Graph

An Orchestrator MAY use LangGraph routing, commands, branching, fan-out/fan-in, configured models, and subgraphs.

It MUST NOT bind or directly invoke domain tools for convenience. Domain reads belong in Context Graphs and domain actions belong in Task Graphs.

### Context Graph

A Context Graph MAY bind only capabilities classified as:

- `pure`
- `read`

The implementation MUST reject `write`, `external-communication`, and `destructive` bindings during graph assembly or capability registration.

The tool invocation boundary MUST enforce the same restriction at runtime. Prompt instructions alone are not enforcement.

A Context Graph MUST NOT bypass restrictions by calling a provider-hosted action, raw SDK, subprocess, callback, or nested Task Graph.

### Task Graph

A Task Graph MAY receive the explicitly authorized capabilities required by its contract.

All domain effects MUST pass through tools so that validation, effect classification, resource scoping, authorization, approval, auditing, retry policy, and outcome semantics are consistently applied.

## 5. State and subgraphs

Each graph owns its internal state schema and update semantics.

Reusable subgraphs SHOULD communicate through explicit parent/child input-output mappings rather than depending on shared internal channels.

Shared state channels MAY be used only when the shared fields are intentionally part of the composition contract and their merge/update behavior is documented and tested.

State SHOULD be bounded and checkpoint-serializable. Durable reports, evidence sets, large datasets, generated files, and source snapshots belong in artifact storage and cross graph boundaries as artifact references.

Reducers MUST be deterministic and tested. Parallel state updates require deliberate conflict/merge behavior; correctness MUST NOT depend on incidental completion order.

## 6. Runtime identity

A recoverable LangGraph deployment SHOULD maintain distinct identities for:

- application workflow/business-process identity,
- LangGraph thread/checkpoint lineage,
- individual execution/run attempts,
- logical effectful operations.

A resume MUST address the intended persisted thread/checkpoint lineage rather than create an unrelated new thread.

A logical domain-action `operationId` MUST remain stable across retries of the same intended action. A fresh run identifier is generally not an adequate idempotency key.

## 7. Checkpointing and recovery

A deployment claiming process-loss recovery MUST use a persistent LangGraph checkpointer appropriate to its topology. In-memory checkpointing is suitable for tests or explicitly non-durable deployments only.

Checkpointing is runtime bookkeeping, not an authorization mechanism and not automatically the artifact system.

A checkpoint does not guarantee exactly-once execution of external effects.

The implementation MUST account for the failure window:

1. a Task Graph invokes an external mutation,
2. the remote system commits it,
3. the worker fails before successful completion is persisted,
4. execution resumes.

Mutating tools therefore require a declared policy such as:

- provider-supported idempotency key,
- atomic deduplication/transactional operation record,
- reconciliation before retry,
- `never-auto-retry` with an explicit `unknown` outcome.

A timeout MUST NOT be interpreted as proof that a remote operation did not commit.

## 8. Node replay and durable boundaries

Developers MUST assume node code can execute again after interruption, failure, or recovery depending on the selected LangGraph execution mode and persistence configuration.

Non-idempotent domain operations SHOULD be isolated behind deliberate durable boundaries rather than grouped casually with unrelated effects.

Where reproducibility across recovery matters, nondeterministic results such as model decisions or external reads SHOULD be captured at an appropriate durable boundary or as immutable evidence/artifacts instead of silently re-observed and treated as the same fact.

Pure Functional Core code may be recomputed freely.

## 9. Interrupts, approvals, and resume

Human approval SHOULD occur before the protected effect.

An approval record SHOULD identify:

- proposed operation,
- relevant arguments,
- target resource/scope,
- approving principal,
- timestamp/version information needed for audit.

The application boundary MUST authenticate and authorize the party resuming an interrupted workflow. A model-generated field such as `approved: true` is never sufficient.

If an approval changes the proposed operation or material arguments, those values MUST be revalidated and reauthorized before execution.

Code before an interrupt may execute again after resume. Non-idempotent external work MUST NOT be placed before an interrupt without explicit replay protection.

Framework interrupt/control-flow signals MUST NOT be swallowed and normalized into ordinary business-success results.

## 10. Cancellation and limits

Runtime configuration SHOULD define finite applicable limits for:

- graph/node iterations,
- model invocations,
- tool calls,
- retries,
- elapsed duration,
- concurrency,
- input/output volume.

Delegation to subgraphs MUST preserve overall budgets rather than implicitly grant each child an unlimited new allowance.

Cancellation/deadlines SHOULD propagate into supported model and tool operations.

If cancellation races with an external effect, the resulting contract MUST preserve the possibility that the remote action already committed.

## 11. Artifact handling

The LangGraph checkpointer MUST NOT be treated as the default storage location for durable workflow artifacts merely because it can persist graph state.

Artifact storage SHOULD expose a scoped interface that:

- restricts destinations,
- enforces caller/principal scope,
- commits data before returning a reference,
- records artifact identity/version and provenance,
- prevents credentials from being embedded in references.

Publishing an artifact to a business repository is a Task Graph `write`; storing run-owned evidence in approved harness artifact storage is harness bookkeeping.

## 12. Observability

Observation MAY use LangSmith, Langfuse, OpenTelemetry, custom logs, or another configured backend.

Instrumentation SHOULD correlate:

- workflow/thread/run identity,
- graph and node identity,
- configured model/provider identifiers,
- tool calls and effects,
- effect outcomes,
- artifact references,
- relevant usage/latency metadata.

Observability MUST NOT expose secrets, unrestricted private graph state, or unreviewed prompt/model content to unauthorized consumers.

Mandatory audit evidence for privileged domain actions MAY have stricter durability/retention requirements than ordinary debugging traces.

## 13. Conformance declaration

Using LangGraph does not automatically make a deployment `L2 — Recoverable` or `L3 — Durable Distributed`.

The deployment MUST declare and test the actual topology and persistence configuration.

### L1 — Basic

A TypeScript/LangGraph deployment may claim L1 when architectural restrictions and execution semantics are enforced but no process-loss recovery guarantee is made.

### L2 — Recoverable

A deployment may claim L2 only after tests demonstrate at least:

- persistent checkpointer survives process loss,
- stable thread identity resumes the intended workflow,
- parent/child state boundaries remain valid after resume,
- approval interrupts survive restart where promised,
- external-effect retry/reconciliation behavior is safe,
- graph/configuration versions needed for resume are recorded or otherwise controlled.

### L3 — Durable Distributed

This profile does not assume LangGraph alone supplies L3 guarantees. A deployment claiming L3 MUST document and test the surrounding scheduling, worker ownership, concurrency, recovery, and version-management mechanisms that provide distributed durability.

## 14. Compatibility and upgrades

Persisted executions may outlive the code that created them.

Changes to any of the following require a resume-compatibility assessment:

- state schema,
- reducers,
- routing,
- node identity/topology,
- subgraph composition,
- public contracts,
- configured tool capabilities,
- pending effect semantics,
- checkpoint serialization,
- configured model/provider behavior where interpretation depends on it.

A checkpoint does not pin application code, model behavior, or live remote data. Starting a fresh run against current dependencies is not equivalent to deterministic replay of a historical run.

Repositories SHOULD pin LangGraph and related dependencies and test runtime behavior against those pinned versions before adopting APIs from newer documentation.

## 15. Minimum profile tests

A conforming TypeScript/LangGraph implementation SHOULD include tests demonstrating:

- Context Graph rejects write-tool binding,
- deterministic code cannot bypass tool-effect enforcement,
- Orchestrator cannot directly invoke domain tools,
- invalid graph and tool contracts fail at boundaries,
- parent does not depend on child-private state,
- parallel reducers behave deterministically,
- artifact references are returned only after successful commit,
- unauthorized artifact access fails,
- configured model instances are not persisted in state,
- process restart resumes the correct thread when recovery is claimed,
- approval/rejection/resume paths behave correctly,
- unauthorized resume is rejected,
- external commit followed by lost local completion record does not trigger a blind duplicate action,
- cancellation preserves uncertain-effect semantics,
- exhausted retry/budget limits terminate predictably,
- observation does not expose private state/secrets.

## 16. Adoption checklist

Before declaring this profile adopted:

- [ ] Record Node.js, TypeScript, LangGraph, LangChain/provider package versions.
- [ ] Record schema-validation library and public contract conventions.
- [ ] Record checkpoint backend and durability claim.
- [ ] Record artifact backend and reference format.
- [ ] Record ToolEffect enforcement mechanism.
- [ ] Record approval/resume authentication path.
- [ ] Record observability backend and redaction policy.
- [ ] Execute the profile's conformance/fault tests.
- [ ] Declare the verified L1/L2/L3 deployment level.

---

## Profile principle

> **Use LangGraph as the execution mechanism, not as the definition of the architecture.**

The TypeScript implementation is conforming when it preserves the same Orchestrator, Task, Context, Tool, Functional Core, contract, state, artifact, and effect semantics that another stack would implement differently.
