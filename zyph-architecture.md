# Zyph Architecture: Service Workers, Master Investigation, and Observability

## Architectural Overview

Zyph should be built around a clear separation of responsibilities:

- **Workers know services.**
- **The Master knows incidents.**
- **Grafana and the existing observability stack know observability data.**
- **Zyph provides the intelligence that reasons across all of them.**

This document captures the core architectural principles and the resulting investigation model.

---

## 1. Service Context Belongs to the Worker

For a service such as `payments-service`, the Worker should maintain a working understanding of the service.

### Service Context

```text
Service Context
│
├── Identity
│   ├── service name
│   └── environment(s)
│
├── Codebase
│   ├── repository
│   ├── relevant code structure
│   ├── deployment configuration
│   └── infrastructure files
│
├── Changes
│   ├── recent commits
│   ├── deployments
│   └── configuration changes
│
├── Observability knowledge
│   ├── where metrics are
│   ├── where logs are
│   └── what queries to use
│
├── Runtime state
│   ├── healthy/unhealthy
│   ├── pod/container state
│   ├── restarts
│   └── resource behaviour
│
└── Service relationships
    ├── dependencies
    └── downstream/upstream services
```

The important distinction is that the Worker **does not need to own all of this data**.

Instead, it needs to:

1. Know where the information lives.
2. Know how to retrieve it.
3. Understand the information in the context of its service.

For example:

```text
Worker A
   │
   ├── Git → Service A code
   ├── Grafana → Service A metrics
   ├── Grafana/Loki → Service A logs
   ├── Kubernetes → Service A runtime state
   └── Git/deployment system → recent changes
```

Therefore, this should be called the Worker's **service context**, rather than its data store.

This also preserves Grafana and the existing observability stack as the authoritative storage layer.

---

## 2. The Master Owns the Investigation

The Worker should **not** own the full investigation lifecycle.

The initial model was:

```text
detect → investigate → explain
```

The preferred model is:

```text
detect → report signal → Master investigates using Workers
```

The architecture becomes:

```text
                    ┌─────────────────────┐
                    │        MASTER       │
                    │                     │
                    │    Main AI Agent    │
                    │                     │
                    │ Owns investigation  │
                    └──────────┬──────────┘
                               │
                     orchestrates investigation
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌───────────┐    ┌───────────┐    ┌───────────┐
        │ Worker A  │    │ Worker B  │    │ Worker C  │
        │ Service A │    │ Service B │    │ Service C │
        │           │    │           │    │           │
        │ AI Agent  │    │ AI Agent  │    │ AI Agent  │
        └───────────┘    └───────────┘    └───────────┘
```

### Core principle

> **The Master is the investigator. Workers are domain experts and service agents.**

Workers provide service-specific evidence and knowledge. The Master uses that information to build and test hypotheses across services.

---

## 3. Investigation Model

Consider a scenario where `checkout-service` starts failing.

### Step 1: Worker detects an abnormality

The Checkout Worker detects something unusual:

```text
Checkout Worker
      │
      ▼
"Something unusual is happening"
      │
      ▼
Signal → Master
```

The Worker does not need to determine the root cause.

### Step 2: Master starts the investigation

The Master asks the Checkout Worker:

> What changed in `checkout-service` recently?

The Worker responds:

```text
Deployment abc123 happened 12 minutes ago.
The following files changed:
...
```

### Step 3: Master requests additional evidence

The Master asks:

> What are the dominant errors?

The Worker responds:

```text
ConnectionTimeoutException increased 8x.
```

### Step 4: Master investigates related services

The Master identifies a possible dependency relationship and asks the Payment Worker:

> What is happening with `payment-service`?

The Payment Worker responds:

```text
Latency increased 5x during the same period.
Connection pool utilization reached 98%.
```

### Step 5: Master correlates the evidence

```text
Checkout
   │
   ├── deployment
   │
   ├── new downstream call
   │
   ▼
Payment
   │
   ├── latency ↑
   ├── connection pool ↑
   └── timeout ↑
```

The Master can then form a hypothesis such as:

> The new interaction introduced by the Checkout deployment is causing requests to `payment-service` to time out.

This is much closer to the intended Zyph product model.

---

## 4. Responsibilities of a Worker

A Worker is a **Service Specialist**.

Its responsibilities fall into four main areas.

### 4.1 Maintain Service Context

The Worker should understand:

- Which service it is responsible for.
- The service's identity and environments.
- Where the source code lives.
- Where its logs and metrics are available.
- Its infrastructure and runtime configuration.
- Its upstream and downstream dependencies.
- Recent changes and deployments.

### 4.2 Continuously Observe Its Service

The Worker should continuously observe relevant signals such as:

- Metrics
- Logs
- Runtime state
- Deployments
- Configuration changes
- Resource behaviour

### 4.3 Detect Signals

The Worker should detect abnormalities, but should not necessarily conclude:

> "This is the root cause."

Instead, it should report something closer to:

> "Something suspicious happened."

### 4.4 Answer Questions from the Master

This is one of the Worker's most important responsibilities.

The Master may ask:

- Did anything change recently?
- What errors increased?
- What was the state 20 minutes ago?
- Show me logs around this timestamp.
- Did latency change?
- Are pods restarting?
- What services do you depend on?
- Did this error exist before?
- What happened immediately before the signal?

The Worker should be able to answer these questions because it is the expert on its service.

---

## 5. Responsibilities of the Master

The Master owns **global reasoning and the investigation lifecycle**.

Its responsibilities include:

```text
Master
│
├── Receive signals
│
├── Decide whether investigation is necessary
│
├── Create investigation
│
├── Form hypotheses
│
├── Ask workers questions
│
├── Ask multiple workers in parallel
│
├── Correlate answers
│
├── Request additional evidence
│
├── Refine hypotheses
│
├── Determine probable root cause
│
├── Assign confidence
│
└── Produce explanation + suggested actions
```

The distinction is therefore:

> **Worker understands one service. Master understands the incident.**

This should be treated as a fundamental architectural principle.

---

## 6. Zyph Should Not Become Another Observability Database

Zyph should **not** become a second observability storage platform.

### Architectural principle

> **Zyph does not own the raw observability data. Zyph owns the intelligence that operates over the observability data.**

Conceptually:

```text
                    ZYPH
                     │
          ┌──────────┴──────────┐
          │                     │
       Master                Workers
          │                     │
          └──────────┬──────────┘
                     │
               Query / Fetch
                     │
                     ▼
        ┌─────────────────────────┐
        │ Existing Infrastructure │
        │                         │
        │ Grafana / Metrics       │
        │ Loki / Logs             │
        │ Kubernetes              │
        │ Git                     │
        │ Deployment systems      │
        └─────────────────────────┘
```

The primary interaction is:

```text
Zyph → Query / Fetch → Existing Systems
```

Zyph asks:

> "Give me the data I need."

It then reasons over that data and can discard or cache information as appropriate.

This keeps Zyph focused on **intelligence and investigation**, rather than duplicating observability infrastructure.

---

## 7. Detection Should Be Pluggable

The current alerting approach includes techniques such as:

- Stack-trace hashing
- Frequency buckets
- Rolling averages
- Standard deviation
- Z-scores
- New-hash detection
- Absolute thresholds

These are useful initial detection techniques, but **log-hash anomaly detection should not become the fundamental architecture of Zyph**.

Detection should be pluggable.

### Signal Engine

```text
                 Worker
                   │
          ┌────────┴────────┐
          │  Signal Engine  │
          └────────┬────────┘
                   │
       ┌───────────┼────────────┐
       │           │            │
       ▼           ▼            ▼
   Metric       Log          Runtime
  Anomaly      Anomaly        Anomaly
       │           │            │
       └───────────┼────────────┘
                   ▼
                 Signal
                   │
                   ▼
                 Master
```

The current log-stack-trace approach can therefore be one detector among many.

Future detectors could include:

- Metric anomalies
- Latency anomalies
- Error-rate anomalies
- Restart anomalies
- Resource anomalies
- Deployment correlation
- Dependency anomalies
- Traffic anomalies
- Statistical or ML-based detection

The key abstraction is:

```text
Detector → Signal → Master
```

rather than:

```text
Log hash → Alert
```

---

## 8. Signal ≠ Incident ≠ Investigation ≠ Root Cause

This distinction is central to Zyph's architecture.

### Signal

Something unusual happened.

```text
Payment latency increased 4x.
```

### Incident

Multiple signals indicate a meaningful service problem.

```text
Payment latency ↑
+
Checkout errors ↑
+
Timeouts ↑
```

### Investigation

The Master starts asking questions:

```text
What changed?
What dependencies changed?
What happened downstream?
What happened immediately before the incident?
```

### Root Cause Hypothesis

The Master forms a probable explanation:

```text
New checkout deployment introduced
additional synchronous calls to payment.
```

### Conclusion

```text
Probable root cause:
Checkout deployment abc123

Confidence:
87%

Evidence:
...
```

This progression is substantially more powerful than:

```text
log hash → alert
```

---

## 9. Proposed High-Level Architecture

The overall architecture can be represented as follows:

```text
                         ┌──────────────────────┐
                         │       ZYPH UI        │
                         │   Grafana + Zyph     │
                         └──────────┬───────────┘
                                    │
                              User asks
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       MASTER         │
                         │                      │
                         │    Main AI Agent     │
                         │                      │
                         │ Incident lifecycle   │
                         │ Investigation        │
                         │ Reasoning            │
                         │ Correlation           │
                         └──────────┬───────────┘
                                    │
                         Investigation tasks
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
      ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
      │   Worker A  │       │   Worker B  │       │   Worker C  │
      │  Service A  │       │  Service B  │       │  Service C  │
      │             │       │             │       │             │
      │ Service AI  │       │ Service AI  │       │ Service AI  │
      │ Context     │       │ Context     │       │ Context     │
      │ Observation │       │ Observation │       │ Observation │
      │ Detection   │       │ Detection   │       │ Detection   │
      └──────┬──────┘       └──────┬──────┘       └──────┬──────┘
             │                     │                     │
             └─────────────────────┼─────────────────────┘
                                   │
                             Fetch / Query
                                   │
             ┌─────────────────────┼─────────────────────┐
             ▼                     ▼                     ▼
        ┌─────────┐          ┌──────────┐          ┌──────────┐
        │ Grafana │          │   Git    │          │Kubernetes│
        │ / Loki  │          │          │          │          │
        │ Metrics │          │ Codebase │          │ Runtime  │
        └─────────┘          └──────────┘          └──────────┘
```

The arrows from Zyph into these systems are primarily **read/query paths**.

---

## 10. Architectural Principles

Before designing individual components, the following principles should be treated as foundational.

### Principle 1 — Service Ownership

Every service has a dedicated Worker/Service Agent responsible for continuously observing and understanding that service.

### Principle 2 — Centralized Investigation

The Master Agent owns the lifecycle of an investigation.

Workers provide service-specific evidence and reasoning to the Master rather than owning the investigation.

### Principle 3 — Observability Remains External

Grafana and the existing observability stack remain the source of truth for observability data.

Zyph does not become a second observability storage platform.

### Principle 4 — Detection Is Extensible

Workers produce signals using pluggable detection mechanisms.

Log-stack-trace statistical detection is one possible detector, not the foundation of Zyph.

---

## 11. Conceptual Definition

The architecture can be summarized in one statement:

> **Workers know services. The Master knows incidents. Grafana knows observability data. Zyph knows how to reason over all of them.**
