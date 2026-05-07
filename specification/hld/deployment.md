---
chapterKind: deployment

environments:
  - environmentExternalId: ENV-001
    name: Local Evaluation Environment
    description: A single evaluation or developer machine running the full application stack via a container orchestration file. The backend service container and the browser client coexist on the same host. This is the only environment in scope for the PoC.

nodes:
  - nodeExternalId: NODE-0001
    name: Backend Service
    environmentExternalId: ENV-001
    nodeKind: application
    role: app-tier
    description: The single Rust process hosting both the Flight Data Simulator and the Event Stream Server, packaged as a Docker container image. Serves the static frontend bundle and the persistent server-push event endpoint to browser clients on the same host.
    concernsSurfaced: [latency, concurrency]

  - nodeExternalId: NODE-0002
    name: Browser Client
    environmentExternalId: ENV-001
    nodeKind: edge
    role: web-tier
    description: The browser on the evaluator's machine that hosts the React dashboard application. Connects to the Backend Service to receive the continuous event stream and renders the check-in monitor display for ground-handling staff and observers.
    concernsSurfaced: [latency]

modulePlacements:
  - moduleExternalId: MOD-0001
    nodeExternalId: NODE-0001
  - moduleExternalId: MOD-0002
    nodeExternalId: NODE-0001
  - moduleExternalId: MOD-0003
    nodeExternalId: NODE-0002

edges:
  - edgeExternalId: EDGE-0001
    fromNodeExternalId: NODE-0001
    toKind: node
    toNodeExternalId: NODE-0002
    protocol: HTTP event stream
    protocolClass: intra-VPC
    isEncrypted: false
    latencyBudget: "p95 < 5000ms"
    partitionFailureMode: The browser display freezes at last-received state; no error indicator is shown. The browser may attempt automatic reconnection via native retry behaviour.
    description: The persistent server-push event stream from the Backend Service to the browser. Carries continuous flight status update events for all active flights within the check-in window.
    carriesFlows: [FLOW-0002]

  - edgeExternalId: EDGE-0002
    fromNodeExternalId: NODE-0002
    toKind: actor
    toActorExternalId: ACT-0001
    protocol: UI
    protocolClass: intra-VPC
    isEncrypted: false
    latencyBudget: "p95 < 5000ms"
    partitionFailureMode: No display rendered if the browser fails to start; the evaluation session cannot proceed.
    description: The rendered check-in monitor dashboard presented to ground-handling staff (and Amadeus Technical Leadership as observers) on the evaluation machine's screen. Read-only; no user input is accepted.
    carriesFlows: []

concerns:
  - concernKind: observability
    name: Structured logging to stdout
    description: The backend service emits structured log lines to standard output, captured by the container runtime. No metrics collection, distributed tracing, or monitoring dashboards are produced. Log inspection is the sole diagnostic tool available during the 4-week build and evaluation session.
    sortOrder: 1

  - concernKind: availability
    name: Single-instance, no redundancy
    description: The PoC runs as a single container stack on one machine with no failover, load balancing, or replica. If the backend process crashes the stack must be restarted manually. This posture is acceptable for the controlled, time-boxed evaluation context where the evaluation session is short and the environment is operator-managed.
    sortOrder: 2

  - concernKind: security_perimeter
    name: Localhost-only network exposure
    description: All service ports are bound to the loopback interface of the evaluation machine. No services are exposed to external networks. No TLS is required for local container-to-browser communication. The simulated dataset contains no real personal or operational data.
    sortOrder: 3
---

The deployment is a single-environment stack on one evaluation machine. The Backend Service container hosts both the Flight Data Simulator and the Event Stream Server as a single Rust process; the Browser Client is the evaluator's browser loading the React dashboard and connecting to the server-push event endpoint. The intermodule flow between the Flight Data Simulator and the Event Stream Server (FLOW-0001) is an in-process communication within the Backend Service container and requires no network edge. FLOW-0002 crosses the container-to-browser boundary and is carried by EDGE-0001.
