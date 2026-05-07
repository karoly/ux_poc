# Product workspace

This Forge product workspace holds the design and implementation artifacts as the layers progress. The platform manages provisioning and lifecycle; the FDE drives the work via the canvas.

This README is auto-populated with the product's Introduction once Chapter 2 (Goals & constraints) is approved — until then the layout below is the only canonical content.

## Layout

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

The pod-side `CLAUDE.md` describes the agent's runtime contract for working inside this workspace; the platform-level docs live in the Forge repository, not here.
