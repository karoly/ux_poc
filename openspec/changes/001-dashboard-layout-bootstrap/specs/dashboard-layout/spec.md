# dashboard-layout: Per-flight check-in status grid

## Purpose

This capability defines the read-only grid that ground-handling staff and
Amadeus Technical Leadership see when they open the Check-in Monitor
Dashboard. It governs the layout structure (one card per active flight),
the fields each card surfaces, the constraint that all visual components
come from the IBM Carbon Design System, and the read-only posture (no
input controls). The application of incoming live updates to the rendered
state is governed by capability `live-update-reception` and is out of
scope here.

## Dependencies

- Foundation decisions: FND-0006, FND-0007
- Actors: ACT-0001, ACT-0002
- Modules: MOD-0003
- Interfaces: IFC-0001, IFC-0002
- Entities: ENT-0001 (read-only consumer; ownership remains with MOD-0001)

## Requirements

### Requirement: Grid presentation of every active flight

<!-- forge-id: REQ-0001 -->

The dashboard SHALL render one visually distinct card for every flight
present in the current client-side state, where current state is the
collection of flights last received from the Event Stream Server via
IFC-0001 / IFC-0002. When the current state contains at least 100
flights (the simulator's stated minimum active count per FND-0001 §
Affects), the dashboard SHALL render all such flight cards in the DOM.
When the current client-side state is empty (initial load before any
event has arrived), the dashboard SHALL display the empty grid with no
cards and no error indicator (consistent with the passive-degradation
posture of FND-0002).

#### Scenario: Grid renders one card per flight in the current state

- GIVEN the client-side state contains 100 active flights
- WHEN the dashboard is rendered
- THEN the rendered DOM contains exactly 100 flight-card elements, one
  per flight in the state, each tagged with the flight's identifier so
  the test harness can match cards to source flights

#### Scenario: Empty state renders the grid with no cards

- GIVEN the client-side state contains zero flights (initial load)
- WHEN the dashboard is rendered
- THEN the rendered DOM contains zero flight-card elements and no error,
  warning, or stale-data indicator is present

#### Scenario: Grid scales to the simulator's stated minimum

- GIVEN the client-side state contains 100 active flights
- WHEN the dashboard is rendered in a viewport of 1920×1080 pixels
- THEN every one of the 100 flight-card elements is present in the DOM
  (off-screen scroll is acceptable; the harness asserts presence in the
  rendered DOM, not initial visibility)

### Requirement: Each flight card displays the named status fields

<!-- forge-id: REQ-0002 -->

Each flight-card element MUST display the following fields, drawn from
the corresponding flight in the client-side state: the flight's
identifier (flightNumber), origin and destination, total passenger count
(totalPassengers), passengers checked in via the online channel
(checkedInOnline), passengers checked in via the counter channel
(checkedInCounter), bags registered (bagsRegistered), bags dropped
(bagsDropped), and a seat fill indicator derived from seatsAssigned and
totalSeats. Each field MUST be individually identifiable in the rendered
DOM so an automated harness can assert the displayed value matches the
state value. Fields SHALL NOT be combined into a single concatenated
string that the harness cannot decompose.

#### Scenario: Every named field is present and individually addressable

- GIVEN a flight in the client-side state with flightNumber "AF1234",
  origin "CDG", destination "JFK", totalPassengers 250, checkedInOnline
  120, checkedInCounter 40, bagsRegistered 130, bagsDropped 110,
  seatsAssigned 160, totalSeats 250
- WHEN the dashboard is rendered
- THEN the flight-card for AF1234 contains nine individually addressable
  elements, each carrying the corresponding state value, such that a
  test harness can assert each value independently

#### Scenario: Counter and online check-in counts are shown as separate fields

- GIVEN a flight with checkedInOnline 120 and checkedInCounter 40
- WHEN the dashboard is rendered
- THEN the card displays the two counts in two distinct DOM elements
  (one labelled for online, one labelled for counter), not as a sum

### Requirement: Read-only surface with no input controls

<!-- forge-id: REQ-0003 -->

The dashboard MUST NOT render any element that accepts user input —
specifically, no `<button>`, `<input>`, `<select>`, `<textarea>`,
`<form>`, or contenteditable element — anywhere in the layout produced
by this capability. The surface SHALL communicate exclusively
information from the client-side state to the viewer; nothing the
viewer does to the rendered DOM (clicks, keystrokes, focus changes)
results in a state change or a request to the server. This requirement
encodes the Ch 2 constraint that the application is "Not an interactive
tool" and the IFC-0001 / IFC-0002 contract that "no user input is
accepted; the interface is strictly read-only."

#### Scenario: No input-accepting elements in the rendered DOM

- GIVEN the dashboard is rendered against any non-empty client-side state
- WHEN the rendered DOM is queried for input-accepting tag types
  (`button`, `input`, `select`, `textarea`, `form`, or any element with
  `contenteditable="true"`)
- THEN zero matches are returned

#### Scenario: Click events on flight cards do not raise outbound requests

- GIVEN the dashboard is rendered with at least one flight card
- WHEN the test harness dispatches a click event on a flight card
- THEN no outbound HTTP request is issued by the page and no client-side
  state mutation is triggered

### Requirement: Components drawn from the IBM Carbon Design System

<!-- forge-id: REQ-0004 -->

Every visual component used to compose the dashboard layout — the grid
container, the flight cards, the field labels, the field values, the
seat-fill indicator — SHALL be sourced from the IBM Carbon Design System
component library (the package named in FND-0006). Bespoke or
hand-rolled visual primitives SHALL NOT be substituted where a Carbon
component covers the same role; this constraint is what makes the PoC's
usability feedback attributable to Carbon specifically rather than to
ad-hoc UI choices.

#### Scenario: The browser bundle imports from the Carbon component package

- GIVEN the production build of the dashboard
- WHEN the bundle's import graph is inspected
- THEN at least one import from the IBM Carbon Design System component
  package is present, and every component rendered into a flight card
  resolves transitively to a Carbon component

#### Scenario: No bespoke visual primitive substitutes for a Carbon component

- GIVEN the dashboard source tree
- WHEN a static-analysis pass enumerates the React components rendered
  inside any flight card
- THEN every such component either is a Carbon component or composes
  Carbon components without introducing a same-role hand-rolled visual
  primitive (e.g., a custom card frame where Carbon's `Tile` would apply)
