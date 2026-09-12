# AI Workflow Harness Architecture Specification

**Specification ID:** `AI-HARNESS-ARCH`  
**Version:** `2.0.0`  
**Status:** Runtime-neutral baseline  
**Runtime:** implementation-defined  
**Language:** implementation-defined

> **Orchestrators decide. Task Graphs perform work. Context Graphs inform. Tools provide capabilities. The Functional Core computes.**

This specification defines a runtime- and language-independent architecture for model-driven and deterministic workflow harnesses. It defines responsibilities, contracts, state ownership, capability restrictions, artifact handling, and runtime obligations that must remain stable across implementations.

Architecture defines **what is permitted**. A runtime profile defines **how a selected stack executes it**.

## Contents

1. [Scope](#1-scope)
2. [Core components](#2-core-components)
3. [Architectural invariants](#3-architectural-invariants)
4. [Graph responsibilities](#4-graph-responsibilities)
5. [Contracts](#5-contracts)
6. [State and artifacts](#6-state-and-artifacts)
7. [Tools and effects](#7-tools-and-effects)
8. [Functional Core](#8-functional-core)
9. [Configuration](#9-configuration)
10. [Runtime contract](#10-runtime-contract)
11. [Conformance levels](#11-conformance-levels)
12. [Cross-cutting requirements](#12-cross-cutting-requirements)
13. [Testing](#13-testing)
14. [Runtime profiles](#14-runtime-profiles)

## 1. Scope

This architecture governs workflow graphs, graph nodes, tools, pure computation, configuration, contracts, state, artifacts, and the execution behavior required of a conforming runtime.

The architecture is intentionally **runtime-framework agnostic** and **programming-language agnostic**. A conforming implementation may use a graph framework, durable workflow engine, state machine, ordinary functions, or a custom executor.

Portability does **not** require identical source code, a universal executable graph format, a common runtime abstraction, or interchangeable persisted checkpoints. Portability requires preservation of the specified semantics.

`MUST` and `MUST NOT` are normative requirements. `SHOULD` identifies the preferred implementation unless there is a documented reason to deviate. `MAY` identifies an allowed option.

## 2. Core components

| Component | Responsibility | Boundary |
| --- | --- | --- |
| **Configuration** | Defines model, provider, graph, tool, and runtime choices. | Cannot override architectural restrictions. |
| **Orchestrator Graph** | Coordinates subgraphs and owns workflow-level state. | Does not directly perform domain operations. |
| **Task Graph** | Performs one bounded unit of work. | May mutate domain systems only through authorized tools. |
| **Context Graph** | Acquires, transforms, and returns information. | May use only pure/read domain capabilities; no domain mutation. |
| **Tool** | Exposes an external or specialized capability. | Explicit contract, effect classification, and authorization. |
| **Functional Core** | Pure deterministic computation. | No I/O or ambient mutable state. |
| **Graph State** | Holds graph-owned execution information. | Private to the owning graph except declared outputs. |
| **Artifact** | Durable product, evidence, snapshot, or file. | Stored outside graph state and referenced by identity/version. |
| **Execution Runtime** | Schedules and executes workflow behavior. | Must preserve harness semantics and declared guarantees. |
| **Runtime Profile** | Maps the architecture to a concrete stack. | May narrow rules, but never relax the core. |

A **Graph** is a logical workflow component with an explicit contract and defined capability boundaries. It does not have to be represented internally as a literal node-edge graph.

## 3. Architectural invariants

**A01 — retired.** Version 1 selected LangGraph as the mandatory runtime. Version 2 is runtime-neutral.

**A02.** Model/provider selection and parameters MUST come from configuration. Models are configuration concerns, not a separate architectural layer.

**A03.** Orchestrators MUST coordinate domain work through subgraph contracts rather than directly invoking domain tools or systems.

**A04.** Context Graphs MUST NOT perform or delegate domain mutations. Their domain-facing capabilities are limited to `pure` and `read`.

**A05.** Task Graphs MUST represent bounded work. Domain effects MUST occur through tools.

**A06.** Every reusable graph and tool MUST expose explicit input/output contracts with runtime validation at public boundaries.

**A07.** A graph MUST own its internal state. Callers exchange declared inputs and outputs rather than depend on private child state.

**A08.** Durable artifacts MUST be stored outside graph state and tracked through references.

**A09.** Domain-facing external interactions MUST pass through tools. Model inference, checkpointing, scoped artifact persistence, telemetry, and trusted bootstrap operations are explicit harness/runtime exceptions.

**A10.** Tool effects and permissions MUST be enforced in code. Metadata and prompts alone are insufficient.

**A11.** Functional Core functions MUST be pure and deterministic for their explicit inputs.

**A12.** Graphs and tools MAY call the Functional Core directly. Pure computation does not require a tool wrapper.

**A13.** Harness persistence and telemetry exceptions MUST remain narrowly scoped and MUST NOT become a route for domain writes.

**A14.** Identity, authorization, secrets, and execution limits MUST originate from trusted configuration/context rather than model output or untrusted state.

**A15.** Effectful work MUST have an explicit replay/retry policy. Persistence does not imply exactly-once external effects.

**A16.** Model output and retrieved content are untrusted data. Schema validity is not factual correctness or authorization.

**A17.** Dependencies flow from workflow behavior toward capabilities and pure computation. Tools do not control their callers' routing.

**A18.** Execution MUST have explicit termination conditions and finite operational limits.

**A19.** Changes MUST include tests for affected contracts, capability restrictions, and replay/failure behavior.

**A20.** New graph categories or changes to these boundaries require an explicit architecture decision.

**A21.** Workflow semantics MUST remain independent of the selected execution runtime.

**A22.** A deployment MUST explicitly declare which runtime capabilities and durability guarantees it actually provides.

**A23.** Runtime-native features MUST NOT weaken harness restrictions or expose private graph state across stack boundaries.

## 4. Graph responsibilities

### Orchestrator Graph

An Orchestrator coordinates subgraphs and manages workflow-level state.

It owns sequencing, branching, fan-out/fan-in decisions, workflow completion, and failure decisions. It invokes child graphs through their declared contracts and merges only declared outputs into its own state.

An Orchestrator MAY use configured models for planning or routing, but model-generated plans are proposals that must be validated against allowed graph composition and policy.

**Rule of thumb:** Orchestrators decide **what happens**.

### Task Graph

A Task Graph performs a bounded unit of work under an explicit input/output contract.

A bounded task has a defined objective, completion conditions, permitted capabilities, operational limits, and declared result. It may be one node or many nodes.

A Task Graph MAY mutate domain systems, but only through appropriately classified and authorized tools. It MAY also be entirely deterministic or produce a deliverable without any domain mutation.

For effectful work, results MUST distinguish confirmed completion from intended action, partial completion, and unknown outcome when relevant.

**Rule of thumb:** Task Graphs **cause things to happen**.

### Context Graph

A Context Graph acquires, transforms, and returns information required by its caller under an explicit input/output contract.

It MAY search, retrieve, parse, extract, classify, summarize, correlate, or reason over information. It MAY use configured model inference, `pure`/`read` tools, and the Functional Core.

It MUST NOT invoke, wrap, or delegate a `write`, `destructive`, or `external-communication` domain capability.

A Context Graph MAY store run-owned source snapshots or outputs as harness artifacts. That does not authorize updating the source system or shared authoritative memory.

**Rule of thumb:** Context Graphs **determine or return information**.

### Nodes

A node is an implementation unit inside a graph, not a fourth graph category. A node may be deterministic, model-driven, a tool adapter, or a subgraph invocation boundary.

Every node inherits the restrictions of its containing graph.

## 5. Contracts

Every reusable graph and tool MUST define:

- input schema,
- output schema,
- semantics,
- failure behavior,
- relevant version information.

Graphs MUST also define their internal state schema, but internal state is not automatically part of the public contract.

Inputs are validated before use. Outputs are validated before crossing public boundaries. This applies to model-generated arguments, structured model outputs, retrieved records, and reconstructed persisted data.

Public contracts MUST distinguish:

- success,
- valid empty result,
- failure,
- partial completion where applicable,
- uncertain effect outcome where applicable.

Public cross-stack contracts SHOULD use transport-neutral data types and explicitly document timestamps, units, identifier semantics, null/absence behavior, ordering, precision, and versioning.

## 6. State and artifacts

### State ownership

The Orchestrator owns workflow-level state. Each Task and Context Graph owns its private internal state.

Parents MUST provide explicit child inputs and consume explicit child outputs. They MUST NOT rely on child-private message histories, scratch data, retry counters, or intermediate candidates.

State SHOULD contain only execution information needed for continuation: progress, bounded working data, selected inputs, results, identifiers, and artifact references.

### Artifacts

An artifact is durable data with an identity and lifecycle outside graph state. Examples include reports, source snapshots, evidence bundles, generated code, test results, and large datasets.

Size is not the definition. Even small outputs may deserve independent identity, provenance, retention, and access control.

**State tracks execution. Output is a return value. An artifact is a durable product or item of evidence.**

Artifact references MUST identify an unambiguous artifact/version and MUST NOT contain durable credentials or access tokens.

Creating run-owned artifacts is a harness operation. Publishing an artifact into a business repository, sending it to a recipient, or replacing a domain document is a Task Graph domain action through a tool.

## 7. Tools and effects

A Tool provides an external or specialized capability to graph nodes. It may be invoked by deterministic code or selected by a model; both paths use the same validation and authorization.

Every tool MUST declare:

- identity and purpose,
- input/output contract,
- effect profile,
- resource scope,
- retry/idempotency behavior when effectful.

### ToolEffect taxonomy

| Effect | Meaning | Example |
| --- | --- | --- |
| `pure` | Deterministic computation with no domain I/O. | Parse supplied bytes. |
| `read` | Reads domain information without requested mutation. | Search documents. |
| `write` | Creates or updates domain resources. | Modify a file. |
| `external-communication` | Sends/publishes content to a recipient or audience. | Send email. |
| `destructive` | Deletes or irreversibly/restrictively changes a domain resource. | Delete a resource. |

`pure` is exclusive. A destructive mutation also declares `write`. Operations with multiple effects declare all applicable effects.

### Allowed capabilities

| Caller | Allowed domain capabilities |
| --- | --- |
| Orchestrator | No direct domain-tool invocation. |
| Context Graph | `pure`, `read`. |
| Task Graph | Explicitly authorized effects/resources required by its contract. |
| Functional Core | None. |

Graph assembly SHOULD reject incompatible tool bindings. The invocation boundary MUST also check authority, effect, resource scope, and required approval immediately before execution.

Unknown effects, undeclared tools, and missing authorization fail closed.

## 8. Functional Core

The Functional Core contains pure deterministic functions such as parsers, validators, formatters, calculators, transformations, and algorithms.

For the same explicit inputs, a core function MUST produce the same observable result or declared error.

A Functional Core function MUST NOT:

- read files,
- call networks or databases,
- invoke models,
- read environment variables,
- consult the current clock implicitly,
- depend on unseeded randomness,
- mutate shared state.

Values such as time, locale, configuration, or deterministic seeds are passed explicitly when needed.

Graphs and tools MAY call the Functional Core directly.

## 9. Configuration

Configuration MAY include:

- model configuration,
- model provider configuration,
- graph configuration,
- tool configuration,
- runtime configuration,
- observation configuration.

Models and model providers remain configuration concerns. Configured model instances may be created during graph/node assembly or controlled lazy initialization.

A repository MAY use helpers/factories for model construction. That does not imply a required Model Gateway or Model Port.

Configuration MUST NOT widen architectural permissions. For example, configuration cannot make a Context Graph mutating or relabel a write tool as read-only.

## 10. Runtime contract

The harness architecture does not prescribe LangGraph, Temporal, DBOS, Restate, Mastra, XState, or a custom runtime. A runtime is conforming when it preserves this specification and explicitly declares its supported execution capabilities.

### Required baseline runtime capabilities

A conforming implementation MUST support the behavior needed by its workflows, including:

- graph invocation,
- subgraph invocation,
- validated inputs/outputs,
- private graph state,
- branching,
- iteration where required,
- fan-out/fan-in where required,
- failure propagation,
- execution limits,
- explicit termination.

The runtime MUST preserve graph capability restrictions regardless of whether an operation is model-driven or deterministic.

### Optional runtime capabilities

A deployment MAY additionally provide:

- persistent checkpoint/recovery,
- long-running pause/resume,
- durable timers,
- cancellation propagation,
- distributed scheduling,
- multi-worker recovery,
- streaming,
- replay/fork support.

These capabilities MUST be declared and tested. A framework's marketing claims or API availability are not sufficient evidence that a deployment provides them correctly.

### Replay and external effects

An external action may commit while the local worker crashes before recording completion. Therefore, checkpointing alone is never treated as exactly-once execution.

Effectful operations require one or more of:

- provider-supported idempotency keys,
- transactional deduplication,
- durable operation records,
- reconciliation before retry,
- explicit `unknown` outcomes when safe determination is impossible.

A timeout does not prove a remote mutation failed.

## 11. Conformance levels

Conformance levels describe a **deployment**, not a framework in the abstract.

### L1 — Basic

Suitable for short-lived workflows, local execution, CLIs, development, and non-durable services.

Provides the architectural boundaries, contracts, state ownership, tool enforcement, limits, and execution semantics required by the workflow, but makes no process-loss recovery guarantee.

### L2 — Recoverable

Adds verified persistent recovery capabilities such as stable execution identity, durable progress, restart/resume behavior, and tested handling of effectful work across recovery boundaries.

### L3 — Durable Distributed

Adds distributed worker execution, durable scheduling, worker failure recovery, concurrency/ownership control, long-running pause/resume, and version-aware recovery behavior.

A deployment MUST NOT claim a level without fault-injection tests that demonstrate the required behavior in the actual configured topology.

## 12. Cross-cutting requirements

### Security

Retrieved content, model output, tool responses, and artifact payloads are untrusted data. They MUST NOT redefine permissions, create executable configuration, or authorize capabilities.

Secrets MUST remain outside graph state, prompts, normal tool results, and unrestricted logs.

### Limits

Every workflow MUST define termination conditions and finite applicable limits such as iterations, model/tool calls, retries, wall-clock duration, concurrency, and data volume.

Delegation MUST preserve overall limits rather than grant every child a fresh unlimited budget.

### Observability

Execution records SHOULD correlate execution identity, graph/node identity, model/provider identity, tool calls, outcomes, and artifact references.

Privileged domain actions require audit evidence of authorization, affected resource, operation identity, and known result.

Observability MUST NOT make private graph state, secrets, or unrestricted prompts available to ordinary callers.

### Memory and caching

Caching does not bypass authorization. A cached result must retain appropriate scope, provenance, version, and freshness semantics.

Updating shared authoritative memory or a knowledge base is a domain write unless explicitly defined as narrow harness bookkeeping.

## 13. Testing

Tests MUST cover architecture boundaries affected by a change.

At minimum, implementations SHOULD test:

- invalid graph/tool input,
- invalid model-produced structured output,
- Context Graph write attempts,
- Orchestrator direct-domain-tool attempts,
- child-state encapsulation,
- empty result versus failure,
- artifact commit/reference ordering,
- unauthorized artifact resolution,
- cancellation and exhausted budgets,
- external commit followed by lost completion record,
- retry/idempotency behavior,
- private state exposure through observation APIs.

Recoverable/distributed deployments additionally test:

- process death after a committed execution boundary,
- process death while waiting for approval,
- stale/duplicate resume messages,
- worker lease/ownership races,
- version changes while executions are persisted.

Tests MUST identify the dependency/build versions and deployment configuration used.

## 14. Runtime profiles

A runtime profile maps this architecture onto a concrete stack and records:

- language and framework,
- graph/subgraph mapping,
- state and persistence mapping,
- model construction pattern,
- tool integration and enforcement,
- approval/resume behavior,
- cancellation behavior,
- observability integration,
- supported conformance level(s),
- version policy and known limitations.

Profiles may impose narrower implementation rules but MUST NOT relax the core architecture.

Available profile:

- [TypeScript / LangGraph](runtime-profiles/LANGGRAPH_TYPESCRIPT.md)

A repository adopting a profile SHOULD record the exact dependency versions, persistence backend, runtime capabilities, verified conformance level, and fault-test evidence.

---

## Glossary

**Artifact** — durable data/product/evidence with independent identity and lifecycle.  
**Context Graph** — read-only information-producing graph category.  
**Domain mutation** — change to a workflow's target/business resource.  
**Execution context** — trusted invocation metadata and scoped dependencies.  
**Functional Core** — pure deterministic computation.  
**Graph** — independently invocable logical workflow component with a contract and capability boundary.  
**Graph state** — graph-owned execution information.  
**Harness operation** — narrow runtime bookkeeping such as checkpointing, run-owned artifact storage, or telemetry.  
**Orchestrator Graph** — workflow coordination and state ownership across subgraphs.  
**Runtime profile** — concrete mapping of the core architecture to an execution stack.  
**Task Graph** — bounded unit of work that may perform authorized domain effects through tools.  
**Tool** — contracted capability with declared effects and scoped authorization.
