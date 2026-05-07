---
chapterKind: goals
---

The UX PoC establishes a controlled evaluation environment for a named candidate front-end design system under consideration for Amadeus business applications. Every goal and constraint in this chapter is traceable to the Amadeus UI modernisation programme and to the client intake notes.

## 1. What the product does

The product is a read-only web application that displays the real-time check-in progress of airline flights currently within their check-in window. For each active flight the application shows:

- Total passenger count
- Passengers checked in, broken down by channel (online check-in versus counter check-in)
- Bags registered and bags dropped
- Seat fill status

The check-in window for any given flight opens 24 hours before departure. The application covers all flights at or after that threshold up to their departure time. Data is produced by an internal test data generator — not sourced from any external or production system. The generator continuously simulates at least 100 flights with a steady, uniform arrival of events (no peak-load bursts). The display refreshes automatically as the generator produces new events; no user action triggers a refresh.

The application's purpose is narrowly scoped to the evaluation of the IBM Carbon Design System as a candidate framework for Amadeus business applications. Every implementation decision within the PoC is in service of generating credible usability evidence and development-velocity data for that evaluation.

## 2. Who it's for

**Ground-handling staff** are the primary users of the application during the evaluation. Their job is to interpret passenger check-in status across active flights and act on it operationally. In the PoC they serve as the evaluation panel: their usability feedback — whether the layout is readable, the information scannable, and the experience comfortable for their workflow — is the direct pass/fail signal. They can report positive or negative feedback to Amadeus Technical Leadership and escalate concerns about specific interface elements. Their failure mode if the product under-delivers: the display is difficult to read or parse quickly enough to be useful, producing inconclusive or negative feedback that prevents a framework adoption decision.

**Amadeus Technical Leadership** holds the authority to approve or reject the IBM Carbon Design System as a candidate for Amadeus business applications. They interpret ground-handling staff feedback alongside development-velocity measurements to make that adoption decision. They value a clear, unambiguous evaluation signal — either a credible positive result or a credible negative one. Their failure mode if the product under-delivers: staff feedback is ambiguous or the PoC is not delivered within the 4-week window, leaving the framework evaluation open without a decision basis.

**The Amadeus delivery team** builds the PoC, and their build process is itself under measurement. Technical Leadership uses the actual development and testing hours to estimate what Carbon-based development would cost at programme scale. The team can surface implementation friction — or the absence of it — as structured input to the evaluation. Their failure mode if the product under-delivers: the PoC is too simple or too incomplete to generate a meaningful velocity signal, making the time-cost estimate unreliable for programme planning.

## 3. Value delivered

**Usability evidence for the candidate design system.** The observable signal is qualitative usability feedback from ground-handling staff collected during the evaluation session — specifically, whether staff report a positive experience with the layout, information hierarchy, and overall interaction model.

**Development-velocity data.** The observable signal is the actual number of developer hours spent building and testing the PoC over the 4-week delivery window. Technical Leadership uses this figure as the primary input to their cost-of-adoption estimate for the broader UI modernisation programme.

## 4. What this product is NOT

- **Not a production check-in system.** The application does not process, record, or transmit any real check-in transactions. It displays simulated data only.
- **Not connected to any Amadeus production database or external system.** All data originates from the internal test data generator. No network connection to production infrastructure is made.
- **Not an interactive tool.** Users take no actions within the application. There are no buttons, forms, or commands — the display is strictly read-only.
- **Not an authenticated or authorised service.** User identity is not verified. Access control is not in scope.
- **Not a load or stress-test harness.** The test data generator assumes a constant, uniform event rate. Peak-load or burst-traffic simulation is out of scope.
- **Not a monitoring or alerting platform.** The application does not send notifications, raise alerts, or trigger any external action based on threshold conditions.
- **Not a reporting or analytics tool.** Historical trends, aggregate statistics, and data exports are not provided.
- **Not an administrative or configuration interface.** Flight schedules, passenger records, and system parameters cannot be managed through this application.

## 5. Constraints the product must obey

- **4-week delivery window.** The complete, demonstrable PoC must be ready for the evaluation session within four weeks of project start. This is a hard programme deadline; the evaluation session cannot move.
- **Display latency ceiling of 5 seconds.** Changes in simulated check-in data must appear on the active display within 5 seconds of the event being generated. Latency beyond this threshold is a functional failure of the PoC.
- **Test data generator must support at least 100 simultaneously active flights.** The generator produces events at a continuous, steady rate with no simulated peaks. It is the sole data source for the entire application.
- **Technology stack is mandated by the Amadeus UI modernisation programme.** The back-end implementation language, front-end framework and design system, real-time event protocol, and deployment packaging are all pre-selected by Amadeus and are not open for substitution within this PoC's scope.
- **No connection to production systems.** The application must not attempt to connect to, read from, or write to any Amadeus production database or external service.
- **Scope is strictly bounded to this document.** Any capability not explicitly described in this chapter or the project intake notes is out of scope. Additions require explicit programme approval before any implementation work begins.
