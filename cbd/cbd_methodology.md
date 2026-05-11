# Component-Based Development Methodology

**Authored by Sir John Nueva — Senior Architect · CBD Methodology Designer**
**Version 2.0 | 2026 | Classification: Professional Reference**

---

> *"If a component cannot be described in one sentence, it is two components.
> If its failure crashes the system, it is incorrectly implemented.
> If its internals must be known by a caller, the contract was never defined."*
>
> — Sir John Nueva

---

## Table of Contents

1. [Philosophy & Core Pillars](#1-philosophy--core-pillars)
2. [Component Contract Standards](#2-component-contract-standards)
3. [Phase I — The Blueprint](#3-phase-i--the-blueprint)
4. [Phase II — Implementation](#4-phase-ii--implementation)
5. [Worked Example: Support Ticket AI Router](#5-worked-example-support-ticket-ai-router)
6. [Stability & Fault Tolerance](#6-stability--fault-tolerance)
7. [Dual-Stream Logging Architecture](#7-dual-stream-logging-architecture)
8. [Deployment & DevOps Standards](#8-deployment--devops-standards)
9. [Quick Reference](#9-quick-reference)

---

## 1. Philosophy & Core Pillars

### 1.1 What Is Component-Based Development?

Component-Based Development (CBD) is a software engineering methodology that constructs systems from small, independently verifiable **atomic units of logic** — called components — each with a precisely defined input/output contract. No component performs more than one function. No component communicates directly with another. All data flows through explicit **Adaptors** that translate schemas at the boundaries.

CBD is not a framework or a library. It is a **discipline**: a set of non-negotiable structural rules applied consistently from design through deployment. Its value compounds over time — every component written correctly is also automatically testable, replaceable, and reusable without modification.

> **⚖ The Central Law of CBD**
>
> If a component cannot be described in one sentence, it is two components.
> If its failure crashes the system, it is incorrectly implemented.
> If its internals must be known by a caller, the contract was never defined.

---

### 1.2 The Five Core Pillars

| Pillar | Definition |
|---|---|
| **Atomicity** | Every component performs exactly one logical function. Single responsibility is enforced at design time, not as a code review suggestion. |
| **Encapsulation** | Internal logic is invisible. A component is defined exclusively by its IN-Schema, OUT-Schema, and Error-Schema — the Black Box contract. |
| **Interface-First** | Contracts are defined before implementation begins. You cannot write code for a component whose schemas have not been approved. |
| **Adaptor Isolation** | Components never call each other. All inter-component data flows through Adaptors that perform schema translation only — no logic, no I/O. |
| **Failure as Feature** | Error handling is designed in Phase I. Every component has a typed Error-Schema, a circuit breaker, and a `reset()` method. |

---

### 1.3 The Architect's Mindset

- **Structure before syntax.** A system with a clear blueprint and poor code is fixable. A system with no blueprint and elegant code is a maintenance trap.
- **Contracts before code.** The IN/OUT schema is the API. It is agreed upon, approved, and frozen before the first line of implementation is written.
- **Failure is designed, not caught.** The Error-Schema for every component is specified alongside its OUT-Schema — not added after the first bug report.
- **Reuse is designed, not discovered.** Foundation components are identified in Phase I and built first. They are never retrofitted.
- **The orchestrator sequences, never acts.** An Orchestrator calls Domain components in order. It contains zero business logic.

---

## 2. Component Contract Standards

### 2.1 The Six Non-Negotiable Elements

Every component in a CBD system is defined by exactly six elements, finalized in Phase I before any code is written.

| # | Element | Definition |
|---|---|---|
| 1 | **Identity** | Unique name (noun or verb-noun: `InvoiceValidator`) plus a single-sentence Reason for Existence. |
| 2 | **IN-Schema** | The precise typed data structure required for activation. Example: `{userId: string, amount: float}` |
| 3 | **OUT-Schema** | The precise typed data structure guaranteed to be produced. Never "returns object". Every field named and typed. |
| 4 | **Error-Schema** | `{code: string, message: string, component: string, timestamp: ISO8601}`. Returned — never raised. |
| 5 | **Trace Log** | Metadata emitted to `llm_interaction.log`: input preview, intermediate steps, output preview. |
| 6 | **Failure Map** | Every logic block requiring try-catch protection. Designed in Phase I, implemented in Phase II. |

---

### 2.2 The Four Layers

All components belong to exactly one layer. The layer determines build order, dependency direction, and allowed behaviours.

```
┌──────────────────────────────────────────────────────────────────┐
│  ORCHESTRATOR — Coordinates flow. Owns zero business logic.      │
├──────────────────────────────────────────────────────────────────┤
│  DOMAIN — Pure business rules. One atomic function per component.│
├──────────────────────────────────────────────────────────────────┤
│  ADAPTOR — Schema mapping ONLY. No I/O, no logic, no API calls.  │
├──────────────────────────────────────────────────────────────────┤
│  FOUNDATION — Logger, ErrorSchema, ConfigLoader. Built first.    │
└──────────────────────────────────────────────────────────────────┘
         Build order: Foundation → Adaptor → Domain → Orchestrator
```

| Layer | Key Rules |
|---|---|
| **Foundation** | No dependencies on other layers. Logger and ErrorSchema exist in every project. |
| **Adaptor** | Named `Source_To_Target_Adaptor`. Performs schema mapping ONLY — no I/O, no API calls, no business rules. |
| **Domain** | Contains all business logic. One atomic function per component. Circuit breaker for MEDIUM+ risk. |
| **Orchestrator** | Sequences calls only — owns zero logic. Returns a terminal OUT-Schema or Error-Schema. |

---

## 3. Phase I — The Blueprint

> ⛔ **Phase I Hard Constraint:** NO APPLICATION CODE is permitted in Phase I. Zero. Not even a helper function. Any code written before the blueprint is approved is written against an unapproved contract and will be rewritten.

### 3.1 Requirement Extraction

Before drawing any component, extract answers to six questions. Ask **one at a time** if unclear.

| Question | Purpose |
|---|---|
| What is the system's single top-level responsibility? | Defines the root boundary of the component graph |
| What are the primary data inputs and final outputs? | Anchors the IN-Schema of the entry point and OUT-Schema of the exit point |
| What external systems or APIs must it integrate with? | Forces Adaptor planning upfront — every external system needs an Adaptor |
| What are the expected failure modes? | Seeds the Error-Schema design for every component touching that failure path |
| What must be reusable vs. single-use? | Identifies shared Foundation components early, before duplication occurs |
| What are the performance or scale constraints? | Flags HIGH and CRITICAL risk components before implementation begins |

---

### 3.2 The Master Blueprint Table

The Blueprint Table is the **System Source of Truth**. Once approved, it is frozen. All Phase II implementations must match it exactly.

| Column | Definition |
|---|---|
| **#** | Sequence number. Determines build order. |
| **Component Name** | Unique identifier. Noun or verb-noun. PascalCase. |
| **Layer** | Foundation \| Adaptor \| Domain \| Orchestrator |
| **Reason for Existence** | One sentence. No "and". If "and" appears, split the component. |
| **IN-Schema** | Fully typed field map: `{field: type}` |
| **OUT-Schema** | Fully typed field map: `{field: type}`. Never "returns object". |
| **Error-Schema** | Always: `{code, message, component, timestamp: ISO8601}` |
| **Adaptor Needed** | Name of required Adaptor or "None" |
| **Trace Points** | Fields emitted to `llm_interaction.log` on each invocation |
| **Risk Level** | LOW \| MEDIUM \| HIGH \| CRITICAL |

---

### 3.3 Risk Level Definitions

| Level | Conditions | Required Mechanisms |
|---|---|---|
| **LOW** | Deterministic, internal only, no external deps | try-catch only |
| **MEDIUM** | File I/O, internal services, user-supplied data | Circuit breaker |
| **HIGH** | External APIs, variable latency or availability | Circuit breaker + retry + alerting |
| **CRITICAL** | LLM calls, payments, no-SLA external systems | Circuit breaker + fallback + monitoring |

---

### 3.4 The Mandatory Gate

> **✋ Phase I Gate — Exact Wording Required:**
>
> *"This is the proposed architecture. The blueprint is now the System Source of Truth. Which specific component should I implement first?"*

---

## 4. Phase II — Implementation

### 4.1 Blueprint Restatement (Step 0 — Mandatory)

Before writing any code, restate verbatim from the blueprint. If the restatement differs — stop and resolve.

```
Component:    [Name]
Layer:        [Foundation | Adaptor | Domain | Orchestrator]
IN-Schema:    {field: type, ...}
OUT-Schema:   {field: type, ...}
Error-Schema: {code: str, message: str, component: str, timestamp: ISO8601}
Trace Points: [fields logged to llm_interaction.log]
Risk Level:   [LOW | MEDIUM | HIGH | CRITICAL]
```

---

### 4.2 Implementation Order (Inside Every Component)

| Step | Action |
|---|---|
| 1 | Docstring: Component name, Layer, IN/OUT/Error schemas, Reason for Existence |
| 2 | Imports — constructor injection only, no module-level singletons |
| 3 | Class/struct definition with constructor-injected dependencies |
| 4 | IN-Schema guard — validates ALL required fields before any logic runs |
| 5 | Primary method (`.execute()` / `.invoke()` / `.run()`) with try-catch on every operation |
| 6 | Circuit breaker (`_failure_count`, `_circuit_open`, `CIRCUIT_THRESHOLD = 3`) |
| 7 | `_handle_error()` private method |
| 8 | `reset()` and `isAlive()` public methods |

---

### 4.3 Test Requirements (Generated Immediately — Never Deferred)

| Test | Assertion |
|---|---|
| `test_happy_path()` | Valid IN-Schema → correct OUT-Schema fields present and typed |
| `test_missing_fields()` | Partial IN-Schema → Error-Schema returned (never raised) |
| `test_wrong_types()` | Type mismatch → Error-Schema returned |
| `test_never_raises()` | Forced internal exception → Error-Schema returned, component never raises |
| `test_circuit_opens()` | N consecutive failures → circuit opens (MEDIUM+ risk only) |
| `test_reset()` | `reset()` closes circuit, clears failure counter |
| `test_is_alive()` | `isAlive()` returns correct structure with all required fields |

---

### 4.4 The Implementation Gate

> **✋ Implementation Gate — Exact Wording Required:**
>
> *"[ComponentName] is implemented with tests. OUT-Schema matches blueprint exactly. Ready to implement [NextComponentName] or shall I generate the deployment artifacts first?"*

---

## 5. Worked Example: Support Ticket AI Router

A customer submits a support ticket via HTTP. The system uses an LLM to categorise it as *billing*, *technical*, or *general*, then dispatches it to the correct team queue.

### 5.1 Requirement Extraction

| Question | Answer |
|---|---|
| Top-level responsibility? | Receive ticket → categorise with AI → route to team queue |
| Primary inputs / outputs? | IN: HTTP POST body. OUT: `{ticketId, status: routed\|failed}` |
| External integrations? | LLM API (DeepSeek) + team queue REST endpoint |
| Expected failure modes? | LLM timeout, LLM hallucinated category, queue unavailable |
| Reusable components? | Logger, ErrorSchema, ConfigLoader reused across all systems |
| Scale constraints? | Peak 500 tickets/min — queue dispatcher must handle backpressure |

---

### 5.2 Master Blueprint Table

| # | Component | Layer | IN-Schema (key fields) | OUT-Schema (key fields) | Risk |
|---|---|---|---|---|---|
| 1 | `Logger` | Foundation | `(level, component, message)` | `(logged: bool)` | LOW |
| 2 | `ErrorSchema` | Foundation | `(code, message, component)` | `(code, msg, component, timestamp)` | LOW |
| 3 | `ConfigLoader` | Foundation | `(env vars)` | `(apiKey, queueUrl, maxRetries)` | LOW |
| 4 | `TicketIngestion_Adaptor` | Adaptor | `(httpBody, headers)` | `(ticketId, userText, userId)` | MED |
| 5 | `TicketProcessor` | Domain | `(ticketId, userText, userId)` | `(ticketId, cleanText, language)` | LOW |
| 6 | `LLMCategorizer_Adaptor` | Adaptor | `(ticketId, cleanText)` | `(prompt, systemMessage, ticketId)` | MED |
| 7 | `LLMCategorizer` | Domain | `(prompt, systemMessage, ticketId)` | `(ticketId, category, confidence)` | **CRIT** |
| 8 | `QueueRouter_Adaptor` | Adaptor | `(ticketId, category)` | `(ticketId, queueUrl, priority)` | LOW |
| 9 | `QueueDispatcher` | Domain | `(ticketId, queueUrl, priority)` | `(ticketId, dispatched, queueAck)` | HIGH |
| 10 | `TicketOrchestrator` | Orchestrator | `(httpRequest)` | `(ticketId, status, latency_ms)` | MED |

---

### 5.3 Technical Risk Assessment

**Critical Bottlenecks**
- `LLMCategorizer` (CRITICAL): Non-deterministic output may return a category outside the enum. Enforce strict output parsing with fallback to `general`. Circuit breaker: 3 failures → open + alert.
- `QueueDispatcher` (HIGH): External queue may be slow or unavailable. Exponential backoff required; max retries from ConfigLoader.

**Integration Friction**
- `LLMCategorizer_Adaptor` → `LLMCategorizer`: Long tickets may exceed token limits. Adaptor must truncate `cleanText` to 2,000 chars with a trace warning.
- `QueueRouter_Adaptor`: The category→URL mapping must be config-driven, not hard-coded in the Adaptor.

> **✅ Feasibility Verdict: GO** — All dependencies are standard HTTP + LLM API calls. Highest risk is LLM non-determinism, fully mitigated by enum enforcement and a general fallback.

---

### 5.4 Stability Card — LLMCategorizer

| Property | Value |
|---|---|
| **Isolation Level** | try-catch boundary + 10s request timeout |
| **Circuit Breaker Trigger** | 3 consecutive exceptions within any 60s window |
| **Self-Healing** | `reset()` closes circuit. Default safe output: `category="general", confidence=0.0` |
| **Log Point Detail** | `event, ticketId, prompt_tokens, raw_reply[:120], category_extracted, confidence` |

---

### 5.5 Phase II — LLMCategorizer Blueprint Restatement

```
Component:    LLMCategorizer
Layer:        Domain
IN-Schema:    {prompt: str, systemMessage: str, ticketId: uuid}
OUT-Schema:   {ticketId: uuid, category: enum[billing,technical,general],
               confidence: float}
Error-Schema: {code: str, message: str, component: str, timestamp: ISO8601}
Trace Points: event, ticketId, prompt_tokens, raw_reply, category, confidence
Risk Level:   CRITICAL
```

---

### 5.6 Implementation

```python
class LLMCategorizer:
    COMPONENT         = "LLMCategorizer"
    CIRCUIT_THRESHOLD = 3
    VALID_CATEGORIES  = {"billing", "technical", "general"}

    def __init__(self, llm_client, config: dict) -> None:
        self._client        = llm_client   # constructor injection
        self._config        = config
        self._failure_count = 0
        self._circuit_open  = False

    def execute(self, payload: dict) -> dict:
        if self._circuit_open:
            log_system("ERROR", self.COMPONENT, "Circuit OPEN")
            return make_error("CIRCUIT_OPEN", "Categorizer disabled", self.COMPONENT)

        # IN-Schema guard
        required = ["prompt", "systemMessage", "ticketId"]
        missing  = [f for f in required if not payload.get(f)]
        if missing:
            return make_error("INVALID_IN_SCHEMA",
                              f"Missing: {missing}", self.COMPONENT)

        try:
            log_trace("PHASE_II", self.COMPONENT, {
                "event":        "categorize_start",
                "ticketId":     payload["ticketId"],
                "prompt_chars": len(payload["prompt"]),
            })

            response = self._client.call({
                **self._config,
                "system_prompt": payload["systemMessage"],
                "messages": [{"role": "user", "content": payload["prompt"]}],
            })
            if is_error(response):
                return self._handle_error("LLM_CALL_FAIL", response["message"])

            # Parse and validate — enforce enum, never trust raw LLM output
            raw  = response["reply"].strip().lower()
            cat  = raw if raw in self.VALID_CATEGORIES else "general"
            conf = 0.95 if raw in self.VALID_CATEGORIES else 0.0

            result = {
                "ticketId":   payload["ticketId"],
                "category":   cat,
                "confidence": conf,
            }

            log_trace("PHASE_II", self.COMPONENT, {
                "event":     "categorize_complete",
                "ticketId":  payload["ticketId"],
                "raw_reply": raw[:120],
                "category":  cat,
                "confidence": conf,
            })

            self._failure_count = 0
            return result

        except Exception as exc:
            return self._handle_error("CATEGORIZE_EXCEPTION", str(exc))

    def is_alive(self) -> dict:
        return {
            "component":     self.COMPONENT,
            "status":        "degraded" if self._circuit_open else "ok",
            "circuit_open":  self._circuit_open,
            "failure_count": self._failure_count,
        }

    def reset(self) -> None:
        self._failure_count = 0
        self._circuit_open  = False
        log_system("INFO", self.COMPONENT, "Circuit reset")

    def _handle_error(self, code: str, message: str) -> dict:
        self._failure_count += 1
        log_system("ERROR", self.COMPONENT,
                   f"[{code}] {message} (#{self._failure_count})")
        if self._failure_count >= self.CIRCUIT_THRESHOLD:
            self._circuit_open = True
            log_system("CRITICAL", self.COMPONENT, "Circuit OPENED")
        return make_error(code, message, self.COMPONENT)
```

---

### 5.7 Wire Diagram

```
[HTTP Request]
      |
      v
TicketOrchestrator.run(httpRequest)
      |
      |--1--> TicketIngestion_Adaptor.adapt(httpBody)
      |              OUT: {ticketId, userText, userId}
      |
      |--2--> TicketProcessor.execute({ticketId, userText, userId})
      |              OUT: {ticketId, cleanText, language}
      |
      |--3--> LLMCategorizer_Adaptor.adapt({ticketId, cleanText})
      |              OUT: {prompt, systemMessage, ticketId}
      |
      |--4--> LLMCategorizer.execute({prompt, systemMessage, ticketId})
      |              OUT: {ticketId, category, confidence}
      |
      |--5--> QueueRouter_Adaptor.adapt({ticketId, category})
      |              OUT: {ticketId, queueUrl, priority}
      |
      |--6--> QueueDispatcher.execute({ticketId, queueUrl, priority})
                     OUT: {ticketId, dispatched, queueAck}
      |
      v
      {ticketId, status: "routed", latency_ms: 234}
```

---

## 6. Stability & Fault Tolerance

### 6.1 The Five Stability Mechanisms

| Mechanism | Definition |
|---|---|
| **Graceful Failure** | All exceptions caught and translated to Error-Schema. Global crashes forbidden. A component that raises an uncaught exception has not been implemented correctly. |
| **Circuit Breaker** | After `CIRCUIT_THRESHOLD` (default 3) consecutive failures, the component trips to OPEN state and returns Error-Schema immediately without executing logic. |
| **IN-Schema Guard** | The first operation of every primary method validates all required fields exist and are of the correct type. Logic does not begin if the guard fails. |
| **Self-Healing** | Every component exposes `reset()` to close the circuit and clear the failure counter. Default Safe Output returned when circuit is open. |
| **Health Protocol** | Every component implements `isAlive()` returning `{component, status, circuit_open, failure_count}`. HTTP services expose `GET /health` as JSON. |

---

### 6.2 Circuit Breaker State Machine

```
CLOSED (normal)
  Component executes normally. Failure count increments on each exception.
  |
  | failure_count >= CIRCUIT_THRESHOLD
  v
OPEN (degraded)
  Returns Error-Schema immediately. No execution. Count does not increment.
  |
  | reset() called
  v
CLOSED (recovered)
  failure_count = 0. Circuit breaker count = 0. Next call executes normally.
```

---

### 6.3 Anti-Pattern Registry

> **⚠ Hard Stop** — Block the implementation gate if any of these are present.

| Anti-Pattern | Correct Action |
|---|---|
| `except Exception: pass` | Error-Schema return + system log entry. Silent failure is worse than visible failure. |
| Dependency instantiated inside class | Move to constructor injection. Enables testing without environment contamination. |
| `print()` used for logging | Replace with dual-stream logger. `print()` is not structured, persistent, or queryable. |
| `return None` on failure | Replace with `make_error()`. Callers cannot distinguish `None` from a missing result. |
| Logic inside an Adaptor | Extract to Domain component. Adaptors map fields only. |
| `os.environ["KEY"]` inline | Move to ConfigLoader. Inject config dict via constructor. |
| Test file missing at gate | Block gate — tests are not optional. |
| Orchestrator with business logic | Extract to Domain component. Orchestrators sequence — they do not decide. |

---

## 7. Dual-Stream Logging Architecture

### 7.1 The Two Sinks

| Sink | File | Content |
|---|---|---|
| **System Sink** | `logs/system.log` | Technical telemetry. Errors, stack traces, circuit state changes, startup events. Queryable by component and level. |
| **Trace Sink** | `logs/llm_interaction.log` | AI audit trail. Every LLM prompt, every response, every intermediate transformation. Must enable reconstruction of any AI decision without reading source code. |

---

### 7.2 Log Entry Formats

**System Sink Entry:**
```json
{
  "timestamp": "2026-05-10T08:12:34+00:00",
  "level":     "ERROR",
  "component": "LLMCategorizer",
  "message":   "[LLM_CALL_FAIL] Timeout after 10s (failure #2)",
  "meta":      {"ticketId": "abc-123"}
}
```

**Trace Sink Entry:**
```json
{
  "timestamp":   "2026-05-10T08:12:38+00:00",
  "phase":       "PHASE_II",
  "component":   "LLMCategorizer",
  "event":       "categorize_complete",
  "ticketId":    "abc-123",
  "raw_reply":   "billing",
  "category":    "billing",
  "confidence":  0.95
}
```

---

### 7.3 Wire-Check Protocol

When a component boundary fails in production, execute in order — skip nothing:

1. Read `system.log` — find the ERROR entry. It names the exact component and failure code.
2. Read `llm_interaction.log` — find the last trace before the failure. Identify the triggering input.
3. **Wire-Check** — does Component A's OUT-Schema field names and types match the Adaptor's IN-Schema? Check: names (case-sensitive), types, nullability.
4. **Isolate** — the fix goes in exactly one Black Box. Do not touch adjacent components.
5. **Fix and re-test** — run the integration test for this boundary after the fix.
6. **Schema regression** — if any field changed, re-validate the full chain root to leaf.

---

## 8. Deployment & DevOps Standards

### 8.1 Mandatory Project Layout

```
/project-root
├── src/
│   ├── foundation/        ← Logger, ErrorSchema, ConfigLoader — ALWAYS FIRST
│   ├── domain/            ← Business logic, one function per component
│   ├── adaptors/          ← Schema translation only
│   └── orchestrator/      ← Flow sequencing only, zero logic
├── tests/
│   ├── unit/              ← Per-component tests (all 7 test types)
│   └── integration/       ← Wire-check tests (A.OUT → Adaptor → B.IN)
├── config/
│   └── .env.example       ← All env vars documented, no real secrets
├── docs/
│   ├── architecture.md    ← Layer diagram + Adaptor wire diagram
│   └── solution-design.md ← Per-component internals + decision log
├── build/
│   ├── Dockerfile         ← Multi-stage, non-root user, log volume
│   └── docker-compose.yml
├── logs/                  ← Gitignored — runtime only
└── README.md              ← Setup, run, deploy, Wire-Check guide
```

---

### 8.2 Dockerfile Requirements

| Requirement | Reason |
|---|---|
| **Multi-stage build** | Separate dependency installation from runtime image. Smaller, faster final image. |
| **Minimal base image** | No general-purpose OS in production stage. Reduces attack surface. |
| **Non-root user** | Application process runs as unprivileged user. Security baseline. |
| **No secrets in image** | All secrets injected at runtime via environment variables. |
| **Log volume declared** | Logs directory is a Docker volume — persists across container restarts. |
| **HEALTHCHECK instruction** | Docker and Kubernetes use this to determine if traffic should be routed to the instance. |

---

### 8.3 Health Protocol

```python
# Every component implements:
def is_alive(self) -> dict:
    return {
        "component":     self.COMPONENT,
        "status":        "degraded" if self._circuit_open else "ok",
        "circuit_open":  self._circuit_open,
        "failure_count": self._failure_count,
    }

# HTTP services expose:
# GET /health → same structure as JSON
# Circuit open → status: "degraded" (alive, operating in safe-default mode)
```

---

### 8.4 Documentation Suite

| Document | Contents |
|---|---|
| `docs/architecture.md` | Layer diagram, component map, Adaptor wire diagram, error propagation, log routing, build order |
| `docs/solution-design.md` | Per-component: the *how* — exact algorithm, prompt template, transformation logic. Decision log. |
| `README.md` | Prerequisites, local run (copy-pasteable commands), Docker run, environment vars, Wire-Check guide |

---

## 9. Quick Reference

### 9.1 Phase I Checklist

- ☐ Extract requirements (6 questions; ask one at a time)
- ☐ Identify and name all components by layer
- ☐ Produce Master Blueprint Table (all 10 columns)
- ☐ Produce Technical Risk Assessment (Bottlenecks, Friction, Warnings, Verdict)
- ☐ Produce Stability Cards for all MEDIUM+ risk components
- ☐ Pass the Mandatory Gate — ask the exact gate question

### 9.2 Phase II Checklist (per component)

- ☐ Restate IN/OUT/Error schemas verbatim from blueprint
- ☐ Scaffold project structure (first component only)
- ☐ Implement in order: docstring → imports → class → guard → method → circuit breaker → reset → isAlive
- ☐ Implement dual-stream logging (both sinks, all required fields)
- ☐ Generate test file (all 7 test types)
- ☐ Generate deployment artifacts (Dockerfile, docker-compose, .env.example)
- ☐ Generate documentation suite (architecture, solution-design, README)
- ☐ Pass the Implementation Gate — ask the exact gate question

---

### 9.3 Naming Conventions

| Item | Convention | Example |
|---|---|---|
| **Component** | PascalCase noun or verb-noun | `InvoiceValidator`, `TicketOrchestrator` |
| **Adaptor** | `Source_To_Target_Adaptor` with underscores | `LLMResponse_To_Display_Adaptor` |
| **Primary method** | `.execute()`, `.invoke()`, or `.run()` | `component.execute(payload)` |
| **Private helpers** | Prefixed with `_` | `_validate_in()`, `_handle_error()` |
| **Log files** | Fixed names — not configurable | `logs/system.log`, `logs/llm_interaction.log` |
| **Error codes** | `SCREAMING_SNAKE_CASE` with component prefix | `INVOICE_VALIDATION_FAIL` |

---

### 9.4 The Two Gate Questions (Exact Wording)

> **✋ Phase I Gate:**
>
> *"This is the proposed architecture. The blueprint is now the System Source of Truth. Which specific component should I implement first?"*

> **✋ Phase II Gate:**
>
> *"[ComponentName] is implemented with tests. OUT-Schema matches blueprint exactly. Ready to implement [NextComponentName] or shall I generate the deployment artifacts first?"*

---

### 9.5 Feasibility Verdicts

| Verdict | Meaning |
|---|---|
| **GO** | All components feasible within defined schemas. Integration paths clear. Proceed to Phase II. |
| **CAUTION** | One or more components have identified risks. Listed mitigations must be applied. Proceed conditionally. |
| **STOP** | Fundamental design flaw: schema impossibility, missing dependency, circular dependency, or unresolvable constraint. Do not proceed. Redesign Phase I. |

---

---

*Component-Based Development Methodology — Version 2.0*
*Authored by **Sir John Nueva** — Senior Architect · CBD Methodology Designer*
*2026 — All rights reserved*
