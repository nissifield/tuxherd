# Tuxherd Workspace Capability and MAM

**Status:** Proposed design supplement · **Date:** 2026-10-26

This supplements [DESIGN.md](../DESIGN.md), especially its work/personal separation, trust boundaries, TRP profiles and roadmap. It proposes extensions, not shipped functionality. The TRP-Workspace 2.0 proposal below explicitly supersedes the workspace profile definition in DESIGN §11.1; other base-design rules continue to apply unless stated here.

## Purpose and limits

Tuxherd should support mobile application management without device enrollment (MAM-WE) as **T0: Unenrolled Workspace**. A tenant manages its apps, identity and encrypted work data on a personal Linux host without acquiring authority over that host. Signed OCI workspaces also provide an app delivery option inside enrolled T1 BYOD and T2 corporate environments; DESIGN §7.2's AppStream, Flatpak and native delivery remain available.

The core rule is: **everything the tenant ships is a digest; everything the user creates is ciphertext; everything else is temporary.** Ciphertext means persistent user data; working plaintext necessarily exists while unlocked. Persistence exceptions are public registration metadata, verified image caches, wrapped DEK/lease blobs, epoch and lease counters, the audit chain head and the workspace-instance key (or its wrapped form). These exceptions must not contain plaintext work content or tokens.

T0 protects against loss of a locked device, offboarding and honest-user mistakes. A separate UID resists non-root personal-session malware reading workspace storage, sockets and process memory. When workspace windows are shown by the personal user's compositor, personal-session code that compromises the compositor can capture workspace pixels and input. That applies especially when the personal user is an administrator: malware can also capture sudo/polkit credentials and become root. T0 does **not** protect against a hostile device owner, root or compromised host. Containers share the host kernel; immutability does not change that boundary.

**Decision:** ordinary internal data may use the bridged model if tenant policy accepts those risks. DESIGN §9.2's separate seat session with its own compositor is the stronger option and is **REQUIRED for anything above ordinary internal on T0**, subject to the absolute exclusions below. It still shares the host kernel and does not resist root. Bridged local unlock SHOULD use FIDO2 user presence (`hmac-secret`) rather than a PIN/password typed into a workspace window; user presence does not attest the compositor or prevent abuse during unlock.

**T0 is never appropriate for admin consoles, production credentials, high-classification secrets, or policy-designated restricted resources** such as payment-card secrets, clinical record repositories or export-controlled designs. These require a separately approved enrolled/managed environment or remote access architecture; enrollment alone is insufficient. “Regulated” is not a blanket exclusion: ordinary mail can contain GDPR-covered personal data. Data classes are assigned by policy per app/resource/audience, not by an enforcement point guessing content. Above-ordinary classes that are not excluded still need an explicit policy, separate seat and suitable assurance; this supplement grants none by default.

## Authority, capability and evidence

Tiers describe tenant authority, capabilities describe available mechanisms, and assurance describes evidence. **Capability is not assurance.** Installing support proves neither integrity nor correct operation. DESIGN §2 defines the authority tiers; §5.1 defines the evidence vocabulary. T1/T2 workspaces report workspace `assurance` independently of any device `trust_level`.

| Capability configuration | Available mechanism | Runtime claim limit |
|---|---|---|
| No distro workspace feature | Distro- or project-signed rootless OCI launcher, never tenant-supplied; encrypted storage where prerequisites exist, typically under the personal UID | `separation: none`; lowest-risk sessions only, no managed display/printing claim. Tenant processes MUST NOT run as container UID 0 (which maps to the personal UID in this mode). Unsupported prerequisites prevent launch. |
| TRP-Workspace 2.0 support installed | Owner-authorized work-user helper, filtering bridge, managed printing and lifecycle integration | May report UID separation and managed boundaries after local checks; installation alone remains unattested. |
| Support plus verified measured immutable host | Same mechanisms plus consented evidence against independently trusted references | Narrow at-rest, freshness and offline-lease decisions; no runtime-integrity or higher-class live-access guarantee. |

| Authority tier | Ownership / scope | Permitted evidence axis |
|---|---|---|
| T0: Unenrolled Workspace | No device enrollment; workspace-only authority | `assurance: workspace-unattested` or `workspace-measured-host`; no device `trust_level` claim |
| T1: Enrolled BYOD | `ownership: byod`; read-only allowlisted device posture and workspace-only actions, including posture-only BYOD without a work user | Device `trust_level: software / tpm-key / tpm-measured`; workspace `assurance` separately if a workspace exists |
| T2: Corporate | `ownership: corporate`; host management within the agreed ceiling | Same evidence vocabularies, separately evaluated; enrollment never implies measurement |

T0 registers a workspace-instance key bound to the user's IdP identity, not a device certificate. The protocol-fixed maximum permits only its workspace policy, status and `workspace.lock`, `workspace.wipe`, `workspace.revoke-keys`, `workspace.refresh`. There is no tenant root agent, arbitrary host query/script channel, device lock or device wipe.

Before registration, a T0 consent screen MUST list tenant-shipped code, tunnel/destination access and owner-side network exceptions, storage quota, allowed commands, the published report schema and every reported verdict. Evidence and the anonymous host-device-tenant signal require separate consent. Notify before workspace actions (emergency wipe notifies at execution); show all actions locally. The user can remove a workspace at any time, after a data-loss warning, and registration is retired. Removal of one definition must not delete siblings sharing its work user.

Evidence consent withdrawal removes the evidence claim and triggers policy re-evaluation, **not wipe**. Revoke any affected offline lease server-side; the instance where consent is withdrawn locks immediately even offline. Other connected cooperative instances lock when notified; disconnected copies remain bounded by their existing expiry. Keep cached data encrypted without deletion and deny renewal until policy is satisfied. No urgent offline revocation guarantee is implied.

## TRP-Workspace 2.0

**Decision and supersession:** TRP-Workspace 2.0 replaces the TRP-Workspace definition in DESIGN §11.1 and its 1.0 workspace checklist. It requires **TRP-Core packaging/OS-release items only** (C-PKG-01–03 and C-OSR-01–02), not the full Core profile, Desktop profile or an enrolled agent. TRP-Core is a release conformance profile; device enrollment is a TP/1 operation. Packages may exist with the agent disabled.

`systemd-homed` with LUKS2 **SHOULD** be used; fscrypt is permitted at reduced assurance with weaker metadata protection disclosed and lifecycle tests required. This deliberately relaxes base W-HOM-01's MUST for distro adoptability. Neither backend alone supplies remote tenant-key revocation; use the separate tenant-keyed volume described below.

The owner installs and updates distro packages. A polkit-gated setup may require root to allocate users/resources; distro services retain only narrowly scoped privileges under owner control. Tenant code receives no such authority.

| Contract area | Required behavior for the corresponding capability |
|---|---|
| User and storage lifecycle | One Unix user per **(local human user, resource tenant)**, not a machine-wide user shared by humans. Multiple workspace definitions for that pair may share the user but have separate instance keys, volumes and storage paths. Use homed preferably or disclosed fscrypt fallback; enforce quota and scoped removal. |
| Session activation and lock | For bridged sessions, an owner-side helper obtains the local unlock secret/user presence, activates the encrypted home and starts `user@UID` without seat login. Lingering is not an unlock mechanism. In bridged mode, personal-session lock/logout locks all dependent workspaces; in separate-seat mode use work-session lock/logout. Suspend and owner-wide lock affect all workspaces. Idle timeout, lease expiry and instance commands lock only the affected instance: stop its apps, unmount its volume and drop its keys; stop the work user manager and deactivate the home only when no sibling instance remains active. Refuse unlock if safe swap/hibernation is unavailable. |
| Rootless runtime and subid | Tested Podman/runtime, cgroup v2 delegation, isolated storage and network namespace. Allocate nonoverlapping subuid/subgid ranges through supported NSS subid or a predeclared fixed tenant-slot pool; never depend on ad hoc entries surviving host activation. |
| Owner-side network/privacy floor | Deny LAN, link-local (including mDNS) and host-loopback egress outside the tenant tunnel by default, including `host.containers.internal` aliases, IPv4/IPv6 and direct printer routes. Only the owner may grant explicit local exceptions; tenant `NetworkPolicy` cannot weaken the floor. Mask host-sensitive `/sys` (DMI, device/topology identifiers) and `/proc` (host processes, mounts and network state); expose only runtime-required views. |
| Desktop bridge and portals | In bridged mode, dedicated bridge UID, distinct from personal and work UIDs and not ptraceable by them; narrow authenticated per-instance sockets. Workspace-side portal backend runs as the work UID in its isolated bus/namespace and talks only to this bridge. No personal session bus or unrestricted host sockets. Protocol-global/version allowlist and deny tests below are mandatory. |
| Device-access broker | Owner-controlled, per-request grants for PipeWire audio/camera, FIDO hidraw and PC/SC smart cards; no broad uaccess ACL or raw host PipeWire socket for the work UID. Deny screencast nodes unless policy explicitly permits them. Broker IME/input methods and redact notifications; do not share personal input-method services or notification content by default. |
| Accessibility | Isolated workspace AT-SPI bus; no personal accessibility bus or cross-boundary text/input access. Any accessible bridge path needs explicit owner consent, policy and a tested restricted API. |
| Printing | Workspace PDF/virtual printer; preview/export policy; approved forwarding means IPPS to a tenant print service **through the workspace tunnel**. No host CUPS/socket/spool or raw printer devices. `printing: managed` also requires the egress floor to block direct printer bypass; an owner exception enabling bypass makes it unavailable. |
| Key and identity lifecycle | Mount/unmount tenant volume, memory-only working keys, FIDO2 user-presence unlock recommended, isolated identity broker and lease expiry handling. |
| Host ABI | Versioned `workspace-host-abi` for compositor, portals, runtime, networking, GPU and these brokers. Inspect injected GPU libraries locally; report only compatibility verdict, never library inventory. |

TRP-Attested, **not TRP-Workspace 2.0**, owns the optional evidence/TPM broker (A-BRK-01). A T0 host can install it without device enrollment, with separate evidence consent. The work UID receives no `/dev/tpmrm0` or `tss` membership.

A release's `/usr/lib/tuxherd/conformance.json` is a static claim valid for that release's lifetime, with published tests and supported combinations. A per-host runtime report is separate: checks run before launch, lease renewal and after relevant changes. Conformance tests cover denied flows, every denied Wayland global, lock/wipe scope, unsupported configurations, subid collisions and upgrades. In keeping with DESIGN §14.1, optional product features remain profiles; the small, closed capability set below is versioned with ABI 1 and reports runtime availability, not independent feature flags.

## Capability and compliance signals

Local checks may inspect users, services, sockets, namespaces, compositor, print routes and injected libraries, but **MUST NOT send that inventory to T0 tenants**. Publish `workspace-report-v1` on the consent screen. Its closed schema contains only tenant/workspace/user identifiers, image/policy digests, evaluation/expiry times, tier, assurance, ABI level, the fixed capability verdicts below, and per-claim source (`workspace-self`, `host-helper` [unattested], `host-tpm`, `server`), opaque evidence reference and verifier result (`accepted`, `rejected`, `unavailable`). Evidence references must not encode host identifiers. Identifiers, digests and opaque references use constrained formats; no arbitrary diagnostic strings, extensions or host inventory fields are allowed.

| Fixed verdict / ABI field | Meaning and verification limit |
|---|---|
| `separation: none / uid / unavailable` | Distinct work UID, isolated storage and sockets required for `uid`. Same-UID containers, Toolbox/Distrobox or personal-home/session-bus sharing cannot claim it. |
| `display_boundary: unmanaged / managed / unavailable` | Active display/portal paths use the filtering bridge or an isolated seat compositor with equivalent portal/flow restrictions. In bridged mode this **does not resist personal-session compromise of the compositor**. |
| `display_mode: same-uid / bridged / separate-seat / unavailable` | Fixed local configuration verdict, used with `display_boundary` to require the separate-seat model; at unattested assurance it is not proof against owner tampering. |
| `printing: unmanaged / managed / unavailable` | Managed path plus blocked direct printer/spool/network access within the supported boundary. |
| `host_evidence: unverified / measured / unavailable` | Fresh evidence accepted against named reference policies, with scope recorded by the verifier; no mutable runtime attestation. |
| `workspace_abi: 0 / 1` | `1` means **runtime self-check reported ABI 1**, not verified integrity or merely a package declaration; `0` means unavailable/failed. |
| `workspace_integrity: unverified / digest-pinned / unavailable` | Accepted image/signature/policy versions pinned; actual execution remains a cooperative claim against host root. |
| `gpu_compatibility: compatible / incompatible / unavailable` | Local ABI check only; no host library names, versions or paths transmitted. |
| `host_device_tenant: none / read-only / root-capable / undisclosed` | Separately consented anonymous host-authority signal; no device-tenant name. `undisclosed` cannot satisfy a requirement for `none` or `read-only`. |

`assurance: workspace-unattested / workspace-measured-host` is the workspace-subject counterpart of device `trust_level`, not an equivalent enrolled-device `tpm-measured` evidence level. On unattested hosts, requirements are **configuration-correctness gates against accidents and honest misconfiguration, not controls against a dishonest owner**. Missing/stale/failed checks block the affected launch/renewal or permit only an explicitly authorized lower class; never silently substitute an unfiltered path.

Evidence reports prefer nonce-bound quotes and signed PCR 11 / pcrlock policy verification without raw event logs. This version **does not transmit raw event logs**; unsupported verification yields `unavailable`. Local log parsing must not leak firmware versions, paths, db/MOK certificates or serial-bearing identifiers. Per-tenant AKs reduce correlation, but EK handling at a verifier can still correlate hosts; disclose verifier exposure/retention on the consent screen. No anonymity guarantee is implied.

### IdP and SSF enforcement

**T0 workspace subjects MUST NOT be emitted as `device-compliance-change` or as an SSF `device` subject.** Use a proposed SSF complex subject with `user: {format: iss_sub, iss: <IdP issuer>, sub: <user subject>}` and a non-`device` member `urn:tuxherd:workspace: {format: opaque, id: urn:tuxherd:workspace:<id>}`. Signed SETs carry proposed namespaced events `urn:tuxherd:event:workspace-posture-change` (fixed verdicts, assurance and expiry) and `urn:tuxherd:event:workspace-revoked`; user/session revocation may additionally use CAEP `session-revoked` for the explicitly bound session. These workspace events require a Tuxherd-aware receiver; they are not generic CAEP device compliance.

A Tuxherd-aware IdP authenticator (e.g. the DESIGN §6.4 Keycloak integration) MUST require proof of the registered workspace key plus a short-lived, audience-bound posture token signed by the verifier/key broker. IdP rules must restrict each covered app/resource/audience's tokens to eligible registered workspace keys, and deny a host-browser flow without that proof. **Without those rules, host-browser access bypasses T0 data policy.** Registration proves identity and possession of a registered key, **not an approved client**; a dishonest owner can register or emulate software. DPoP binds supported OAuth tokens, while managed-browser SaaS session cookies remain bearer tokens unless DBSC or a comparable service-side binding is deployed. Key release and session revocation cannot recall copied cookies.

## Policy mapping to data classes

`AssuranceMapping` binds policy-assigned app/resource/audience classes to authority, evidence and capabilities. The IdP gates tokens, the key broker gates decryption leases, and the workspace implements cooperative local flow policy. This is a minimum mapping, not a grant:

| Data class/use | Required capabilities and policy | T0 ceiling |
|---|---|---|
| Public / low-risk SaaS session | Approved image and identity policy; `separation: none` permitted; no sensitive local cache | Lowest-risk access, no personal/work isolation claim |
| Ordinary internal mail, chat and documents | `separation: uid`, `display_boundary: managed`, `printing: managed`, ABI 1; deny exports/printing unless allowed; short online lease | Bridged model allowed only with explicit tenant acceptance of compositor/input risk; separate seat recommended |
| Offline cache of the same internal class | Preceding requirements plus measured evidence, acceptable boot/image freshness and TPM-bound lease/counters | Bounded offline lease only; no class promotion |
| Above ordinary internal, excluding forbidden classes | Explicit separately approved policy, separate seat/compositor, required evidence and capabilities | Separate seat REQUIRED; denied until that policy exists |
| Admin consoles, production credentials, high-classification or policy-designated restricted resources | Approved enrolled/managed environment or remote access policy and evidence | **Never T0**, regardless of measurement or separate seat |

`WorkspaceDefinition` pins delivery/ABI; `DataProtectionPolicy` defines clipboard, drag/drop, files, URI, screencast and print flows; `IdentityBinding` defines accounts, DPoP and SSF; `LaunchPolicy` sets re-authentication and capabilities; `NetworkPolicy` sets tunnel/DNS/destinations subject to the owner floor; `KeyPolicy` sets release/rotation/leases. `AuditEvidence` records provenance/outcomes, not host inventory. Unknown classes default to denial.

## Workspace protocol

T0 uses a workspace-scoped HTTPS surface with TP/1 envelope/status conventions, **not the TP/1 device channel or device certificate**. Discovery publishes the endpoints and pinned tenant trust root. Registration uses short-lived IdP authentication plus nonce-bound proof of the instance key and local consent; subsequent calls use instance-key-signed requests binding method, path, body hash, audience, nonce and expiry over TLS. The server checks replay, scope and registration state.

| Proposed endpoint | Function |
|---|---|
| `POST /sp1/workspaces/register` | Bind user, resource tenant and instance public key; return instance ID, signed policy/manifest and check-in limits. No host inventory. |
| `POST /sp1/workspaces/<id>/check-in` | Submit signed fixed-schema status deltas and last command receipt; receive signed manifest references and pending command IDs. Default maximum interval 5 minutes including jitter; policy may shorten it. |
| `POST /sp1/workspaces/<id>/leases` | Evaluate policy/evidence and issue or renew a bounded key lease. |
| `GET /sp1/workspaces/<id>/commands` | Fetch signed, scoped command envelopes; acknowledge results through check-in. |
| `GET /sp1/workspaces/<id>/stream` | Optional authenticated wake-only stream; carries no commands/content; polling remains required. |

Revocation stops future server releases immediately. A connected cooperative instance learns commands within the check-in interval plus delivery delay; offline action waits for reconnection or lease expiry. Report those bounds rather than claiming instantaneous remote wipe.

### Keys and leases

```text
volume DEK → wrapped by independent instance KEK → wrapped by tenant KMS key
```

The tenant KMS key wraps independently random instance KEKs stored only server-side; the instance KEK wraps that instance volume's DEK. No shared KEK is destroyed for a single-instance wipe. On release, the broker unwraps the instance KEK inside KMS/HSM-backed processing, unwraps the DEK and HPKE-encrypts it to a certified instance recipient key (or a registered software recipient key at unattested assurance). Signing and recipient keys are distinct and registration-bound. Only the workspace unwraps the response locally; working DEKs are memory-only.

A broker-signed lease contains version, tenant/user/instance IDs, recipient-key thumbprint, audience, volume/lineage ID, image and policy hashes, minimum security epoch, lease sequence/counter floor, issued/not-before/expiry times, online/offline scope and the encrypted DEK envelope. Verify the pinned delegated lease-signing key, audience, hashes, epoch and counter before mount. The lease signer is a narrowly scoped KMS-backed delegation under the tenant TUF root, not command authority.

On an unattested host, persistent work secrets are restricted to encrypted cache, wrapped DEK/lease blobs and wrapped keys, alongside the registration/image/audit persistence exceptions listed above; offline enforcement depends on cooperative local time/counters and rollbackable software secrets. Consequently this proposal allows **no sensitive internal offline cache at unattested assurance**. Public low-risk offline data may have a policy-bounded software lease without a hardware guarantee. TPM-backed offline leases seal the recipient secret to an approved signed PCR policy and enforce expiry using TPM clock/reset state plus NV counters; fail closed on uncertain clock continuity or rollback. TPM policy limits **unsealing**, not use of keys already unsealed; root can retain an already released DEK.

### Commands and wipe scope

Commands use DESIGN §10.3's per-tenant TUF root and roles: `admins` threshold signatures for `workspace.lock`, `workspace.wipe` and `workspace.revoke-keys`; `policy` for routine `workspace.refresh`; `timestamp` for freshness. Verify target tenant/instance, ceiling, nonce, expiry, minimum epoch and required transparency inclusion. Server identity/session revocation may independently deny access but does not authorize local destructive commands.

`workspace.lock` saves encrypted state, stops apps, unmounts the volume and drops keys. `workspace.revoke-keys` denies renewals and invalidates the instance's server wrapping record; a cooperative live instance locks. `workspace.wipe` additionally destroys the instance KEK and all recoverable broker/backup copies for that instance and requests local tenant-volume removal. A pending retention/recovery copy means shred is incomplete and must be reported as such. It MUST NOT remotely remove a shared homed user or sibling definitions. Local owner removal may later remove an empty user.

When two encryption layers exist (homed/fscrypt home plus tenant-keyed inner volume), the remote crypto-shred claim covers **only the tenant-keyed volume**, not the outer home. It requires elimination of every usable instance key, lease and recovery copy. It cannot recall plaintext, captured keys, exports or already unsealed secrets. Outer-home deletion alone is not this protocol's remote key-revocation guarantee; distinguish it from DESIGN §8.2's legacy work-user removal.

### TRP-Attested broker API (A-BRK-01)

The owner-controlled broker exposes per-instance `create` (key generation inside the TPM), `certify` (`TPM2_Certify` by a per-tenant AK), nonce-bound `quote`, `seal`, `unseal`, and `counter.read/increment` operations. It binds the recipient key to fresh boot evidence and signed PCR policies without arbitrary tenant TPM commands. No raw TPM device access is delegated. Per-tenant isolation includes a hard default budget of **two 8-byte NV counter indices per (human, tenant)**: one epoch floor and one lease/rollback counter shared by that pair's definitions. No extra indices per definition; slots/updates are rate-limited for NV capacity/endurance. Owners may lower the budget; exhaustion yields unavailable offline capability, never software downgrade. Updates across definitions must serialize monotonically; removal cleans up indices only after retiring all affected leases. AKs and sealed objects use broker-managed blobs rather than unbounded persistent TPM handles.

## Immutable workspace lifecycle

1. **Ship a signed OCI digest.** Verify against the pinned tenant TUF root/delegation, provenance/SBOM, minimum epoch, freshness and revocation. A mutable channel/tag resolves to an approved digest, never launch authority.
2. **Launch fresh.** Read-only root, no persistent writable application layer, tmpfs for temporary files/rendered signed policy; only allowlisted encrypted data/settings persist. `noexec` does not stop interpreters executing data.
3. **Evaluate release centrally.** Check registration, signer/digest, policy, epoch, revocation, fixed capability verdicts and evidence as required. Log releases. Without runtime attestation/TEE, a broker cannot prove a root-controlled launcher executes its claimed bits.
4. **Expire/revoke as specified above.** Reimaging replaces app state without deleting the volume. Offboarding revokes sessions/releases and requests scoped lock/wipe; short leases bound cooperative offline exposure.

Plaintext keys/tokens, crash dumps, logs, thumbnails, recent files and print spools must not become persistent host artifacts. Encrypt necessary token caches, disable dumps/plaintext container logging and define safe swap/hibernation before unlock. These are tested cooperative-host requirements, not guarantees against root.

Measured evidence must bind a fresh nonce and workspace key to the verified chain and independently trusted references. An immutable distro name or “Secure Boot enabled” is insufficient: validate signing authorities and signed PCR policies. Claim a filesystem digest only when cryptographically bound to measured boot. Mutable `/etc`, `/var`, user services and post-boot changes remain outside that evidence; measurement never upgrades `separation: none`.

## Screenshots, portals and printing

In bridged mode, work windows reach the personal compositor through the dedicated-UID bridge. Separate-seat mode has its own compositor, isolated buses and the same managed portal/flow restrictions; it must not forward work windows to the personal compositor. ABI 1 MUST publish a minimal allowlist of Wayland globals **and versions**, permitting only tested rendering/input paths. Deny unknown globals and versions by default; additions require ABI review and conformance tests. Deny ext/wlr data-control, screencopy/image-capture, foreign-toplevel enumeration, virtual keyboard, input-method and xdg-foreign protocols unless a specific restricted broker API is separately specified/tested. Test each denied global and prevent raw socket escape. `wp_security_context_v1` tags clients; it does not filter them. Claim support only for tested compositor/version combinations, including GNOME/Mutter and KDE; no assumption that tagging is supported or sufficient.

Filter clipboard, drag/drop, file import/export, open-URI and screencast by tenant/class. The workspace portal backend under the work UID mediates with the bridge (or its isolated compositor in separate-seat mode); never expose personal `$HOME`, session/AT-SPI buses, keyring/agent sockets or broad runtime directories. **No XWayland in the workspace.** IME, device, accessibility and notification paths must follow the contract above.

This can prevent workspace apps capturing personal windows and reduce accidental exports through supported APIs. It cannot promise screenshot blocking: personal-session compromise of the compositor can capture every workspace pixel and keystroke, and root or an external camera can capture too. Watermarks/audit deter; they cannot prevent capture. A dedicated bridge UID protects the filter from direct same-UID tampering, not from the compositor it must trust.

Applications print only to the workspace PDF/virtual printer. Deny, keep encrypted, or audit and forward by IPPS through the tenant tunnel; no host CUPS forwarding exception is permitted in this proposal. Previews/temp PDFs/spools stay encrypted or ephemeral. No direct discovery/printer path; owner network exceptions that permit bypass remove the managed-print verdict. Exported PDFs and physical output leave workspace control. Host capture/hostility defeats local print restrictions.

## Fedora and NixOS implementation notes

**Fedora Atomic first.** Prototype the work user, rootless Podman, encrypted volumes and bridge; test GNOME/KDE separately. Package availability proves no lifecycle or portal support. Measured-host capability depends on **sealed bootc/UKI boot plus cryptographic composefs-digest binding** and independently published reference policies; it is unshipped integration work, not a property of ordinary ostree immutability or a default shim/GRUB/BLS path. Until validated, report `workspace-unattested`.

**NixOS module later.** Declare a **fixed pool of tenant user slots**, nonoverlapping subuid/subgid ranges, services, polkit rules and portal/runtime integration. Do not rely on dynamically appended subid entries surviving declarative activation; tests must cover rebuilds and upgrades. The owner controls host rebuilds; tenant policy controls only workspace artifacts. A flake lock/store path proves no active boundary. **NixOS caps at `workspace-unattested` unless an independently trusted third party builds and signs the closure/boot chain and supplies verified references.** Owner-generated boot keys/references alone are insufficient for measured-host assurance.

Both distros inspect live compatibility locally after upgrades, including host-injected GPU drivers outside the pinned image; send only fixed verdicts. Block incompatible combinations instead of relaxing isolation. Publish minimum tested combinations before ABI 1 conformance; no measured-host claim precedes working boot-to-image verification.

## Multi-tenant rules

The isolation unit is the **resource tenant** that owns the data, including B2B guest access. Each (human, resource tenant) pair has its own work UID, network namespace, identity broker, pinned TUF root and rootless storage. Each instance has independently generated keys/volume; multiple definitions per pair do not imply isolation from that pair's other apps. Personal space is `tenant: none`. Same-UID fallback cannot claim these isolation properties.

Cross-tenant flows default to denial. Transfer needs both source egress and destination ingress permission, mediated/audited. No shared mutable profiles, caches or identity sessions across tenants. V1 uses one resource tenant per app/workspace; multiple identities inside one app are deferred.

A host may have multiple T0 workspace tenants but at most one T1/T2 device tenant. Any device-tenant workspace allowlist is enforced **locally only**, appears on the BYOD consent screen, and MUST NOT report identities of present, blocked or non-allowed tenants to the device tenant (including via rejection diagnostics). T0 tenants receive no general inventory or other tenant names.

The device tenant is in co-resident workspaces' TCB when its core can act as root, even for T1's read-only posture: a compromised root core can read unlocked data. T2 `root-full` authority makes this exposure explicit. With separate owner consent, expose only `host_device_tenant: none / read-only / root-capable`; absent consent use `undisclosed` and re-evaluate access. The read-only value describes the agreed authority, not a guarantee against core compromise. Preserve DESIGN's server-side per-tenant CA/KMS/database isolation.

## Threat-model additions (mirrored in DESIGN §10.6)

| Threat | Mitigation | Residual |
|---|---|---|
| Personal-session compositor compromise | Dedicated bridge UID and default-deny protocols; FIDO2 presence unlock; stronger separate seat recommended, required above ordinary internal on T0 | Bridged pixels/input are exposed; an administrator's captured sudo/polkit credentials can lead to root. Separate seat still shares the kernel. |
| Tenant code surveys personal LAN/host | Owner-controlled LAN/link-local/loopback floor, `/sys` and `/proc` masking, brokered devices, verdict-only reporting | Owner exceptions weaken protection; shared kernel and permitted external traffic retain fingerprinting/covert-channel risk. |
| Device tenant vs. co-resident workspace tenant | Local-only allowlist, consented anonymous host-authority signal, encrypted locked volumes | Root core compromise or T2 root authority exposes unlocked co-resident data; measured boot does not remove this TCB. |

## Roadmap impact and decisions

| Existing phase | Proposed supplement |
|---|---|
| 0–1: protocol/MVP | Reserve registration, resource tenant, T0 maximum, report/evidence schema and data-class mapping; no workspace desktop in Phase 1. |
| 2: configuration/software | **Phase 2b pilot: public/low-risk SaaS only**, signed immutable browser image, encrypted low-risk data, identity/network/key policy and short leases. No ordinary internal mail/cache before UID and managed display/printing exist. |
| 3: actions/attestation | Reuse verifier/reference/signature infrastructure; TRP-Attested owns A-BRK-01 and separately consented evidence. |
| 5: Workspace capability and MAM | T0/T1 lifecycle, bridged desktop and stronger separate-seat option, portals/printing, multi-tenant isolation and TRP-Workspace 2.0 tests; measured leases only after boot-to-image verification, then NixOS module. T1 may use either model under its policy; bridging does not inherit separate-seat strength. |

These additions need a new effort estimate for desktop, broker/verifier, tests and distro maintenance; DESIGN's base totals exclude them. Reuse OCI signing across T0/T1/T2 without raising T0's ceiling or displacing host management.

Settled proposal decisions are the TRP-Workspace 2.0 name/supersession, packaging/OS-release-only dependencies, homed SHOULD/fscrypt fallback, ordinary-internal bridge allowance with a separate seat required above it, TRP-Attested broker placement, tunnel-only IPPS forwarding, no raw event-log transmission, the bounded NV budget, one resource tenant per app in v1 and low-risk-only Phase 2b. They remain reviewable design choices, not shipped guarantees.

Before specification freeze, publish minimum tested ABI combinations, lease/clock and recovery-retention bounds, verifier reference distribution and EK privacy/retention policy, safe suspend/swap behavior, concrete encrypted-volume/print implementations and staged ABI upgrade/recovery procedures. End-to-end IdP and key-broker tests must demonstrate denial of forbidden classes and host-browser bypass attempts. These implementation details do not reopen the authority or privacy limits above.
