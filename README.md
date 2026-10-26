# Tuxherd

**An open endpoint-management protocol and distro conformance profile for Linux** — the layer that would make any Linux install enterprise-manageable from day one, the way Windows is under Intune or macOS is under Apple's MDM.

Tuxherd (Tux + shepherd) has three deliverables:

1. **Tuxherd Protocol (TP/1)** — a wire protocol between a device agent and a management plane: identity and enrollment, declarative configuration and compliance, inventory, and signed remote actions.
2. **Tuxherd Ready Profile (TRP)** — a versioned, testable spec for what a distro ships so its installs speak TP/1 out of the box.
3. **Reference implementations** — a small agent (`tuxherdd`, Rust) and a minimal reference server to prove the spec.

It is deliberately **not** a Fleet/Rudder/Landscape competitor — it's the shared protocol layer underneath them.

## Status

Architecture proposal, draft 0.1 (2026-10-26). No code yet.

## Read the design

**[DESIGN.md](DESIGN.md)** — the full architecture document: gap analysis vs Intune-on-Linux, TPM-backed enrollment, declarative policy model, remote actions with BYOD safeguards, the distro conformance checklist, threat model, phased roadmap, and the open decision list.

## License

Documentation and specification: [CC-BY-4.0](LICENSE).
