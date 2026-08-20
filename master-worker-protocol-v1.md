# Zyph Domain Model

## 1. Service

### Definition
A **Service** is an application or microservice hosted on a cluster that performs a specific business or technical operation. During its operation, it produces observable signals such as logs, metrics, and potentially traces.

### Purpose
Represents the application whose behavior Zyph monitors and investigates.

### Ownership
- **System / Service Registry**
- The Worker is responsible for monitoring the Service but does **not** own the Service itself.

### Identity
- `ServiceId`

### Required Fields
- `ServiceId`
- `ServiceName`
- `Environment`
- `ServiceContextId`

### Optional Fields
- `Description`
- `Cluster`
- `Namespace`
- `Team`
- `Dependencies`

### Relationships
- A Service has a **ServiceContext**.
- A Service is monitored by a **Worker**.
- A Service can produce **Signals**.
- A Service can have multiple **Incidents** over time.
- A Service can have multiple **Deployments**.

### Lifecycle
1. The Service is onboarded into Zyph by engineers.
2. Engineers can deploy new versions of the Service during its lifetime.
3. Deployments may change the Service's runtime state and ServiceContext information such as `ImageVersion` and `DeploymentId`.

> **Note:** Service health is intentionally **not** modeled as a permanent field of the Service. Health is a dynamic observation and should be represented separately.

---

## 2. ServiceContext

### Definition
**ServiceContext** contains the information required by a Worker to understand and investigate a particular Service.

It represents the operational context that the Worker should have available before and during an Investigation.

### Purpose
Provide the Worker with the information required to monitor and investigate its assigned Service.

### Ownership
- **Service / Zyph configuration**
- The Worker maintains and uses the context for its assigned Service.

### Identity
- `ServiceContextId`

### Required Fields
- `ServiceContextId`
- `ServiceId`
- `RepoHost`
- `RepoUrl`
- **Runtime Context**

### Runtime Context
The context may also contain dynamic information such as:
- `ImageVersion`
- `DeploymentId`
- Current deployment information
- Log source
- Metrics source
- Relevant configuration
- Observability datasource information

### Optional Fields
- `RepositoryBranch`
- `Cluster`
- `Namespace`
- `DatasourceConfiguration`
- `KnownDependencies`
- `ServiceConfiguration`
- `Runbook`
- `KnownErrorPatterns`

### Relationships
- A Service has a **ServiceContext**.
- A Worker maintains a working copy/cache of the ServiceContext.
- A ServiceContext may be updated when the Service configuration or deployment changes.

### Lifecycle
1. A ServiceContext is created when a Service is onboarded.
2. Static information such as repository information generally remains stable.
3. Runtime information such as `ImageVersion` and `DeploymentId` can change when a new deployment occurs.
4. The Worker must receive or discover updated context when relevant Service configuration changes.

---

## 3. Worker

### Definition
A **Worker** is a Zyph component responsible for monitoring and investigating one Service.

The Worker maintains the operational context of the Service assigned to it and executes Tasks received from the Master.

### Purpose
- Monitor the assigned Service.
- Detect abnormal behavior.
- Generate Signals.
- Execute investigation Tasks.
- Collect Evidence.
- Report observations and results to the Master.

### Ownership
- **Master**
- The Master manages the Worker and assigns it to a Service.

### Identity
- `WorkerId`

### Required Fields
- `WorkerId`
- `ServiceContextId`
- `Status`
- `IsHealthy`

### Optional Fields
- `Capabilities`
- `LastHeartbeatTime`
- `WorkerVersion`

### Worker Status
A Worker can be in one of the following states:

| Status       | Description                          |
|--------------|--------------------------------------|
| `IDLE`       | Available for work                   |
| `BUSY`       | Currently executing a Task           |
| `UNAVAILABLE`| Not responding or unreachable        |
| `DRAINING`   | Preparing to shut down / finish work |

### Relationships
- A Worker is assigned to a **ServiceContext**.
- A Worker monitors the associated **Service**.
- A Worker receives **Tasks** from the Master.
- A Worker produces **Evidence** and **TaskResults**.
- A Worker can generate **Signals**.
- A Worker reports its status and capabilities to the Master.

### Lifecycle
1. The Master creates/registers a Worker and assigns it to a ServiceContext.
2. The Worker becomes `IDLE` when available for work.
3. When the Master assigns a Task, the Worker becomes `BUSY`.
4. After completing the Task, it can return to `IDLE`.
5. If the Worker stops responding or becomes unavailable, the Master marks it accordingly.

---

## 4. Signal

### Definition
A **Signal** is an observation or indication that the behavior of a Service may have deviated from its expected behavior.

A Signal does **not** necessarily mean that an Incident exists.

### Purpose
Notify the Master about potentially abnormal behavior that may require further investigation.

### Ownership
- **Worker**

### Identity
- `SignalId`

### Required Fields
- `SignalId`
- `ServiceId`
- `WorkerId`
- `SignalType`
- `Data`
- `Confidence`
- `CreationTime`

### Optional Fields
- `EvidenceId`
- `Severity`
- `ObservedAt`
- `DetectionRule`
- `Source`

### Examples
- Error rate increased significantly
- New stack trace detected
- Latency crossed baseline
- Pod restart frequency increased
- OOMKilled detected
- Unexpected configuration change detected

### Relationships
- A Signal is generated by a **Worker**.
- A Signal belongs to a **Service**.
- A Signal may have one or more **Evidence** objects.
- Multiple Signals may contribute to the same **Incident**.
- A Signal may trigger an **Investigation**.

### Lifecycle

Detected
->
Created
->
Sent to Master
->
Evaluated
->
Ignored / Correlated into Incident / Investigation Trigger

The Master decides whether the Signal is significant enough to contribute to an Incident or start an Investigation.

---

## 5. Incident

### Definition
An **Incident** represents an operational problem affecting a Service.

An Incident can be formed by correlating one or more Signals that appear to represent the same underlying problem.

### Purpose
Represent and track an actual or suspected operational problem affecting a Service.

### Ownership
- **Master**
- The Worker provides observations that contribute to an Incident, while the Master is responsible for correlating and managing the Incident.

### Identity
- `IncidentId`

### Required Fields
- `IncidentId`
- `ServiceId`
- `Status`
- `CreationTime`

### Optional Fields
- `Description`
- `Severity`
- `SignalIds`
- `StartTime`
- `EndTime`

### Relationships
- An Incident belongs to a **Service**.
- An Incident can contain multiple **Signals**.
- An Incident can have one or more **Investigations**.
- An Incident may be updated as new Signals arrive.

### Lifecycle

Created
->
Active
->
Investigating
->
Resolved

- An Incident may be created when the Master determines that one or more Signals represent a meaningful Service problem.
- Multiple Signals should be correlated into the same Incident when they are determined to represent the same underlying problem.

---

## 6. Investigation

### Definition
An **Investigation** represents the process of determining the cause of an Incident.

It is the primary unit of reasoning within Zyph.

### Purpose
Determine why an Incident occurred and produce a Conclusion supported by Evidence.

### Ownership
- **Master**

### Identity
- `InvestigationId`

### Required Fields
- `InvestigationId`
- `IncidentId`
- `CreationTime`
- `Status`

### Optional Fields
- `TriggerSignalId`
- `StartTime`
- `EndTime`
- `ConclusionId`

### Status
| Status        | Description                          |
|---------------|--------------------------------------|
| `CREATED`     | Investigation has been created       |
| `PLANNING`    | Master is planning Tasks             |
| `IN_PROGRESS` | Tasks are being executed             |
| `COMPLETED`   | Investigation finished successfully  |
| `FAILED`      | Investigation failed                 |
| `CANCELLED`   | Investigation was cancelled          |

### Relationships
- An Investigation belongs to an **Incident**.
- An Investigation may have been triggered by a **Signal**.
- An Investigation contains **Tasks**.
- Tasks produce **Evidence**.
- Evidence is used to evaluate **Hypotheses**.
- An Investigation can contain multiple **Hypotheses**.
- An Investigation produces a **Conclusion**.


### Lifecycle

Created
->
Planning
->
Tasks Assigned
->
Evidence Collected
->
Hypotheses Evaluated
->
Conclusion
->
Completed


- The Master creates an Investigation when it determines that an Incident requires investigation.
- The Master continuously updates the Investigation as Tasks complete, Evidence arrives, and Hypotheses are evaluated.

---

## 7. Task

### Definition
A **Task** represents a unit of work that the Master asks a Worker to perform as part of an Investigation.

### Purpose
Obtain specific information required to progress an Investigation.

### Ownership
- **Master**

### Identity
- `TaskId`

### Required Fields
- `TaskId`
- `InvestigationId`
- `WorkerId`
- `TaskType`
- `Query`
- `Status`

### Optional Fields
- `CreationTime`
- `Deadline`
- `Priority`

### Example Task Types
- `INSPECT_LOGS`
- `INSPECT_METRICS`
- `INSPECT_RECENT_CHANGES`
- `QUERY_SERVICE_STATE`
- `COLLECT_EVIDENCE`
- `VALIDATE_HYPOTHESIS`

### Relationships
- A Task belongs to an **Investigation**.
- A Task is assigned to a **Worker**.
- A Worker executes the Task.
- A Task produces a **TaskResult**.
- A Task may produce one or more **Evidence** objects.


### Lifecycle

CREATED
->
ASSIGNED
->
RUNNING
->
COMPLETED / FAILED / TIMEOUT

- The Master creates and assigns the Task.
- The Worker executes it and returns the result.

---

## 8. Evidence

### Definition
**Evidence** is an observable piece of information collected during an Investigation that can support or contradict a Hypothesis.

Evidence should always have provenance so that the Master can determine where the information came from.

### Purpose
Provide factual observations that the Master can use during reasoning.

### Ownership
- **Worker**

### Identity
- `EvidenceId`

### Required Fields
- `EvidenceId`
- `InvestigationId`
- `TaskId`
- `ServiceId`
- `Data`
- `CollectedAt`

### Optional Fields
- `SignalId`
- `Source`
- `EvidenceType`
- `Confidence`

### Example Evidence
- 5xx rate increased from 1.2% to 18.7%.
- Deployment `abc123` occurred 4 minutes before the error-rate increase.
- New stack-trace hash appeared after deployment.
- Database connection pool utilization reached 100%.

### Relationships
- Evidence belongs to an **Investigation**.
- Evidence is produced by a **Task**.
- Evidence is produced by a **Worker**.
- Evidence may be related to a **Signal**.
- Evidence may support or contradict one or more **Hypotheses**.


### Lifecycle

Collected
->
Returned to Master
->
Added to Investigation
->
Used for Hypothesis evaluation


> **Important:** Evidence should be **immutable** once recorded. If the Worker discovers new information, it should produce new Evidence rather than silently modifying existing Evidence.

---

## 9. Hypothesis

### Definition
A **Hypothesis** is a possible explanation for the problem being investigated.

An Investigation can contain multiple competing Hypotheses.

### Purpose
Represent possible causes and allow the Master to evaluate them against collected Evidence.

### Ownership
- **Investigation**
- The Master creates and manages Hypotheses as part of the Investigation.

### Identity
- `HypothesisId`

### Required Fields
- `HypothesisId`
- `InvestigationId`
- `Data`
- `Confidence`
- `Status`

### Status
| Status         | Description                              |
|----------------|------------------------------------------|
| `PROPOSED`     | Newly created hypothesis                 |
| `VALIDATING`   | Currently being validated                |
| `SUPPORTED`    | Supported by Evidence                    |
| `REJECTED`     | Contradicted / disproven                 |
| `INCONCLUSIVE` | Insufficient evidence to decide          |

### Optional Fields
- `CreatedAt`
- `UpdatedAt`
- `SupportingEvidenceIds`
- `ContradictingEvidenceIds`

### Relationships
- An Investigation can have multiple **Hypotheses**.
- A Hypothesis belongs to one **Investigation**.
- Evidence can support or contradict a Hypothesis.
- Tasks can be created specifically to validate a Hypothesis.
- A Conclusion may select one Hypothesis as the most likely root cause.


### Lifecycle

Proposed
->
Validating
->
Supported / Rejected / Inconclusive


- Hypotheses are created and updated as the Investigation progresses.
- The Master should be able to maintain multiple competing Hypotheses rather than committing to the first possible explanation.

---

## 10. Conclusion

### Definition
A **Conclusion** represents the final result of an Investigation.

It communicates what Zyph believes happened, why it happened, and how confident it is in that determination.

### Purpose
Provide the engineer with the final outcome of the Investigation and the reasoning supporting it.

### Ownership
- **Master**

### Identity
- `ConclusionId`

### Required Fields
- `ConclusionId`
- `InvestigationId`
- `ConclusionType`
- `Summary`
- `Confidence`
- `CreationTime`

### Optional Fields
- `RootCause`
- `HypothesisId`
- `SupportingEvidenceIds`
- `RecommendedActions`

### Conclusion Types
| Type                    | Description                              |
|-------------------------|------------------------------------------|
| `ROOT_CAUSE_IDENTIFIED` | Clear root cause found                   |
| `LIKELY_ROOT_CAUSE`     | Most probable root cause identified      |
| `NO_ROOT_CAUSE_FOUND`   | No root cause could be determined        |
| `INSUFFICIENT_EVIDENCE` | Not enough evidence to conclude          |
| `INVESTIGATION_FAILED`  | Investigation itself failed              |

### Relationships
- A Conclusion belongs to an **Investigation**.
- A Conclusion may reference the **Hypothesis** that led to the final determination.
- A Conclusion may reference supporting **Evidence**.
- The Master produces the Conclusion.

### Lifecycle
A Conclusion is produced when the Master determines that the Investigation has enough information to reach an outcome.

The Conclusion becomes the **final output** of the Investigation.