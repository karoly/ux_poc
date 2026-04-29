# Chapter 2 — Goals

## Vision

Amadeus's airport ground handling staff monitor check-in processes through terminal-based interfaces that demand extensive training (measured in days of onboarding) and impose high cognitive load due to text-only, command-driven interaction. This proof-of-concept demonstrates that a modern graphical interface for check-in status display can deliver the same operational information with significantly reduced learning curve and improved readability — providing Amadeus's Head of Development with evidence to greenlight a broader UX modernization programme across their ground handling product suite.

## Stakeholders

### Airport Ground Handling Staff
- **Role:** Primary end-users who monitor check-in status during airport operations.
- **Authority:** Evaluate usability; provide feedback on whether the redesigned view meets their operational needs.
- **Success signal:** Staff can locate check-in status information without terminal command knowledge; task completion time for status lookup is observably shorter than the terminal equivalent.

### Amadeus Head of Development
- **Role:** Decision-maker evaluating the PoC outcome.
- **Authority:** Approves or rejects advancement to a full UX redesign programme.
- **Success signal:** The PoC demonstrates sufficient UX improvement to justify investment in a wider modernization effort.

### Amadeus Product Leadership
- **Role:** Strategic sponsors of the UX modernization initiative.
- **Authority:** Set programme scope, budget, and timeline if the PoC succeeds.
- **Success signal:** The PoC provides a credible design reference that can inform roadmap planning.

### Amadeus UX/Design Team
- **Role:** Consumers of the PoC as a design reference.
- **Authority:** Adopt, adapt, or extend the PoC's design patterns for future product redesigns.
- **Success signal:** The PoC establishes reusable visual patterns and interaction conventions for ground handling workflows.

## Objectives

### OBJ-1: Prove UX feasibility for check-in status monitoring
Demonstrate that a graphical, read-only check-in status view can present the same information currently available in the terminal interface, with measurably improved clarity and reduced time-to-comprehension.

**Observable signal:** A ground handling staff member unfamiliar with the new interface can identify the check-in status of a specific flight within 30 seconds without training, compared to the multi-day training required for the terminal equivalent.

### OBJ-2: Secure stakeholder confidence for broader redesign
Provide the Head of Development with a concrete, demonstrable artefact that supports a go/no-go decision on wider UX modernization.

**Observable signal:** The PoC is presented to the Head of Development and a decision (proceed, revise scope, or decline) is recorded.

### OBJ-3: Establish design language for ground handling UX
Define visual patterns, information hierarchy, and interaction conventions that can serve as a foundation for future Amadeus ground handling product redesigns.

**Observable signal:** The PoC produces a documented set of UI conventions (layout, colour usage, typography, status encoding) referenced by at least one subsequent design artefact.

## Value Delivered

| Value item | Observable signal |
|---|---|
| Reduced onboarding time for status monitoring | New users locate check-in status without prior terminal training |
| Improved operational readability | Check-in status information is parsed visually (colour, layout, grouping) rather than through text scanning |
| Executive decision enablement | Head of Development receives a functional demonstration and records a go/no-go decision |
| Reusable design reference | UX/Design team references PoC patterns in subsequent redesign work |

## Constraints

| Constraint | Source | Impact |
|---|---|---|
| Read-only scope | FDE direction | No write operations; the PoC displays check-in status but does not modify it |
| Single workflow | FDE direction | Only the check-in status display workflow is in scope; other ground handling workflows are excluded |
| Proof-of-concept fidelity | Project nature | The PoC is a demonstration artefact, not a production-ready system; production-grade reliability, scalability, and security are not required |

## Non-Goals

1. **Replacing the existing terminal system.** The PoC does not retire or modify the current terminal-based solution. It exists alongside it as a demonstration.
2. **Production deployment.** The PoC is not intended to be deployed into a live airport operations environment.
3. **Write operations on check-in data.** Staff will view status only; no capability to modify, override, or act on check-in records.
4. **Multi-workflow coverage.** Only the check-in status display is in scope. Boarding, baggage, gate management, and other ground handling workflows are excluded.
5. **Performance under production load.** The PoC does not need to handle concurrent users at airport-scale volumes.

## Success Criteria

1. The check-in status display presents all information categories currently available in the terminal view (flight, passenger count, check-in progress, status indicators).
2. A ground handling staff member can interpret the display without terminal-specific training.
3. The Head of Development views the PoC and records a decision on next steps.
4. The PoC's design conventions are documented in a form usable by the Amadeus UX/Design team.
