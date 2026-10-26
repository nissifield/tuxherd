# Tuxherd Workspace Capability and MAM

**Status:** Proposed design supplement · **Date:** 2026-10-26

This supplements [DESIGN.md](../DESIGN.md), especially its work/personal separation, trust boundaries, TRP profiles and roadmap. It proposes extensions; it does not describe shipped functionality or replace the base design.

## Purpose and limits

Tuxherd should support mobile application management without device enrollment (MAM-WE) as **T0: Unenrolled Workspace**. A tenant manages its apps, identity and encrypted work data on a personal Linux host without acquiring authority over that host. Signed OCI workspaces also provide the app delivery model inside enrolled T1 BYOD and T2 corporate environments.

The core rule is: **everything the tenant ships is a digest; everything the user creates is ciphertext; everything else is temporary.** Ciphertext means persistent user data; working plaintext necessarily exists while the workspace is unlocked. Public registration metadata and verified image caches are explicit nonsecret persistence exceptions.

T0 protects against loss of a locked device, offboarding and honest-user mistakes. Separate Unix users can also resist ordinary non-root personal-session malware. It does **not** protect against a hostile device owner, root or compromised host. The host can inspect memory, intercept input, copy unlocked data and capture visible pixels. Containers share the host kernel; immutability does not change that boundary.

**T0 is never appropriate for admin consoles, production credentials or regulated/high-classification data**, including on measured immutable hosts. These uses require a separately approved enrolled/managed environment or remote access architecture; enrollment alone is not sufficient assurance either.

## Authority, capability and evidence

Tiers describe tenant authority. Capabilities describe available enforcement mechanisms. Assurance describes the evidence supporting a claim. These are separate axes: **capability is not assurance**. Installing a workspace-support feature is not proof that it is untampered, active or correctly configured.

| Capability ladder | Available mechanism | Evidence and access implications |
|---|---|---|
| No distro workspace feature | Rootless OCI launcher and encrypted workspace storage where prerequisites exist; typically the personal user's UID | `separation: none`; self-reported, lowest-risk session access only. No managed display or printing claim. Unsupported prerequisites mean launch is unavailable. |
| Workspace-support feature installed | Owner-authorized helper creates a separate work user; filtering display/portal bridge, managed print path and lifecycle integration | Can report `separation: uid`, managed boundaries and an ABI version after runtime checks. Still `workspace-unattested`; installation proves no integrity. |
| Feature installed + measured immutable host | Same mechanisms plus consented boot/image evidence verified against reference values | T0 remains unenrolled. `workspace-measured-host` supports narrower at-rest, patch-freshness and offline-lease decisions, not higher-classification live access or runtime trust. |
| T1/T2 enrolled/managed host | T1 adds enrolled, read-only BYOD posture with workspace-only actions; T2 adds corporate host management within its ceiling | Device identity and host controls are evaluated separately. Neither enrollment nor an installed agent automatically proves measured or runtime integrity. |

T0 registers a workspace-instance key bound to the user's IdP identity, not a device certificate. Its protocol-fixed maximum permits only its own workspace's policy, status and actions: `workspace.lock`, `workspace.wipe`, `workspace.revoke-keys` and `workspace.refresh`. There is no tenant root agent, arbitrary host query/script channel, device lock or device wipe.

The only proposed host-evidence exception is a user-consented, fixed-schema broker owned by the distro. Tenants cannot extend its queries or configure the host through it. Revoking evidence consent removes the evidence claim and triggers policy re-evaluation, not an automatic wipe. This explicitly extends the earlier proposal that all T0 reports remain unattested; ordinary workspace reports still are.

## Workspace Capability Standard / TRP-Workspace

The **Workspace Capability Standard** is the distro integration contract for stronger MAM without tenant host ownership. TRP-Workspace supplies its versioned packaging, APIs and conformance tests. The existing work-user profile needs a separately installable workspace-support path that does not require enrolling in TRP-Core device management.

The owner installs and updates distro packages. A polkit-gated setup operation may need root once to create users and allocate resources; this grants no persistent tenant authority over the host. Distro services may remain privileged for narrowly defined operations under owner control.

| Contract area | Required behavior for the corresponding capability |
|---|---|
| User and lifecycle helper | Per-tenant Unix user and encrypted home, preferably `systemd-homed`; owner consent; lock on suspend/logout; scoped removal and visible action history. Any fscrypt fallback must declare its weaker metadata protection and pass equivalent lifecycle tests. |
| Rootless runtime | Supported Podman/runtime versions, subuid/subgid allocation, cgroup v2 delegation, isolated storage, per-tenant network namespace and rootless networking such as `pasta`. |
| Desktop bridge | Versioned Wayland/portal filtering API, compositor security-context integration where supported, no direct personal session bus or unrestricted host sockets. Verify that every allowed path crosses the policy boundary. |
| Printing | Workspace-side PDF/virtual printer, policy-controlled preview/export and optional approved forwarding; no direct host CUPS socket or printer-device access. |
| Key and identity lifecycle | Encrypted volume mount/unmount, memory-only working keys, local re-authentication, identity-broker isolation and lease expiry handling. |
| Host ABI | `workspace-host-abi` contract for compositor, portal, container runtime, networking and GPU compatibility; explicit reporting of host-injected libraries. |
| Optional evidence broker | Consent-gated, fixed-schema, nonce-bound TPM quotes and event logs; verifier reference values and privacy rules. This is an additional attestation capability, not implied by workspace support. |

Conformance tests must exercise denied flows, lock/wipe scope, unsupported configurations and upgrades. A release's conformance declaration identifies intended support; runtime compliance must verify actual state before launch and lease renewal, and after relevant state changes.

## Capability and compliance signals

Signals identify the tenant, workspace instance, user, image/policy digests, evaluation time and expiry. Each claim records its source (`workspace-self`, `host-tpm` or `server`), evidence reference and verifier result. Carry tier, assurance and capabilities through TP/1 status and the SSF/CAEP integration; the IdP and key broker evaluate them rather than flattening them into one “compliant” flag.

| Example signal | Meaning and verification limit |
|---|---|
| `separation: uid` | Workspace uses a distinct host Unix user with isolated storage and sockets. Same-UID containers, Toolbox/Distrobox and personal-home/session-bus sharing cannot claim this. |
| `display_boundary: managed` | The active workspace display and portal paths traverse the filtering bridge. Does not mean the host cannot capture the workspace. |
| `printing: managed` | Workspace printing is mediated by the managed print path; direct printer/spool access is blocked within the supported boundary. |
| `host_evidence: measured` | A verifier accepted fresh boot evidence against named reference values. State exactly what was measured; this does not attest mutable runtime state. |
| `workspace_abi: 1` | The running host/workspace combination satisfies ABI version 1, not merely that a package declares it. |
| `workspace_integrity: digest-pinned` | Accepted image/signature/policy versions are pinned. On T0, local execution of those bits remains a cooperative claim against host root. |
| `assurance: workspace-unattested` / `workspace-measured-host` | Distinguishes self-reported workspace state from verified, narrowly scoped host evidence. Neither equals enrolled-device `tpm-measured` authority. |

Missing, stale, unsupported or failed checks cannot satisfy a required capability. Report the failure explicitly; block the affected launch/key renewal or apply only a policy-authorized lower access class. Never silently substitute an unfiltered display or print path.

## Policy mapping to data classes

`AssuranceMapping` binds data classes to tier, assurance, separation and required capabilities. The IdP gates applications/tokens, the key broker gates decryption leases, and the workspace enforces cooperative local flow policy. The following is an illustrative minimum mapping, not a grant of access:

| Data class/use | Required capabilities and policy | T0 ceiling |
|---|---|---|
| Public / low-risk SaaS session | Approved image and identity policy; `separation: none` permitted; no sensitive local cache | Lowest-risk access only; no claim of personal/work isolation |
| Ordinary internal mail, chat and documents | `separation: uid`, `display_boundary: managed`, `printing: managed`, `workspace_abi: 1`; exports/printing denied unless explicitly allowed; short online lease | Allowed only if tenant policy accepts T0 risk; view/edit inside workspace |
| Offline cache of the same internal class | All preceding requirements plus verified `host_evidence: measured`, acceptable boot/image freshness, TPM-bound lease and encrypted cache | May permit a bounded offline lease; no promotion to a more sensitive data class |
| Admin consoles, production credentials, regulated/high-classification data | Separately defined enrolled/managed-host policy and evidence, or an approved remote access design | **Never T0**, regardless of installed features or measured boot |

`WorkspaceDefinition` pins delivery and ABI requirements; `DataProtectionPolicy` defines clipboard, drag/drop, files, URI, screencast and print flows; `IdentityBinding` defines IdP accounts, DPoP and CAEP; `LaunchPolicy` sets re-authentication and required capabilities; `NetworkPolicy` defines tunnel/DNS/destinations; `KeyPolicy` sets release, rotation and offline leases. `AuditEvidence` records claim provenance and enforcement outcomes. Unknown data classes default to denial.

## Immutable workspace and key lifecycle

1. **Ship a signed OCI image pinned by digest.** Verify against the tenant's pinned TUF root and signer delegation, with provenance/SBOM checks, minimum security epoch, freshness and revocation metadata. Channels resolve to approved digests; a mutable tag is not a launch authority.
2. **Launch fresh instances.** Use a read-only root filesystem, no persistent writable application layer, and tmpfs for temporary files and signed policy rendered at launch. Persist only allowlisted encrypted data/settings. A `noexec` data mount reduces exposure but does not stop an interpreter or compromised app executing data as code.
3. **Separate keys from delivery.** A tenant key broker releases a bounded lease for the encrypted volume's DEK, wrapped under a per-tenant KEK with instance-specific wrapping/revocation scope. Bind the volume to tenant and workspace lineage. The identity broker holds scoped tokens bound to the workspace key where supported.
4. **Evaluate release centrally.** Check registration, allowed digest/signer, policy version, epoch floor, revocation, required capability reports and any fresh host evidence. Log releases in the tenant audit/transparency system. Without a TEE or runtime attestation, a broker cannot prove that a root-controlled launcher is executing its claimed digest.
5. **Expire and revoke.** Offboarding revokes identity sessions and future key releases, destroys the instance's server-side wrapping material, and requests workspace lock/wipe. A cooperative running instance saves into its encrypted volume, unmounts and drops keys. Offline access ends at lease expiry; urgent revocation requires short leases. Local clocks/counters are cooperative without stronger hardware support, and root can retain an already released key.

Reimaging replaces app state without deleting the data volume. Crypto-shredding is conditional on eliminating every usable key/lease and recovery copy; it cannot recall exported plaintext or keys captured while unlocked. Shared tenant keys must not be destroyed to wipe one instance if that would wipe other instances.

Plaintext keys/tokens, temporary data, crash dumps, logs, thumbnails, recent-file entries and print spools must not become persistent host artifacts. Keep necessary token caches encrypted; disable dumps and plaintext container logging, and define safe swap/hibernation behavior before unlock. These are tested implementation requirements within the cooperative-host model, not guarantees against root.

Measured-host evidence must bind a fresh nonce and workspace key to the verified boot chain and named reference values. An immutable distro name or “Secure Boot enabled” is insufficient: verify signing authorities and event logs. Claim the host filesystem digest only when it is actually cryptographically bound to measured boot. Mutable `/etc`, `/var`, user services and post-boot changes remain outside that evidence. Measured evidence never upgrades `separation: none`.

## Screenshots, portals and printing

Work windows reach the personal desktop through a filtering Wayland/portal bridge. Filter clipboard, drag/drop, file import/export, open-URI and screencast requests by tenant and data class. Do not pass through personal `$HOME`, the session D-Bus, agent/keyring sockets or broad runtime directories. **No XWayland in the workspace**; X11 compatibility must not bypass the managed boundary.

This can prevent workspace apps capturing personal windows and reduce accidental work-data export through supported APIs. It cannot promise screenshot blocking: **the host compositor or root can still capture visible workspace pixels**, and an external camera remains possible. Watermarks and audit can deter misuse; they cannot prevent capture. `display_boundary: managed` must never be presented as “screenshots impossible.”

Applications print only to a managed PDF/virtual printer inside the workspace. The policy may deny printing, retain the PDF inside encrypted storage, or permit audited forwarding/export to an approved destination. Preview, temporary PDF and spool storage follow the same encrypted/ephemeral rules. No direct host CUPS socket, discovery path or physical printer access is exposed. Once policy permits a PDF export or physical print, copies leave the workspace's control. Host capture and a hostile host also defeat local print restrictions.

## Fedora and NixOS implementation notes

**Fedora Atomic first.** Prototype on Silverblue/Kinoite or a bootc-derived host with a distro-packaged workspace-support feature, separate work users, rootless Podman, encrypted volumes and the Wayland/portal bridge. Validate GNOME and KDE paths independently; package availability alone does not establish compositor, portal or homed lifecycle support. Ordinary ostree immutability is not proof of a measured filesystem digest. Add measured-host claims only when the boot-to-image binding and reference-value verifier are implemented.

**NixOS module later.** An owner-enabled module should declare users, subuid/subgid ranges, services, polkit rules, runtime and portal integration. Tenant policy controls the workspace image and bundle; the owner controls host configuration and rebuilds, consistent with DESIGN.md's “policy source, not enforcer” limit. A module, flake lock or Nix store path proves neither the active generation nor the active runtime boundary. Compliance must inspect the running users, services, sockets, namespaces, compositor/backend, print routes and ABI on both distros, including after host updates.

GPU compatibility is part of the ABI. Report host-injected driver libraries explicitly; they are outside the pinned workspace image. Test host upgrades against supported workspace versions and block incompatible combinations instead of relaxing isolation.

## Multi-tenant rules

The isolation unit is the **resource tenant** that owns the data, including B2B guest access. Each isolated tenant has its own Unix user, independently generated KEK, network namespace, identity broker, pinned TUF root, encrypted volumes and rootless container storage. Personal space is `tenant: none`. The no-feature, same-UID fallback cannot claim these isolation properties.

Cross-tenant flows are denied by default. An A-to-B transfer requires both A's egress allowance and B's ingress allowance, mediated and audited at the boundary. No shared mutable profiles, token caches or identity-broker sessions; v1 uses one resource tenant per app/workspace rather than multiple tenant identities inside one app.

A host may have multiple T0 workspace tenants but at most one T1/T2 device-management tenant. That device tenant may restrict allowed workspace tenants within its agreed ceiling. Workspace registration remains distinct from device enrollment, and workspace tenants receive no general host inventory. Preserve DESIGN.md's server-side per-tenant CA, KMS and database isolation.

## Roadmap impact and open decisions

| Existing phase | Proposed supplement |
|---|---|
| 0–1: protocol/MVP | Reserve workspace registration, tenant identity, T0 hard maximum, capability/assurance/evidence fields and data-class mapping. Do not expand Phase 1 into shipping a workspace desktop. |
| 2: configuration/software | Add a Phase 2b Fedora Atomic pilot: signed immutable browser/mail image, encrypted data, identity/network/key policies and short leases. Establish the host ABI and report unavailable capabilities honestly. |
| 3: remote actions/attestation | Reuse reference measurements, verifier and signed-command infrastructure; design consented fixed-schema evidence separately from device enrollment. |
| 5: workspace | Combine T0/T1 work-user lifecycle, filtering display/portal/print paths, multi-tenant isolation and conformance tests. Add measured-host leases after reference measurements and boot-to-image binding exist; follow with the NixOS module. |

These additions require a new estimate for desktop integration, broker/verifier work, testing and distro maintenance; the base roadmap's effort totals do not include this supplement. Reuse the OCI/signing pipeline across T0/T1 workspaces and T2 hosts, while preserving separate authority and evidence. Avoid letting the easy T0 adoption path displace host management or weaken its access ceiling.

Open decisions before specification freeze:

- Define the independently installable TRP-Workspace dependencies, ABI versioning and minimum tested compositor/portal/runtime combinations.
- Fix the measured-evidence schema, verifier reference-value distribution and privacy policy, including per-tenant attestation keys and the correlating EK exposure during verification. Do not promise cryptographic anonymity from a retention policy.
- Set data-class lease limits, offline rollback protection, revocation latency, recovery/backup key retention and safe suspend/swap/hibernation rules.
- Select the encrypted-volume and managed-print implementations, and decide whether non-homed fallback configurations can meet the same capability contract.
- Specify consent withdrawal, required-capability failure behavior and staged rollout/recovery when host updates break the ABI.
- Confirm the one-resource-tenant-per-app v1 scope and how the IdP/key broker enforce T0's forbidden data classes end to end.

## Design inputs

This synthesis uses [DESIGN.md](../DESIGN.md) and three local advisory notes dated 2026-10-26: “Container question,” “Tuxherd - Intune-style MAM and tenant model,” and “Tuxherd - Immutable design and MAM without enrollment.” It carries forward the container-as-delivery conclusion, adds workspace capabilities, and narrows measured-host claims to boot evidence and at-rest/offline policy. The proposals above remain subject to implementation and conformance validation.
