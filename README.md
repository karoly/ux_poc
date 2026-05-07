# UX PoC

UX PoC is a read-only web application built to serve a single, time-bounded purpose: providing a controlled environment in which the IBM Carbon Design System can be evaluated as a candidate framework for Amadeus business applications. It sits within the Amadeus UI modernisation programme and exists not as a deployable product in its own right but as an instrument of measurement, designed to generate two concrete outputs — qualitative usability evidence and quantifiable development-velocity data — within a fixed four-week window.

The application displays the real-time check-in progress of simulated airline flights that are currently within their check-in window. For each active flight it surfaces passenger counts, channel-level check-in breakdowns, bag registration and drop figures, and seat fill status, refreshing automatically as a purpose-built internal data generator produces new events. No production data is involved, no user interaction is required, and no external systems are contacted. The scope is deliberately narrow: every implementation decision is in service of the evaluation, not of operational utility.

The product serves three distinct parties whose interests converge on a single adoption decision. Ground-handling staff interact with the display during the evaluation session and provide the usability signal that indicates whether Carbon-based interfaces are readable and scannable enough for operational workflows. Amadeus Technical Leadership interprets that signal alongside the delivery team's actual build hours to decide whether Carbon warrants adoption at programme scale. The Amadeus delivery team constructs the PoC under measurement, and the friction or fluency they experience during development is itself evidence that feeds the evaluation. A credible result — positive or negative — is the only outcome that satisfies all three parties.

## Workspace

```
specification/
├── hld/             High-Level Design chapters (Layer 1).
│                    Chapters 2-5 are authored by the Designer agent
│                    in chat with the FDE; Chapters 1 and 6 are
│                    auto-generated from the authored content.
└── validation/      Validation scenarios (Layer 1).

openspec/            Module specs + change proposals (Layer 2).
solution/            Implementation source (Layer 3).
```

## How to make changes

Use the canvas — the Forge platform owns the artifact lifecycle (review, approval, version bumps, audit trail). Editing files here directly bypasses governance and is not recommended.
