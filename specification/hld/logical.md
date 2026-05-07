---
chapterKind: logical

subsystems:
  - subsystemExternalId: SUB-backend
    name: Backend
    description: The Rust server process that owns the simulated flight domain, generates continuous passenger and baggage events, and serves the real-time event stream to browser clients.
    sortOrder: 1

  - subsystemExternalId: SUB-frontend
    name: Frontend
    description: The browser-resident React application that receives the live event stream and renders the check-in monitor dashboard using the IBM Carbon Design System component library.
    sortOrder: 2

modules:
  - moduleExternalId: MOD-0001
    name: Flight Data Simulator
    subsystemExternalId: SUB-backend
    tagline: Generates and maintains simulated flight state
    category: business
    description: The Flight Data Simulator seeds and maintains an in-memory pool of at least 100 active flights, each with its full set of passengers and baggage records. It continuously produces passenger check-in, seat assignment, and baggage events at a steady uniform rate, updating the in-memory state and notifying the Event Stream Server of each change. It owns all three domain entities and enforces the in-memory retention policy (flights are evicted after departure).
    technologies: ["Rust", "Tokio"]
    serves: [ACT-0001, ACT-0002]
    provides: []
    consumes: []
    dependsOn: []
    encodedFrom: [FND-0001, FND-0002, FND-0005, FND-0007]
    foundationConcernsTouched: ["Data lifecycle & retention", "Business exception handling", "Application Platform"]
    capabilities:
      - capabilityExternalId: CAP-0001
        name: flight-generation
        tagline: Bootstrap and maintain the active flight pool
        description: Initialises a pool of at least 100 simulated flights whose check-in windows are open, and continuously introduces new flights as others depart. Enforces the eviction policy — departed flights are removed from the in-memory store once their departure time passes.
        actorsServed: []
        entitiesTouched: [ENT-0001, ENT-0002, ENT-0003]
        sortOrder: 1
      - capabilityExternalId: CAP-0002
        name: event-simulation
        tagline: Produce continuous passenger and baggage events
        description: Continuously generates passenger check-in events (assigning each to either the online or counter channel), seat assignment events, and baggage registration and drop-off events at a steady, uniform rate. No peak-load bursts are produced. Each event updates the in-memory state and triggers a state-change notification to the Event Stream Server.
        actorsServed: []
        entitiesTouched: [ENT-0001, ENT-0002, ENT-0003]
        sortOrder: 2

  - moduleExternalId: MOD-0002
    name: Event Stream Server
    subsystemExternalId: SUB-backend
    tagline: Bridges simulation state to connected browser clients
    category: integration
    description: The Event Stream Server accepts persistent browser connections and pushes flight state-change events to all connected clients as they arrive from the Flight Data Simulator. It translates internal state-change notifications into serialised events and delivers them over the outbound event stream. On client disconnect the connection is closed cleanly; no event replay is provided on reconnection.
    technologies: ["Rust", "Tokio"]
    serves: [ACT-0001, ACT-0002]
    provides: []
    consumes: []
    dependsOn: [MOD-0001]
    encodedFrom: [FND-0002, FND-0003, FND-0004, FND-0005, FND-0007]
    foundationConcernsTouched: ["Async & event architecture", "Business exception handling", "Observability", "Application Platform"]
    capabilities:
      - capabilityExternalId: CAP-0003
        name: sse-connection-management
        tagline: Accept and lifecycle-manage browser connections
        description: Opens and maintains persistent server-push connections with browser clients, tracks the set of active connections, and handles client disconnects by closing the connection cleanly. No reconnection state or event backlog is maintained.
        actorsServed: [ACT-0001, ACT-0002]
        entitiesTouched: []
        sortOrder: 1
      - capabilityExternalId: CAP-0004
        name: state-broadcast
        tagline: Push flight status events to all connected browsers
        description: Receives flight state-change notifications from the Flight Data Simulator and serialises each change as a text event pushed to every currently connected browser client. Delivers events within the 5-second latency ceiling.
        actorsServed: [ACT-0001, ACT-0002]
        entitiesTouched: []
        sortOrder: 2

  - moduleExternalId: MOD-0003
    name: Check-in Monitor Dashboard
    subsystemExternalId: SUB-frontend
    tagline: Renders real-time flight check-in status
    category: presentation
    description: The Check-in Monitor Dashboard is a browser-based read-only single-page application that connects to the Event Stream Server, receives live flight status events, and renders the check-in progress for every active flight using IBM Carbon Design System components. It displays total passenger count, online and counter check-in breakdowns, bag registration and drop-off counts, and seat fill status. The display updates automatically on each incoming event without a page reload.
    technologies: ["React", "TypeScript", "IBM Carbon Design System"]
    serves: [ACT-0001, ACT-0002]
    provides: [IFC-0001, IFC-0002]
    consumes: []
    dependsOn: [MOD-0002]
    encodedFrom: [FND-0003, FND-0006, FND-0007]
    foundationConcernsTouched: ["Async & event architecture", "Front-end Framework"]
    capabilities:
      - capabilityExternalId: CAP-0005
        name: dashboard-layout
        tagline: Render the per-flight check-in status grid
        description: Displays the list of all active flights in a grid layout where each card shows the flight's total passenger count, online and counter check-in counts, bags registered and dropped, and current seat fill status. All components are drawn from the IBM Carbon Design System library.
        actorsServed: [ACT-0001, ACT-0002]
        entitiesTouched: []
        sortOrder: 1
      - capabilityExternalId: CAP-0006
        name: live-update-reception
        tagline: Receive and apply incoming status events
        description: Establishes a persistent connection to the Event Stream Server, receives server-pushed flight status events, and applies each incremental update to the in-memory display state so the rendered grid reflects the latest check-in figures without a page reload.
        actorsServed: [ACT-0001, ACT-0002]
        entitiesTouched: []
        sortOrder: 2

technicalFoundations:
  - foundationExternalId: TF-001
    area: application_platform
    name: Back-end application platform
    summary: The server-side process is implemented in Rust and runs as a single async service. Rust provides memory safety and high-throughput async I/O suited to managing many concurrent browser connections without per-connection thread overhead.
    technologies: ["Rust", "Tokio"]
    rationale: Rust is the Amadeus target systems programming language — a programme-level constraint, not a choice open to the PoC. See FND-0005 for the constraint rationale.
    sortOrder: 1

  - foundationExternalId: TF-002
    area: persistence
    name: In-memory state (no persistent store)
    summary: All flight, passenger, and baggage state is held in process memory. There is no database or durable storage. State is generated at process startup and discarded when the process exits.
    technologies: []
    rationale: The PoC is explicitly prohibited from connecting to any Amadeus production database. An in-memory dataset is the simplest arrangement that satisfies the simulation requirement and the 4-week delivery constraint. See FND-0001 for the data lifecycle and eviction policy.
    sortOrder: 2

  - foundationExternalId: TF-003
    area: other
    name: Front-end framework and design system
    summary: The browser client is built with React and the IBM Carbon Design System component library. React is Amadeus's strategic front-end framework; IBM Carbon is the specific design system under evaluation.
    technologies: ["React", "TypeScript", "IBM Carbon Design System"]
    rationale: Both React and IBM Carbon are named programme-level constraints. The entire purpose of the PoC is to generate usability evidence for IBM Carbon. See FND-0006 for the constraint rationale.
    sortOrder: 3

  - foundationExternalId: TF-004
    area: security
    name: No authentication or authorisation
    summary: No authentication or authorisation layer is applied. The application is not exposed beyond the local evaluation environment and the simulated data requires no protection.
    technologies: []
    rationale: Authentication and authorisation are explicitly out of scope per Chapter 2. Adding them would increase implementation time without contributing to the usability or velocity evaluation.
    sortOrder: 4

  - foundationExternalId: TF-005
    area: messaging
    name: Real-time event delivery
    summary: Flight status updates are pushed from the server to connected browser clients using a persistent server-push event stream. The delivery is unidirectional — server to client — and continuous; clients receive updates without issuing repeated requests.
    technologies: ["Server-Sent Events (SSE)"]
    rationale: SSE is the delivery mechanism named in the project intake notes. See FND-0003 for the full alternatives analysis.
    sortOrder: 5

  - foundationExternalId: TF-006
    area: observability
    name: Basic structured logging
    summary: Structured log output to standard output is the sole observability instrumentation. No metrics collection, distributed tracing, or monitoring dashboards are produced.
    technologies: []
    rationale: The project intake notes explicitly limit observability to basic logging. Full instrumentation would add implementation time without contributing to the evaluation. See FND-0004 for the scoping decision.
    sortOrder: 6

  - foundationExternalId: TF-007
    area: build_runtime
    name: Container-based packaging
    summary: The complete application is packaged as container images and coordinated by a single Compose file. This allows the evaluation team to start the full stack with one command on any machine with a container runtime installed.
    technologies: ["Docker", "Docker Compose"]
    rationale: Docker and Docker Compose are named in the project intake notes as the delivery format. See FND-0007 for the packaging constraint rationale.
    sortOrder: 7

foundationDecisions:
  - decisionExternalId: FND-0001
    concernName: "Data lifecycle & retention"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "The product holds all flight and passenger state in process memory. Without a retention policy, the in-memory store grows without bound as the simulator continuously generates new flights over hours of operation."
    rationale: Flights are evicted from the in-memory store once their departure time passes. This caps memory to approximately the active check-in window (100+ simultaneous flights) regardless of how long the process runs.
    optionsConsidered:
      - label: Evict on departure
        characterisation: Remove a flight and all its passenger and baggage records from memory once the departure time passes. Memory use is bounded to the active window size at all times. No historical data is available, which is acceptable since reporting is explicitly out of scope.
      - label: Retain all records for process lifetime
        characterisation: Keep every generated record in memory indefinitely. Simplifies the simulator loop but causes unbounded memory growth — a process running for hours with 100+ flights per hour would exhaust available memory in the evaluation environment.
    decision: Evict flight records (and their associated passenger and baggage records) from the in-memory store after the departure time passes. The display covers only flights currently within the check-in window.
    consequences:
      - kind: positive
        text: Memory use is bounded and predictable regardless of run duration.
      - kind: negative
        text: No historical flight data is retained; the display cannot show recently departed flights or cumulative trends.
    revisitIf:
      - "A requirement to display recently departed flights or historical check-in trends is introduced."
    encodedIn: [MOD-0001]
    verifiedByDescription: "The simulator's in-memory flight map contains no entries with departure times in the past."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0002
    concernName: "Business exception handling"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "A posture must be chosen for two failure modes: the event generator encounters an internal error and stops producing events; and an SSE client connection is interrupted mid-session. Without a declared posture, failure handling is undefined and the evaluation may be disrupted."
    rationale: Passive degradation is chosen. In a controlled local evaluation environment, interruptions are unlikely. The minimum-complexity posture is appropriate for the PoC scope.
    optionsConsidered:
      - label: Passive degradation — display stale data, no error UI
        characterisation: If the generator stalls or the stream drops, the display freezes at the last-received state. The client may reconnect automatically via native retry behaviour. No error indicator is shown. Matches the PoC's read-only, no-user-action constraint.
      - label: Active error indication — show stale-data banner after SLO breach
        characterisation: Display a visual warning when no update has arrived within the 5-second SLO window. Makes failures visible to evaluators but adds UI complexity not justified by the PoC environment.
    decision: Passive degradation. The display shows the last-received state on any interruption. No error indicator or manual retry control is in scope.
    consequences:
      - kind: positive
        text: Zero additional UI complexity; the evaluation environment is controlled and interruptions are unlikely.
      - kind: negative
        text: A frozen display is indistinguishable from a genuine pause in check-in activity; evaluators cannot tell whether data is stale.
    revisitIf:
      - "The evaluation session reveals that a frozen display causes confusion or invalidates usability feedback."
      - "The PoC is operated in an environment where network reliability cannot be controlled."
    encodedIn: [MOD-0001, MOD-0002, MOD-0003]
    verifiedByDescription: "No error payload is emitted on stream interruption; the client receives a clean connection close."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0003
    concernName: "Async & event architecture"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "Flight status updates must reach the browser display within 5 seconds of generation. A delivery mechanism must be chosen for the server-to-browser path."
    rationale: Server-Sent Events (SSE) is chosen. It is the mechanism named in the project intake notes and is the most appropriate fit for a unidirectional, continuously-streamed feed where no messages flow from client to server.
    optionsConsidered:
      - label: Server-Sent Events (SSE)
        characterisation: A persistent HTTP connection over which the server pushes text events. Unidirectional — server to client only. Native browser support with automatic retry on disconnect. Simpler server implementation than WebSockets for a one-way feed; sub-second event delivery comfortably meets the 5-second ceiling.
      - label: WebSockets
        characterisation: Full-duplex protocol suited to bidirectional communication. Higher implementation complexity; the bidirectional capability is unnecessary since no client-to-server messages are sent in this read-only product.
      - label: Short-interval HTTP polling
        characterisation: Client requests full flight state every N seconds. Simple but cannot guarantee sub-5-second latency without a polling interval shorter than 5 seconds, and repeated full-state fetches impose higher server load than event-driven delivery.
    decision: Server-Sent Events (SSE). The unidirectional constraint and the 5-second ceiling make SSE the natural fit; it avoids WebSocket overhead and guarantees near-real-time event delivery.
    consequences:
      - kind: positive
        text: Low implementation complexity; native browser reconnection; the 5-second latency ceiling is met with margin.
      - kind: negative
        text: SSE is server-to-client only; any future requirement for client-to-server messages requires a separate HTTP endpoint or a protocol change to WebSockets.
    revisitIf:
      - "A requirement for client-to-server messages is introduced (e.g., flight filtering, acknowledgement)."
      - "The evaluation environment's network stack does not support persistent HTTP connections."
    encodedIn: [MOD-0002, MOD-0003, IFC-0001, IFC-0002]
    verifiedByDescription: "The SSE endpoint returns Content-Type: text/event-stream; events arrive in the browser within 5 seconds of generation."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0004
    concernName: "Observability"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "An observability posture must be declared. The PoC must be diagnosable during the 4-week build and evaluation session without adding infrastructure that does not contribute to the evaluation."
    rationale: Structured log output to standard output is sufficient for the PoC. The project intake notes explicitly state that observability and monitoring are not in scope; only basic logging is required.
    optionsConsidered:
      - label: Structured logging to stdout only
        characterisation: Emit structured log lines at INFO and ERROR levels to standard output. Sufficient for diagnosing issues during development and the evaluation session. Zero external dependencies.
      - label: Full observability stack (metrics + traces + log aggregation)
        characterisation: Integrate a metrics library, distributed trace instrumentation, and a log aggregator. Provides accurate latency percentile data but adds several days of implementation work that does not contribute to the usability evaluation.
    decision: Structured logging to standard output only. No metrics, distributed traces, or monitoring dashboards are produced.
    consequences:
      - kind: positive
        text: Zero observability infrastructure to operate; implementation time is negligible.
      - kind: negative
        text: If the 5-second latency SLO is breached during the evaluation, diagnosing root cause requires manual log inspection with no automated percentile data.
    revisitIf:
      - "The 5-second latency SLO is breached during the evaluation session and logs are insufficient to identify the cause."
    encodedIn: [MOD-0001, MOD-0002]
    verifiedByDescription: "The server process emits structured log lines to stdout on startup and on each connection lifecycle event."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0005
    concernName: "Application Platform"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "The back-end implementation language must be selected. The choice determines the runtime model, available async I/O ecosystem, and the Amadeus team's ability to maintain the service after the PoC."
    rationale: Rust is the Amadeus target systems programming language — a programme-level constraint. The PoC adopts it as a given; the velocity measurement is intended to reflect real Amadeus delivery conditions.
    optionsConsidered:
      - label: Rust
        characterisation: The Amadeus designated target language for server-side services. The entire UI modernisation programme standardises on Rust; the PoC is consistent with that direction. The async Tokio ecosystem provides the concurrency primitives needed for concurrent SSE connections.
      - label: Go
        characterisation: An alternative systems language with simpler concurrency primitives and a large standard library. Not in scope — adopting Go for the PoC would introduce a platform inconsistency and make the velocity measurement inapplicable to Amadeus's actual delivery environment.
    decision: Rust. This is a programme-level constraint from the Amadeus UI modernisation programme, not a free choice for this PoC.
    consequences:
      - kind: positive
        text: The velocity measurement reflects real Amadeus conditions; the result directly informs the programme adoption decision.
      - kind: negative
        text: The PoC team must be proficient in Rust; the evaluation cannot fully separate framework usability from language proficiency effects on delivery speed.
    revisitIf:
      - "The Amadeus programme revises its language standardisation policy."
    encodedIn: [MOD-0001, MOD-0002]
    verifiedByDescription: "The back-end crate compiles with the stable Rust toolchain."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0006
    concernName: "Front-end Framework"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "The browser client must be built on a specific framework and design system. The PoC exists specifically to evaluate the IBM Carbon Design System; the framework is also a programme constraint."
    rationale: React is Amadeus's strategic front-end framework. IBM Carbon is the specific design system under evaluation. Both are named programme-level constraints; the entire purpose of the PoC is to generate usability evidence for Carbon on top of React.
    optionsConsidered:
      - label: React + IBM Carbon Design System
        characterisation: The Amadeus strategic framework (React) paired with the evaluation subject (IBM Carbon). The PoC is the evaluation vehicle for Carbon; no alternative design system is in scope.
      - label: React + an alternative design system
        characterisation: Would change two variables simultaneously — framework and design system — making it impossible to attribute evaluation results to Carbon specifically. Contradicts the programme objective.
    decision: React with the IBM Carbon Design System. Both are programme-level constraints.
    consequences:
      - kind: positive
        text: Usability evidence is directly attributable to IBM Carbon; no scope ambiguity in the evaluation result.
      - kind: negative
        text: The evaluation cannot separate Carbon's contribution from React's baseline UX; a mixed result cannot be cleanly attributed to either alone.
    revisitIf:
      - "The Amadeus programme changes its strategic front-end framework or selects a different design system for evaluation."
    encodedIn: [MOD-0003]
    verifiedByDescription: "The browser bundle imports the IBM Carbon Design System component package."
    verifiedByKind: grep_pattern

  - decisionExternalId: FND-0007
    concernName: "Build & Runtime Packaging"
    concernListVersion: 1
    resolution: yes_see_section
    problem: "The PoC must be runnable by the evaluation team without installing language-specific toolchains on their machines. A delivery format must be chosen."
    rationale: Docker container images with a Docker Compose file are the named delivery format in the project intake notes. A single command starts the full stack.
    optionsConsidered:
      - label: Docker images + Docker Compose
        characterisation: The complete stack is packaged as container images and started with docker compose up. Evaluators need only a container runtime installed; no Rust toolchain or Node.js is required on their machines.
      - label: Pre-built native binaries + static HTML bundle
        characterisation: Distribute compiled binaries and a built frontend bundle directly. Simpler runtime model but requires separate build artifacts per operating system and architecture, and the team must manage two separate startup processes.
    decision: Docker images with a Docker Compose file. Named explicitly in the project intake; provides the simplest possible evaluation-environment setup.
    consequences:
      - kind: positive
        text: Single-command startup; no language runtime dependencies on the evaluation machine.
      - kind: negative
        text: Docker must be available on the evaluation machine; container image builds add to the iterative development cycle.
    revisitIf:
      - "The evaluation environment does not support Docker (e.g., locked-down corporate workstation without container runtime access)."
    encodedIn: [MOD-0001, MOD-0002, MOD-0003]
    verifiedByDescription: "A docker-compose.yml at the repository root starts the full stack with docker compose up."
    verifiedByKind: grep_pattern

  - decisionExternalId: NA-authentication
    concernName: "Authentication"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — authentication is excluded per Chapter 2 constraints."
    rationale: Authentication is explicitly out of scope. The PoC handles no real data requiring protection and operates in a controlled local evaluation environment.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-authorization
    concernName: "Authorization"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — authorization is excluded per Chapter 2 constraints."
    rationale: Authorization is excluded alongside authentication. No role separation or access control is needed for the PoC.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-audit-compliance
    concernName: "Audit & compliance"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — no real data is processed; no regulatory or audit obligation applies."
    rationale: The PoC processes only simulated data in a controlled evaluation environment. No compliance framework applies.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-notifications-events
    concernName: "Notifications & events"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — no push notifications, alerts, or messages are sent to actors."
    rationale: The product does not deliver notifications in the conventional sense. The real-time display update mechanism is the async event architecture decision (FND-0003), not a notification concern.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-reporting-analytics
    concernName: "Reporting & analytics"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — reporting and analytics are explicitly excluded per Chapter 2 non-goals."
    rationale: No historical trend views, aggregate reports, or data exports are in scope.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-file-blob
    concernName: "File / blob handling"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — the product handles no files or binary objects."
    rationale: All data is numeric or string state held in memory. No file upload, download, or blob storage is involved.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-caching
    concernName: "Caching"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — the in-memory dataset is the authoritative source; no cache layer sits between a persistent store and the application."
    rationale: There is no persistent store to cache results from. The in-memory state maintained by the Flight Data Simulator is the primary and only data store.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-search
    concernName: "Search"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — no search, filtering, or query capability is in scope."
    rationale: The display shows all active flights simultaneously. No search or filter UI is provided.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-batch-jobs
    concernName: "Batch / scheduled jobs"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — the simulator runs as a continuous in-process loop, not as a scheduled batch job."
    rationale: Event generation is continuous and uniform. There are no periodic background tasks or scheduled operations.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

  - decisionExternalId: NA-data-migration
    concernName: "Data migration"
    concernListVersion: 1
    resolution: no_na
    problem: "N/A — no persistent database exists; there is no schema to migrate."
    rationale: All state is in-memory and generated fresh at startup. Schema migration is not applicable.
    optionsConsidered: []
    decision: "Not applicable."
    consequences: []
    revisitIf: []
    encodedIn: []
    verifiedByDescription: "N/A"
    verifiedByKind: not_applicable

entities:
  - entityExternalId: ENT-0001
    name: Flight
    owningModuleExternalId: MOD-0001
    purpose: Represents a scheduled airline flight currently within its 24-hour check-in window, along with its pre-computed aggregate check-in status figures used by the display.
    attributes:
      - name: flightId
        logicalType: identifier
        nullable: false
      - name: flightNumber
        logicalType: string
        nullable: false
      - name: origin
        logicalType: string
        nullable: false
      - name: destination
        logicalType: string
        nullable: false
      - name: departureTime
        logicalType: timestamp
        nullable: false
      - name: checkInWindowOpens
        logicalType: timestamp
        nullable: false
      - name: totalPassengers
        logicalType: integer
        nullable: false
      - name: checkedInOnline
        logicalType: integer
        nullable: false
      - name: checkedInCounter
        logicalType: integer
        nullable: false
      - name: bagsRegistered
        logicalType: integer
        nullable: false
      - name: bagsDropped
        logicalType: integer
        nullable: false
      - name: seatsAssigned
        logicalType: integer
        nullable: false
      - name: totalSeats
        logicalType: integer
        nullable: false
    relationships: []
    isCrossCutting: false

  - entityExternalId: ENT-0002
    name: PassengerRecord
    owningModuleExternalId: MOD-0001
    purpose: Represents a single passenger's check-in state on a specific flight, including their chosen check-in channel and seat assignment.
    attributes:
      - name: passengerId
        logicalType: identifier
        nullable: false
      - name: checkInChannel
        logicalType: enum
        nullable: true
      - name: seatAssigned
        logicalType: string
        nullable: true
      - name: checkInTime
        logicalType: timestamp
        nullable: true
    relationships:
      - toEntityExternalId: ENT-0001
        cardinality: many_to_one
        label: belongs to flight
    isCrossCutting: false

  - entityExternalId: ENT-0003
    name: BaggageRecord
    owningModuleExternalId: MOD-0001
    purpose: Represents a single piece of luggage associated with a passenger, tracking whether it has been registered and subsequently dropped at the counter.
    attributes:
      - name: baggageId
        logicalType: identifier
        nullable: false
      - name: status
        logicalType: enum
        nullable: false
    relationships:
      - toEntityExternalId: ENT-0002
        cardinality: many_to_one
        label: belongs to passenger
    isCrossCutting: false

intermoduleFlows:
  - flowExternalId: FLOW-0001
    entityExternalId: ENT-0001
    fromModuleExternalId: MOD-0001
    toModuleExternalId: MOD-0002
    direction: push
    consistency: async-at-most-once
    fieldsCarried: ["flightId", "checkedInOnline", "checkedInCounter", "bagsRegistered", "bagsDropped", "seatsAssigned"]
    encodedIn: [FND-0003]

  - flowExternalId: FLOW-0002
    entityExternalId: ENT-0001
    fromModuleExternalId: MOD-0002
    toModuleExternalId: MOD-0003
    direction: push
    consistency: async-at-most-once
    viaInterfaceExternalId: IFC-0001
    fieldsCarried: ["flightId", "checkedInOnline", "checkedInCounter", "bagsRegistered", "bagsDropped", "seatsAssigned"]
    encodedIn: [FND-0003]
---

The PoC is composed of three modules across two subsystems. The backend is a single Rust process hosting both the Flight Data Simulator — which owns the domain model and continuously generates passenger and baggage events — and the Event Stream Server, which bridges simulation state changes to connected browsers. The frontend is a React single-page application built with the IBM Carbon Design System that receives the live event stream and renders the check-in monitor grid. Seven foundation decisions cover the active architectural choices and technology constraints; the remaining seven of the fourteen standard concerns are not applicable at this scope.
