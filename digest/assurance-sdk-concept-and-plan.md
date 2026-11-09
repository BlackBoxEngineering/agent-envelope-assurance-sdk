# AgentEnvelope Assurance SDK: concept and plan

Status: design note; the SDK is not implemented yet.

## Purpose

`agent-envelope-assurance-sdk` is a narrow, application-neutral layer for recording and reconciling attempts to cause effects.

AgentEnvelope answers a cryptographic authority question:

> Was this actor authorised to issue this exact action within these signed bounds?

That answer is deliberately insufficient to describe the operational lifecycle. AgentEnvelope cannot, by authority verification alone, determine that:

- an authorised action was never attempted;
- execution started but failed;
- execution may have occurred after a timeout;
- an expected outcome or observation never arrived;
- the same action was attempted or executed more than once;
- several individually authorised actions form a harmful sequence;
- malformed, forged or out-of-scope attempts reached a host boundary but failed before they could become AgentEnvelope records.

The Assurance SDK fills that narrow gap. It provides a neutral handle for anything in a host system that does something, or tries to do something, to report the attempt and its observable outcome.

## Foundational rule

The unit of assurance is an **attempt**, not a business operation and not an AgentEnvelope authority record.

An attempt is an observed effort to cause an effect. It may ultimately be:

- rejected before authority verification;
- denied by AgentEnvelope verification;
- admitted but never started;
- started and completed successfully;
- started and failed;
- duplicated;
- missing an expected outcome;
- uncertain because execution may have occurred;
- reconciled later using additional evidence.

The Assurance SDK records these states without deciding what a factory, refrigerator, payment system, robot or other application ought to do.

## Architectural separation

```text
Untrusted request, tool call, event or scheduled obligation
                         |
                         v
Host boundary observes an attempt
                         |
                         v
AgentEnvelope Assurance SDK
  - correlation
  - durable lifecycle
  - reservation and idempotency
  - outcome tracking
  - deadlines and reconciliation
                         |
                         v
AgentEnvelope SDK verifies signed authority
                         |
                         v
Host application decides whether and how to execute
                         |
                         v
Assurance receives outcome and observation records
```

The responsibilities remain distinct:

| Layer | Responsibility |
|---|---|
| AgentEnvelope SDK | Derivation, signatures, immutable scope and deterministic authority verification |
| Assurance SDK | Attempt lifecycle, durable reservation, outcomes, missing expectations and reconciliation |
| Host application | Domain policy, operational admission and execution of effects |
| Optional hosted services | Storage, visibility, governance operations and shared infrastructure |
| Application adapters | Translation between domain events and generic assurance records |

Assurance can consume AgentEnvelope authority and records, but it cannot create, broaden or redefine that authority.

## Lifecycle

The intended baseline sequence is:

```text
observe attempt
-> assign correlation identifier
-> record the expected attempt
-> verify AgentEnvelope authority
-> atomically admit and reserve where required
-> host executes or rejects the action
-> record a signed or attested outcome
-> reconcile the complete lifecycle
```

Recording begins before AgentEnvelope verification. This is important because malformed commands, invalid signatures, unknown signers, expired authority, scope breakouts and replay attempts may never produce a normal AgentEnvelope action record.

For rejected input, the untrusted caller does not sign the assurance fact. A trusted boundary records a bounded digest of what it observed and the verification result it received.

## Minimal record model

The core should remain small. Initial record families are:

### 1. Observed attempt

States that a trusted boundary observed an attempt.

Minimum information:

- globally unique attempt/correlation identifier;
- record type and schema version;
- boundary identity;
- observed time;
- bounded input digest;
- optional operation and resource hints, explicitly marked as untrusted until verified;
- transport or adapter class;
- links to predecessor or parent attempts when applicable.

Raw untrusted payload retention is not required by the core and should be avoided by default.

### 2. Expected attempt

States that the host expects one operational attempt and defines the conditions required to reconcile it.

Minimum information:

- attempt identifier;
- command or intent digest;
- relevant deadline;
- expected outcome class;
- idempotency or uniqueness key;
- application-defined policy reference, if any.

### 3. Authority decision reference

Links the attempt to an AgentEnvelope verification result without changing its meaning.

It should preserve:

- the exact Action Envelope or record digest;
- the signed command digest;
- verifier identity;
- decision and reason code;
- evaluation time;
- any hosted or offline verification reference.

An authority decision is not an execution outcome.

### 4. Reservation

Represents an atomic claim on an attempt or externally enforced use allowance.

It supports:

- single-consumer admission;
- duplicate and concurrent-attempt detection;
- durable use consumption;
- restart-safe ownership;
- explicit expiry, release or reconciliation rules.

A reservation cannot broaden the underlying AgentEnvelope authority.

### 5. Execution transition

Records whether the host crossed the side-effect boundary.

At minimum, the lifecycle must distinguish:

- not started;
- starting or dispatched;
- started;
- completed;
- failed;
- outcome unknown.

The transition into `started` is significant. A timeout before it can normally be retried according to host policy; a timeout after it may require reconciliation before any retry.

### 6. Outcome

Records an executor or observer's statement about what happened.

An outcome should include:

- attempt identifier;
- command digest;
- reporter identity and role;
- outcome class;
- observation time;
- optional evidence digests or references;
- signature or AgentEnvelope-verifiable provenance.

Outcome classes should be generic, for example `succeeded`, `failed`, `denied`, `not_started` and `unknown`. Applications can attach versioned domain details without adding them to the generic SDK.

### 7. Expected observation

Declares that an observation, heartbeat, receipt or other report is expected by a deadline.

This makes a later absence assessable. A non-event has no payload or signature of its own, so the assurance claim must be precise:

> The designated observer did not receive the declared observation by the declared deadline.

It must not claim that the event did not occur anywhere.

### 8. Reconciliation finding

Compares expected and observed lifecycle records and produces findings such as:

- `MISSING`: a declared outcome or observation did not arrive by its deadline;
- `DUPLICATED`: more than one incompatible attempt or outcome exists;
- `UNCERTAIN`: execution may have started but no conclusive result is available;
- `ORPHANED`: execution evidence exists without a matching expected and admitted attempt;
- `CONFLICTING`: authoritative reporters disagree;
- `LATE`: a required record arrived after its reconciliation deadline;
- `RECONCILED`: later evidence resolved an earlier finding.

Findings are assurance evidence, not AgentEnvelope authority decisions.

## Using AgentEnvelope for provenance

The Assurance SDK should compose with `agent-envelope-sdk` rather than inventing a second identity or signing system.

Trusted host components can receive narrowly scoped AgentEnvelope authority for operations such as:

- `assurance.attempt.observe`;
- `assurance.expectation.record`;
- `assurance.admission.record`;
- `assurance.execution.start`;
- `assurance.outcome.record`;
- `assurance.finding.issue`;
- `assurance.finding.reconcile`.

Resources can bind those permissions to a service, device, workflow, attempt namespace or other host-defined boundary.

The same deterministic derivation, canonical encoding, hashing and signature primitives may be used through AgentEnvelope, with strict purpose and operation separation. An assurance observer key must never be usable to mint or impersonate execution authority.

For an invalid or malicious request, the boundary signs a statement equivalent to:

> I observed an input with this digest at this boundary and AgentEnvelope verification produced this result.

The attacker's asserted identity, operation and resources remain untrusted data.

Hosted AgentEnvelope services are optional. The same provenance model must work with sovereign offline derivation and verification.

## Portable integration model

The core must not depend on a particular transport, runtime, database or application framework.

Adapters should translate different environments into the same lifecycle records. Likely adapters include:

- HTTP/API middleware;
- MCP tool-call wrappers;
- queues and event consumers;
- agent and tool runtimes;
- scheduled jobs;
- telemetry and heartbeat monitors;
- local processes and command handlers;
- device and industrial controllers;
- human approval workflows.

Replaceable ports should cover:

- durable record storage;
- atomic reservation/locking;
- clocks and deadline scheduling;
- identifier generation;
- AgentEnvelope verification and signing;
- evidence storage;
- application policy evaluation;
- external history or evidence providers;
- publication and witnessing.

The SDK should define contracts and deterministic state transitions. It should not require one database, cloud, portal or process topology.

## Cross-attempt assurance

Some harm is visible only across multiple individually authorised commands. For example, eight valid one-hour defrost commands may collectively create an unsafe condition even though each command is independently in scope.

The core should preserve queryable, tamper-evident attempt history and support application-supplied evaluators over that history. It should not embed generic risk scores or domain rules.

This separation allows applications to define rules such as:

- no more than two equivalent operations per day;
- require an observation between two state-changing commands;
- halt when the preceding attempt is uncertain;
- escalate an unusual credential history for review.

The resulting assessment is an assurance decision. It does not revoke or invalidate otherwise valid AgentEnvelope authority.

## External evidence

MIR or another evidence provider can be integrated through an adapter. External evidence may contribute:

- inclusion and commitment proofs;
- credential or action history;
- independent observations;
- witness receipts;
- evidence-custody references.

The SDK must preserve source, time, digest, verification status and freshness for each assertion. No external provider becomes an AgentEnvelope issuer merely by supplying assurance evidence.

MIR is therefore a possible adapter, not a core dependency.

## Explicit non-goals

The Assurance SDK must not:

- hold or derive a customer's AgentEnvelope vault root as a platform service;
- mint, alter, broaden or revoke AgentEnvelope authority;
- replace AgentEnvelope verification;
- treat an unverified caller's claims as trusted facts;
- execute physical or digital side effects;
- contain factory, refrigeration, payment or other domain assumptions;
- become a general intrusion-detection product;
- require AI;
- require the AgentEnvelope hosted backend or portal;
- require MIR or any other evidence provider;
- claim absolute knowledge of events outside its instrumented boundaries;
- claim that a missing event never occurred anywhere;
- promise exactly-once physical effects from database state alone.

## Security and truthfulness requirements

1. **Instrumented-boundary honesty**
   
   The SDK can report only attempts and observations presented to an integrated boundary. It must never imply visibility into traffic blocked or diverted elsewhere.

2. **Bounded hostile input**
   
   Pre-verification recording must cap sizes, canonicalise safely, hash before retention where appropriate and avoid placing attacker-controlled text into trusted labels.

3. **Durability before effects**
   
   When an application requires durable coordination, the expected attempt and reservation must be committed before execution begins.

4. **Atomic reservation**
   
   Concurrent workers must not both acquire the same one-use attempt or allowance.

5. **Uncertain-outcome barrier**
   
   If execution may have started, retries must remain blocked until application policy and reconciliation make them safe.

6. **Independent state semantics**
   
   Authority validity, operational assurance and execution outcome must remain separate fields. One must never be inferred from another.

7. **Evidence provenance**
   
   Every trusted assertion must identify its reporter, signature or verification basis, time and schema version.

8. **Privacy by default**
   
   Prefer digests and bounded metadata. Raw payloads and domain evidence should remain under application-controlled retention policy.

## First release scope

The first release should implement the durable execution boundary sometimes described as Row 8:

1. versioned attempt identifiers and canonical record schemas;
2. durable expected-attempt creation;
3. AgentEnvelope authority-result linking;
4. atomic attempt/use reservation;
5. duplicate and concurrent-attempt rejection;
6. explicit pre-start, started, completed, failed and uncertain states;
7. signed or AgentEnvelope-verifiable outcome submission;
8. restart-safe lifecycle recovery;
9. retry barriers for uncertain outcomes;
10. deterministic reconciliation and findings;
11. an in-memory reference adapter for conformance tests;
12. one durable local adapter suitable for demonstrations.

Missing-observation schedules, cross-attempt evaluators and MIR adapters should follow the durable foundation rather than delaying it.

## Proposed implementation phases

### Phase 0: contract and threat model

- Fix terminology and state transitions.
- Define canonical record schemas and versioning.
- Define trust levels for unverified input, boundary assertions, AgentEnvelope decisions and outcome evidence.
- Define crash points and restart behaviour.
- Define storage atomicity requirements.
- Document what can and cannot be concluded from each record.

### Phase 1: core lifecycle

- Implement attempt creation and correlation.
- Add deterministic record hashing.
- Link AgentEnvelope verification results.
- Implement atomic reservation and lifecycle transitions.
- Add outcome submission and reconciliation.
- Add uncertain-outcome retry barriers.
- Produce framework-independent conformance tests.

### Phase 2: host adapters

- Implement a generic function/tool wrapper.
- Implement an MCP adapter.
- Implement an HTTP middleware adapter if justified by a real integration.
- Keep storage and signing behind replaceable interfaces.

### Phase 3: expected observations

- Add schedules, deadlines and grace periods.
- Add received, late, duplicated and missed-observation records.
- Model observer health and observer outage separately from a missing application event.

### Phase 4: history and external evidence

- Add application-supplied cross-attempt evaluators.
- Add provider-neutral external evidence interfaces.
- Add a MIR adapter only against a stable, verifiable contract.
- Preserve the difference between authority, assurance and final host admission.

## Factory validation

The Factory is a useful execution-attempt test bed because it has commands, independent roles and visible effects.

It should validate:

- malformed and unknown-signer attempts are observed before verification;
- expired, out-of-scope and invalid-signature commands are recorded as rejected attempts;
- one valid command receives one durable reservation;
- duplicate and concurrent submissions do not execute twice;
- the executed command is bound to the verified digest;
- a crash before execution is distinguishable from a crash after execution may have started;
- an uncertain outcome blocks blind retry;
- a separately signed outcome closes the attempt;
- restart recovery does not repeat a completed action;
- orphaned or conflicting outcome records produce findings.

No Factory operation names or robot semantics belong in the core package.

## Cold-chain validation

Cold-chain is a useful observation and cumulative-behaviour test bed.

It should validate:

- invalid attack attempts are retained as observed rejections even when AgentEnvelope cannot verify them;
- expected temperature readings are declared with cadence, deadline and observer identity;
- missing, late and duplicate readings produce distinct findings;
- an observer outage is not misreported as proof that a sensor event did not occur;
- engineer dispatch, arrival, repair and departure can be correlated as one recovery lifecycle;
- an authorised recovery action receives a conclusive outcome;
- repeated individually authorised defrost commands are available to an application-supplied history rule;
- uncertain recovery execution blocks unsafe automatic retry.

No temperature thresholds, engineer workflow or refrigeration policy belongs in the core package.

## Initial success criteria

The first release is credible when the same assurance core can be used by both Factory and cold-chain without domain branches in the SDK, and the tests demonstrate that it can truthfully answer:

- What attempt was observed?
- What was expected?
- What authority decision was linked to it?
- Was a unique operational reservation acquired?
- Did execution begin?
- What outcome was reported, by whom and with what evidence?
- Is any required record missing, duplicated, conflicting or uncertain?
- Is retry currently safe according to the host's explicit policy?

The concise architectural statement is:

> AgentEnvelope proves permission. The Assurance SDK preserves the truthful lifecycle of attempts and expected observations. The host application decides and performs the effect.
