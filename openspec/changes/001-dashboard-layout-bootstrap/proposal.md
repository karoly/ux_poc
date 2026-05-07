# Bootstrap dashboard-layout capability

## Why

The Check-in Monitor Dashboard module (MOD-0003) has two capabilities named
in the locked HLD: `dashboard-layout` (CAP-0005) renders the per-flight grid;
`live-update-reception` (CAP-0006) receives and applies the SSE event stream.
This proposal bootstraps the first OpenSpec spec for `dashboard-layout` so
the layout contract — what the grid shows, which library the components come
from, the read-only posture — is captured before any frontend work begins.

Without this spec the IBM Carbon Design System usability evaluation (the
explicit purpose of the PoC per Ch 2) has no anchor: there is no record of
which fields each flight card must surface, no constraint that components be
drawn from the Carbon library, and no record that the surface is read-only
to ground-handling staff. Each of those is load-bearing for the evaluation
panel's feedback being attributable to Carbon specifically.

## What

Adds one OpenSpec spec at `specs/dashboard-layout/spec.md` with four
requirements covering: grid presentation of every active flight; the
fields each flight card must display; the read-only constraint (no input
controls); and the IBM Carbon Design System component-library constraint.
Every requirement is ADDED — there is no prior `dashboard-layout` spec.

## Out of scope

- Receiving and applying the SSE event stream (CAP-0006 territory; deferred
  to its own bootstrap proposal).
- Stale-state handling on stream interruption (CAP-0006 + FND-0002 passive
  degradation; deferred to CAP-0006's spec).
- Specific Carbon component selection per field (an L3 build-time decision;
  L2 fixes only the library, not the components).
- Latency budgets on update delivery (FND-0003 / IFC-0001 SLO; that's the
  Event Stream Server's responsibility, not the layout's).

## Anchors

- Ch 4 CAP-0005 (target capability)
- Ch 4 MOD-0003 (parent module — Check-in Monitor Dashboard)
- FND-0006 (React + IBM Carbon Design System — front-end framework constraint)
- FND-0007 (Docker + Compose packaging — affects MOD-0003 deliverable shape)
- ACT-0001 (ground-handling staff — primary evaluator)
- ACT-0002 (Amadeus Technical Leadership — observer)
- IFC-0001 (dashboard interface served to ACT-0001)
- IFC-0002 (dashboard interface served to ACT-0002)
