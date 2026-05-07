---
chapterKind: actors

actors:
  - actorExternalId: ACT-0001
    name: Ground-handling staff
    actorType: human
    isExternal: true
    role: Ground-handling staff monitor passenger check-in progress across active flights. During the PoC evaluation they interact with the dashboard as they would in an operational context, and their usability feedback — whether the layout is readable and the information easy to interpret quickly — is the direct pass/fail signal for the evaluation.
    purpose: To view and interpret real-time flight check-in status across all active flights.
    duties:
      - Monitor passenger check-in counts by channel (online and counter) across active flights
      - Track bag registration and drop-off counts per flight
      - Assess current seat fill status
      - Provide usability feedback on the display during the evaluation session

  - actorExternalId: ACT-0002
    name: Amadeus Technical Leadership
    actorType: human
    isExternal: true
    role: Amadeus Technical Leadership observes the evaluation session and interprets ground-handling staff feedback alongside the development-velocity data recorded during the PoC build. They hold the authority to approve or reject the candidate design system as a framework for Amadeus business applications.
    purpose: To evaluate the PoC outcome and make the framework adoption decision.
    duties:
      - Observe the ground-handling staff evaluation session
      - Assess the quality and consensus of usability feedback
      - Review development-velocity data (actual hours spent building and testing the PoC)
      - Make the final framework adoption decision

interfaces:
  - interfaceExternalId: IFC-0001
    actorExternalId: ACT-0001
    direction: outbound
    protocol: UI
    summary: The flight check-in monitor dashboard delivers a continuously updated view of all flights currently within their check-in window. Data is pushed from the server to the browser as a continuous event stream; the display refreshes automatically without any user action. Each flight entry shows total passenger count, check-in counts broken down by channel (online and counter), bag registration and drop-off counts, and current seat fill status. No user input is accepted; the interface is strictly read-only.
    replaySemantics: at_most_once
    idempotencyRequired: false
    encodedIn:
      - "Ch 2 — Display latency ceiling of 5 seconds"
      - "Ch 2 — Scope is strictly bounded to this document"
    sloContract: "p95 update delivery < 5000ms"
    failureMode: If the server push stream is interrupted, the display freezes at the last-received state. No error indicator is in scope for the PoC; the client may attempt to reconnect automatically, but no additional recovery mechanism is provided.

  - interfaceExternalId: IFC-0002
    actorExternalId: ACT-0002
    direction: outbound
    protocol: UI
    summary: Amadeus Technical Leadership accesses the same flight check-in monitor dashboard as ground-handling staff. Their interaction is observational — they view the running application during the evaluation session to assess the quality of the experience delivered by the candidate design system and to witness the interface that ground-handling staff are evaluating.
    replaySemantics: at_most_once
    idempotencyRequired: false
    encodedIn:
      - "Ch 2 — Display latency ceiling of 5 seconds"
      - "Ch 2 — Scope is strictly bounded to this document"
    sloContract: "p95 update delivery < 5000ms"
    failureMode: Same degraded behaviour as IFC-0001 — display freezes at last-received state on stream interruption; no recovery mechanism in PoC scope.

digitalTwinApplies: false
digitalTwinNote: The product reads exclusively from an internal test data generator that simulates flight and passenger events at a continuous, uniform rate. It does not observe or synchronise the lifecycle of any external real-world entity — no operational counterpart exists in the deployment scope and no live data feed is consumed. A digital twin pattern is not applicable.
---

Two actors interact with the system at its runtime boundary: ground-handling staff who use the dashboard operationally during the evaluation, and Amadeus Technical Leadership who observe it. The Amadeus delivery team, named as a stakeholder in Chapter 2, operates at the build boundary and does not interact with the running application; they are product-external and are not listed here.
