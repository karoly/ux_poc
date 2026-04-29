# Kernel workspace template

The skeleton filesystem a workspace pod sees when a fresh product is provisioned. The Forge backend mounts this tree into the pod's `/workspace` at boot, then the agent inside the pod operates against it.

## What's in here

```
workspace/
├── CLAUDE.md           — In-pod system bootstrap. Tells the agent to call
│                         `forge_status` for directives rather than reading
│                         layer files directly. Replaces the standalone
│                         template's `CLAUDE.md` for the v2 platform path.
├── .claude/
│   ├── settings.json   — Permission mode + status line config for the pod-
│                          side Claude Agent SDK runtime.
│   ├── rules/          — Path-scoped rule files (rust.md, frontend.md,
│   │                     database.md, helm.md, specs.md, testing.md,
│   │                     docker.md, zitadel.md). Each one auto-loads
│   │                     when the agent edits matching files; pointers
│   │                     into `solution/kernel/memory/` references.
│   ├── hooks/
│   │   └── verify.sh   — Stop-hook that runs the `verify.json` ladder before
│   │                     the agent can declare a turn complete.
│   └── skills/         — Empty. Slash-command bundles for in-pod agents
│                          go here when authored. (Session 40 deleted the
│                          v1 `import-tickets` / `import-docs` skills —
│                          designed for the v1 standalone-template "FDE
│                          imports tickets for Layer 0 adoption" workflow,
│                          which has no v2 pod consumer.)
├── solution/           — Empty `.gitkeep`. The agent populates this during
│                          Layer 3.
└── specification/
    ├── business/       — Empty `.gitkeep`. Populated during Layer 1.
    ├── technical/      — Empty `.gitkeep`. Populated during Layer 2.
    └── validation/     — Empty `.gitkeep`. Populated during Layer 1 Phase 10.
```

## How content reaches the runtime

The pod-provisioning code (in `forge-workspace`) clones this tree into a fresh persistent volume on first product creation. From then on the per-product working tree lives independently of this template — edits to the per-product copy don't propagate back, and edits to the template don't retroactively touch existing pods.

When this template changes (new rule file, hook update, CLAUDE.md revision), the pod-provisioning logic decides per-pod whether to refresh — typically only on a fresh provision, not on every pod restart, to avoid clobbering in-flight work.

## How to change it

- **Adding a rule file** (e.g., a new `.claude/rules/<domain>.md`): mirror the existing pattern (one-line summary at top, brief body, pointer to `solution/kernel/memory/technology/<domain>/...`).
- **Editing CLAUDE.md**: this is the in-pod bootstrap; treat it like a system prompt. Test changes against a freshly-provisioned product before considering them landed; in-pod prompt drift is high-blast-radius (every product feels it).
- **Removing a file**: make sure no live pod's working directory inherits a stale reference (the `.claude/rules/` files are name-matched at agent boot — a deleted-but-still-referenced file silently no-ops).

## Cross-references

- Provisioner: `crates/forge-workspace`.
- Backend route triggering provision: `crates/forge-server/src/routes/products.rs` (POST /products).
- LLD reference: §C.10 (workspace pod lifecycle) + §C.18.1 (in-pod tool surface).
- This template's `CLAUDE.md` ≠ the top-level repo's `CLAUDE.md` — the former runs inside a workspace pod against `forge_status` directives; the latter runs in Claude Code against the Forge state machine for kernel/platform development. Don't merge them.
