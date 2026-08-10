# Posternity

**An Out-of-Band Management Enclave Reference Architecture for Research Computing**

*Specification and System Security Plan Skeleton — Draft 0.1*

---

## 0. Naming and Repository Topology

**Posternity** is the project: this specification, the sample codebase, and the reference architecture. Institutions deploy Posternity under their own deployment names.

| Layer | Name | Repository | Visibility |
|---|---|---|---|
| Project / reference architecture | Posternity | `atmarx/posternity` | Public |
| Reference deployment (fictional Northwinds University) | **Mudroom** (`mudroom.northwinds.edu`) | `northwinds/mudroom` | Public |
| Institutional deployment (example) | *institution's choice* (e.g., `mgmt-enclave`) | private | Private |

The Northwinds **Mudroom** deployment is the worked example threaded through this document: it demonstrates what a completed deployment looks like, with every institution-specific value filled in with fictional-but-realistic choices. Real institutions clone `posternity`, copy the structure of `mudroom`, and supply their own values from **Appendix A: Deployment Parameters**.

**Contribution discipline:** anything an institutional deployment needs that is not institution-specific is built in `posternity` and consumed downstream — never patched locally. The moment a private deployment carries a generalizable fix the reference does not, the reference architecture has started lying.

> **Why "Posternity"?** A *postern* is the small, guarded rear gate of a fortification — the private entrance used by those who belong there. The reference architecture is also the durable artifact written *for posterity*, while deployments are its ephemeral instantiations. Northwinds calls their deployment the **Mudroom**: the utility entrance to the house that holds the Hearth — where the muddy work happens, tools and deliveries come in, and nothing dirty tracks into the living space.

---

## 1. Purpose and Scope

Research computing fleets accumulate baseboard management controllers (BMCs) — iDRAC, iLO, OpenBMC, Supermicro BMC — one per server, each a small always-on computer with total power over its host: power control, console access, virtual media, firmware, sensors. BMCs are chronically under-secured (weak default credentials, slow firmware patching, a protocol with known cryptographic defects) and chronically over-exposed (flat management VLANs reachable by whole departments, or worse).

Posternity defines a **dedicated out-of-band (OOB) management enclave** that:

1. **Contains** all BMC traffic within a bounded, segmented network that no research data plane touches.
2. **Segments** tenants (research groups, clusters) so each can reach only its own BMCs.
3. **Gates** all human access through two identity-backed doors: a recorded web gateway for tenants, and an SSO-backed mesh VPN for platform operators.
4. **Audits** everything: full session recording, unified syslog egress, detective reconciliation.
5. **Curates** a software and media library so tenants never need to bring their own tools — and the enclave never needs general internet egress.
6. **Documents** itself as a System Security Plan mapped to NIST SP 800-171, positioned as a control *provider* from which connected systems' own security plans inherit.

**In scope:** the enclave network, its firewall/routing layer, its utility services, tenant jump hosts, the BMCs as connected components, and all access paths.

**Explicitly out of scope:** the research servers' production (in-band) interfaces, operating systems, and data planes. The enclave carries *administrative access to* systems that may process research data; it processes **no research data itself**. This claim bounds the security categorization (§10.3) and justifies several deliberate control decisions (§8).

---

## 2. Architecture Overview

```
                          Campus network
                               │
              ┌────────────────┼──────────────────┐
              │   users (browser, HTTPS)          │ operators (Headscale)
              ▼                                   ▼
        ┌──────────────────────────────────────────────┐
        │   pfSense HA pair (CARP) — 10GbE uplink      │
        │   inter-VLAN policy · DHCP · DNS · NTP       │
        │   Headscale/WireGuard termination            │
        └───────┬──────────────┬──────────────┬────────┘
                │              │              │
         Utility VLAN    Tenant VLANs    Operator VLAN
                │         (one /26 per       │
    ┌───────────┴─────┐   group; /24 per     └── hypervisor BMCs,
    │ Keycloak        │   cluster)               switch mgmt
    │ Guacamole+guacd │        │
    │ Forgejo         │   ┌────┴─────────────────┐
    │ Zabbix          │   │ per-group jump VM    │
    │ Syslog agg.     │   │ BMCs (iDRAC/iLO/...) │
    │ Media share     │   └──────────────────────┘
    └─────────────────┘
```

### 2.1 Network Layer

- **Segmentation:** one VLAN per tenant. Research groups receive a **/26** (62 usable addresses; groups typically manage 2–5 servers, so no renumbering, ever). Clusters receive a **/24** each, with identical logical treatment.
- **Switch-port tagging:** access ports carry the tenant VLAN untagged; BMCs remain ignorant of VLAN details. Adding a server to a tenant is a switch-port configuration action performed during intake (§9.1), which makes network membership part of the controlled intake process.
- **Addressing:** DHCP with **MAC-address reservations** — functionally static addressing with central inventory. The DHCP reservation table *is* the enclave's authoritative device inventory; an unreserved MAC requesting a lease is a detection event (§7.4).
- **Routing/policy:** pfSense is the sole inter-VLAN policy point. Default posture: **deny all inter-VLAN traffic**, with enumerated exceptions (§6). No tenant VLAN can reach another tenant VLAN under any rule.
- **Protocol containment:** IPMI-over-LAN (RMCP+) **never crosses the pfSense boundary** in either direction. IPMI 2.0's RAKP handshake discloses an offline-crackable password hash to any requester, and cipher suite 0 is an authentication bypass on permissive firmware. Remote users interact via HTTPS/SSH/KVM sessions terminated at the gateway or within their own segment. Redfish is preferred over IPMI wherever firmware allows; IPMI-over-LAN is disabled at intake on BMCs that will only be driven via web/Redfish/SSH (§9.2).
- **Bandwidth:** 10GbE at the enclave boundary, 1GbE to endpoints. All traffic is management-plane; the heaviest expected flow is occasional virtual-media ISO mounting (single-digit MB/s sustained). Capacity is not a design pressure and should not be "fixed" later.

### 2.2 Firewall / HA Layer

- Two **pfSense** instances, virtualized on a two-node Proxmox cluster, one instance affined per node, **CARP** for HA with **XML-RPC configuration sync** and **DHCP failover** (a CARP failover that loses reservation state must never gamble a BMC's lease).
- Virtualization notes (build-doc material): Proxmox bridges require MAC-spoofing/promiscuous allowances for the CARP VIP to float; multicast must pass on relevant bridges; disable hardware checksum offload in the pfSense guests.
- **Quorum:** a two-node Proxmox cluster has none. A **corosync QDevice** on any off-cluster host (utility NAS, single-board computer) is required so a single node loss does not strand the surviving node's ability to manage VMs.
- pfSense provides **DNS and NTP** to the entire enclave and is the only component that syncs upstream (campus NTP). BMCs and VMs never leave the enclave for time or names.
- pfSense terminates the **operator mesh VPN** (§3.2) natively, so the operator door survives the death of every VM on the cluster.

### 2.3 Utility Tier

Separate small VMs — never one consolidated utility host. Blast radius, independent patching, and each is a named SSP component with its own hardening line:

| VM | Role |
|---|---|
| **Keycloak** | Enclave identity provider; brokers the institutional IdP; local break-glass realm |
| **Guacamole** (+guacd) | Tenant access gateway; session recording |
| **Forgejo** | Library metadata, runbooks, tenant automation repos, CI |
| **Zabbix** | Availability monitoring, tenant self-service dashboards |
| **Syslog aggregator** | rsyslog/syslog-ng; collects all enclave logs; single TLS egress to enterprise log service |
| **Media share** | nginx (HTTP) and/or NFS; serves ISOs and firmware to BMCs and jump VMs |

On a two-node cluster, affine utility VMs opposite the active pfSense instance where sensible; Proxmox HA restart is the failover story.

### 2.4 Tenant Jump VMs

One **low-resource, dedicated VM per tenant group**, inside the tenant's own VLAN. The jump VM is the universal client: a real browser on the management network reaches every BMC web UI, HTML5 KVM console, and virtual-media function natively — no reverse-proxy fragility, no vendor-console quirks.

- Built from a **golden image** (Proxmox template) whose build definition (cloud-init / Packer + Ansible) lives in Forgejo. **Rebuilt monthly** from the current baseline; jump VMs are semi-disposable by design, which discourages snowflake accumulation. Durable tenant material belongs in the tenant's Forgejo org (§5.3), not on the VM.
- Baseline toolkit: `ipmitool`, `freeipmi`, `racadm`, Redfish tooling (DMTF `redfishtool`, vendor Redfish scripting collections, `python-redfish`), **Ansible** with `dellemc.openmanage`, `community.general.redfish_*`, and IPMI modules preinstalled, a pinned browser known to behave with the fleet's BMC consoles, and standard kit (openssh, tmux, curl, jq, rsync, tcpdump, minicom/screen for Serial-over-LAN, git preconfigured against the internal Forgejo).
- **Automation lives here.** Tenants run Ansible/Redfish/racadm from the jump VM via Guacamole SSH — which means automation sessions pass through session recording like everything else.

---

## 3. Access Paths

Exactly two doors. Both identity-backed. Nothing else.

### 3.1 Tenant Path — Guacamole

Browser → **Apache Guacamole** → (OIDC) → enclave Keycloak → institutional IdP (with institutional MFA) → the tenant's jump VM (RDP/VNC) and/or SSH targets within the tenant's VLAN only.

- **Every session is recorded** server-side: screen + input for graphical sessions, full typescript for SSH. Recordings are stored on a dedicated volume the user cannot reach, with defined retention (§7.3). Session *metadata* (who, which connection, when, duration) flows into the syslog stream.
- Connection visibility is driven by the Keycloak **groups claim** (§4): a tenant sees only their own connections. No Guacamole local accounts exist except the documented break-glass account (§8.3).
- **Clipboard and SFTP are bidirectional** — a documented risk acceptance (§8.1), not an oversight.
- **The half-screen doctrine** (user guide language): *the management window is for managing; your local browser is for searching.* Tenants troubleshoot with Guacamole on one half of the screen and their own local browser on the other. General web access does not exist inside the enclave and is not missed.

### 3.2 Operator Path — Headscale

**Headscale** (self-hosted Tailscale control plane) with the WireGuard data path **terminated on pfSense**. Available to platform operators only.

- Identities are **SSO-backed** (Headscale OIDC against the enclave Keycloak → institutional IdP) with **key expiry** — solving the two structural weaknesses of raw WireGuard (no MFA, no lifecycle) with infrastructure already in the stack.
- The operator peer list is short by design and attested periodically (§10.8).
- Operators use this path for: library refresh (push-in, §5.4), hypervisor and switch management, the hypervisors' own BMCs (§8.3), and incident response when the utility tier is degraded.

**Design consequence:** there is no tenant VPN. The per-group jump VM eliminated the automation use case that traditionally forces one; Guacamole SFTP and the internal library cover file movement; the operator mesh covers privileged work. "Two doors — one for tenants, one for operators, both identity-backed" is the boundary story.

---

## 4. Identity and Authorization

### 4.1 Identity

- A **dedicated enclave Keycloak** — deliberately *not* shared with any research data platform — brokers the **institutional IdP** as its sole upstream, with a local realm existing only for break-glass and development. A research-platform identity incident does not automatically become a management-plane incident, and vice versa.
- **ePPN** (eduPersonPrincipalName) is the stable identifier carried in tokens.
- The research data platform's identity chain (e.g., federated research IdPs, Globus Auth) does **not** appear anywhere in this system. The enclave must function precisely when the research platform is mid-incident.
- **Deprovisioning backstop:** because authentication brokers through the institutional IdP, separation from the institution kills login automatically even before any local group is touched. Local group removal is therefore about *timeliness and least privilege*, not about whether departed users retain access at all. State this layering explicitly in the AC narrative.

### 4.2 Authorization

- **Enclave Keycloak groups are authoritative and local**: `enclave-tenant-<group>`, `enclave-operators`, `enclave-admins`.
- Membership is an **explicit, requested, named grant**: the PI requests access for named individuals; the platform team enrolls them via the onboarding procedure; the grant is a recorded artifact. Management-plane access is deliberately **narrower than research-group membership** — everyone in a lab touches the lab's data; only the PI and one or two designated technical people should hold BMC access. Auto-mirroring the research roster into the management plane would be privilege inflation by automation.
- **One groups claim drives every enforcement point:** Guacamole connection-group visibility, Forgejo org/team mapping, Zabbix user-group → host-group permissions. No per-service rosters exist.

### 4.3 Detective Reconciliation (optional module)

Where the institution operates a research-platform group source of truth (e.g., Globus Groups), a periodic **read-only** job compares each `enclave-tenant-<group>` roster against the corresponding research group and flags enclave members who are no longer in the research group ("left the lab, still holds BMC access — review"). Failure-tolerant: if the upstream is unreachable, the check skips and alerts; nothing about access changes. Drift detection without dependency.

---

## 5. Software and Media Library

The enclave provides all supported tooling so tenants never need to bring their own — the mechanism that makes zero egress (§6) livable. The library is split along its natural seam:

### 5.1 Forgejo — Metadata (everything text, everything diffable)

- Checksum manifests for every binary on the media share
- `LICENSES.md` (§5.5)
- Firmware catalog definitions (e.g., Dell Repository Manager catalogs)
- BMC intake hardening script (§9.2), SoL helpers, support-bundle collectors
- Runbooks: reimage (iPXE and full-ISO paths), disk sanitization, "my BMC is unreachable" triage
- Allowlist request records (§5.6)
- Golden image build definitions; per-group Ansible inventory skeletons

**Why Forgejo:** OIDC group-claim → org/team mapping is in the free product (GitLab CE requires Premium for group sync, recreating the roster-drift problem the architecture exists to avoid); it is lightweight; Forgejo Actions covers the modest CI needs (image builds, manifest verification, playbook lint); and it is the community-governed hard fork of Gitea — a defensible provenance choice for a published reference architecture. Structure: a **`library` org** (platform writes, all tenants read) and a **per-tenant org** mapped from each tenant's Keycloak group, giving tenant automation a version-controlled home that survives monthly jump-VM rebuilds.

### 5.2 Media Share — Binaries (dumb endpoints for dumb clients)

Binaries never enter git (even LFS): the primary consumers are **BMCs mounting virtual media and pulling firmware catalogs**, which want plain unauthenticated HTTP/NFS URLs. A boring directory tree:

```
/isos/
  systemrescue/  gparted/  memtest86plus/  memtest86-free/
  shredos/  clonezilla/  ipxe/
  ubuntu-lts/  rocky/  alma/  debian/    (current + previous, incl. netboot minis)
  vendor-live/
/firmware/
  dell/<model>/       (DRM offline repository + standalone packages + DSU)
  supermicro/<model>/  hpe/<model>/  ...
  lvfs-mirror/        (optional, where fleet hardware is LVFS-covered)
```

Notable inclusions:

- **SystemRescue** — the free rescue-environment workhorse (GParted, ddrescue, testdisk, network tools)
- **ShredOS** — nwipe-based sanitization boot image; the data-destruction SOP points here
- **The iPXE bootstrap ISO** — arguably the most important item: mount the tiny ISO via virtual media, chainload everything else over HTTP from the deploy server. Recommended reimage path (virtual-media performance becomes irrelevant); full ISOs are the fallback.
- **Dell Repository Manager offline repo** — iDRAC "update from network share" pointed at the internal mirror: BMC firmware updates with zero BMC egress.

### 5.3 Manifest ↔ Share Reconciliation

A scheduled job (Forgejo Actions or cron on the media VM) hashes everything on the share against the manifests in git. **An unmanifested file or a hash mismatch is a detection event.** The share cannot silently diverge from the record; the reconciler is itself an integrity control (SI/CM evidence).

### 5.4 Refresh — Push-In, Never Pull-From-Inside

Operators download ISOs and firmware on workstations **outside the boundary**, verify checksums against vendor-published values **there**, then push the binary to the share and the manifest commit to Forgejo over the operator path. Who, what, and which hash are recorded in the commit. The verification step happens outside, by a human, before anything crosses the boundary — a stronger supply-chain posture than allowlisted vendor-domain egress.

### 5.5 LICENSES.md Scaffold

The **public reference repository ships manifests only — never firmware binaries** (vendor EULAs generally prohibit redistribution; internal hosting for entitled hardware is a different question than public re-hosting).

| Item | License / Basis | Redistribution notes |
|---|---|---|
| SystemRescue | GPL (aggregate) | Freely redistributable |
| GParted Live | GPL | Freely redistributable |
| Memtest86+ | GPLv2 | Freely redistributable |
| PassMark MemTest86 Free | Proprietary freeware | Verify current free-edition EULA permits internal hosting |
| ShredOS | GPL | Freely redistributable |
| Clonezilla Live | GPL | Freely redistributable |
| Ubuntu / Debian / Rocky / Alma ISOs | Various FOSS | Freely redistributable |
| RHEL ISOs (if applicable) | Subscription | Internal hosting under site subscription only |
| Dell DRM repo / iDRAC firmware / DSU | Dell EULA | Internal hosting for owned/entitled hardware; **never** in public repo |
| Other vendor firmware | Vendor EULA | Same treatment; document per vendor |
| Parted Magic (only if purchased) | Commercial (~$15/seat class) | Document license; SystemRescue covers most use |

### 5.6 Curation — Allowlist by Request

The library is **allowlist by request, not free-for-all**: a tenant needing an additional tool files a request; the platform team vets and adds it; the addition is a Forgejo commit. In practice nearly everything is approved — the point is that the library's contents are themselves an auditable, change-controlled artifact rather than an unmanaged file dump.

---

## 6. Egress Policy

**Deny all egress, with four enumerated exceptions.** This single posture addresses exfiltration (the enclave cannot stage data outward), command-and-control (a compromised BMC or VM has nowhere to phone home — most malware footholds become inert), and supply-chain drift (nothing inside can fetch unvetted software; the library is the only ingestion path).

| # | Source | Destination | Port/Proto | Purpose |
|---|---|---|---|---|
| 1 | Zabbix VM | Institutional SMTP smarthost | 25/587 TCP | Alert delivery (smarthost accepts by source IP) |
| 2 | Syslog aggregator | Enterprise log service | 6514 TCP (RFC 5425 syslog/TLS) | Audit stream |
| 3 | Keycloak VM | Institutional IdP | 443 TCP | Identity brokering during login |
| 4 | pfSense | Campus NTP | 123 UDP | Upstream time (pfSense serves NTP/DNS internally) |

Notably absent: any library-refresh egress (§5.4 — push-in), any BMC egress of any kind (NTP, DNS, syslog, and alert relay are all provided *inside* the enclave), and any general web access (§3.1 — half-screen doctrine). Ingress is limited to the two doors (§3) plus institutional monitoring of the doors themselves if required.

pfSense does not relay SMTP (no MTA in base pfSense); exception 1 is a plain firewall rule, not a service. If institutional mail later requires authenticated submission, a minimal Postfix satellite VM on the utility tier replaces the rule — but only add it if the direct rule proves insufficient.

---

## 7. Audit and Monitoring

### 7.1 Unified Syslog

Every component — pfSense, Keycloak, Guacamole, Forgejo, Zabbix, jump VMs (preconfigured in the golden image), media share — logs to the **syslog aggregator**, which forwards a single TLS-encrypted, destination-pinned stream to the enterprise log service. The enterprise copy is the **independence witness**; downstream filtering and SIEM routing are the enterprise's concern.

### 7.2 Local Retention

The aggregator retains **90–180 days locally**. The enterprise service filters before its SIEM, and its filter may drop exactly the line needed during the enclave's own incident: their copy is the witness, the local copy is the working record.

### 7.3 Session Recordings

Guacamole recordings are binary artifacts, **stored locally only** (dedicated volume; suggested retention: 1 year hot, then archive per institutional schedule). Only session *metadata* enters the syslog stream. Recordings are the nonrepudiable record of every human action taken against a BMC — the strongest single artifact in the AU narrative.

### 7.4 Zabbix

- Central instance; multi-tenant via **user groups granted permissions on host groups**. Each tenant sees only their own host group: dashboards, availability history, alerting on their BMCs.
- **Tenant permission model:** read-write on their host group for trigger tuning, maintenance windows, and alert routing — but **host lifecycle stays with the platform team** as part of controlled intake (§9.1). Self-service monitoring without collision with configuration management.
- **Credentials:** Zabbix holds read credentials for every BMC in every segment — the enclave's largest cross-tenant secret-holder. Two mitigations convert this from liability to evidence: (a) **Vault as the Zabbix secrets backend** — BMC monitoring credentials live in Vault, referenced at use-time, never at rest in the Zabbix DB; (b) a **dedicated read-only monitoring role** on each BMC (iDRAC role separation), so the credential can observe but never actuate.
- **Detective functions beyond availability:** a BMC that stops answering is an availability event; a **new MAC appearing in DHCP that is not in the reservation inventory is a detection event**; firmware versions polled from BMCs compared against the library's current catalog yields a "fleet currency" trigger — flaw-remediation evidence that generates itself.

---

## 8. Deliberate Risk Acceptances and Contingency

### 8.1 Risk Acceptance: Bidirectional Clipboard and SFTP

Guacamole clipboard and SFTP are bidirectional on all connections. **Rationale:** (1) the enclave contains **no research data** — tenant jump VMs reach BMCs, not data planes; the clipboard moves error messages and configuration text between a user and their own screen. (2) The realistic blocking scenario punishes exactly the wrong person: tenant sessions occur predominantly when in-band access is already broken — high-stress moments when a user urgently needs to move an error message into a search engine or an LLM. (3) Copy-out restrictions are defeated by trivially available screen-reading (an LLM reading the Guacamole half of the screen reproduces the text with zero effort), converting the control into pure friction with no security value. **Compensating control:** full session recording captures everything transiting the human channel. **Residual risk:** assessed low; accepted with this documented rationale. (Assessors respect a reasoned acceptance more than a checkbox control that a screenshot defeats.)

### 8.2 The Circularity Problem, Named

The enclave's control plane (Keycloak, Guacamole, Forgejo, Zabbix) runs on the Proxmox cluster — infrastructure the enclave exists to help manage. If the cluster is unhealthy, the tools used to reach its BMCs are down, and so is the IdP. Every management plane has a bootstrapping story; this one is layered explicitly:

### 8.3 Break-Glass Layers (outermost first)

1. **Operator mesh survives VM death:** Headscale/WireGuard terminates on pfSense, not on a VM.
2. **Hypervisor BMCs on an operator-only segment:** the Proxmox nodes' own BMCs (and switch management interfaces) live in a dedicated VLAN reachable only via the operator path — the "management of the management" tier.
3. **Local break-glass accounts** on Guacamole and pfSense — one documented local admin each, existing precisely because Keycloak may be unreachable. Credentials **escrowed with an organizationally independent custodian** (e.g., the enterprise IT credential vault) — independent escrow is stronger than self-escrow and doubles as the DR story.
4. **Physical console procedure** as the floor: documented, tested, in the runbook.

### 8.4 Availability Notes

CARP handles firewall failover; XML-RPC sync + DHCP failover preserve state; the QDevice preserves Proxmox quorum on single-node loss; utility VMs restart via Proxmox HA on the surviving node. The contingency section of the SSP (§10.9) narrates a full-cluster-loss recovery: physical console → single pfSense instance → operator mesh → restore utility tier from Proxmox backups (Forgejo holds all configuration as code).

---

## 9. Lifecycle Procedures

### 9.1 Tenant / Host Onboarding Checklist

1. Allocate VLAN + /26 (or /24 for a cluster); record in the subnet plan (Appendix A).
2. Configure switch ports (untagged tenant VLAN) for each BMC.
3. Create DHCP MAC reservations (inventory entry).
4. Run the **BMC intake hardening script** per host (§9.2); attach output to the intake record.
5. Create `enclave-tenant-<group>` in Keycloak; enroll the named individuals from the PI's request.
6. Create Guacamole connection group + connections (jump VM RDP/VNC, SSH targets).
7. Provision the jump VM from the golden image.
8. Create the Zabbix host group + hosts (read-only monitoring credential via Vault); grant the tenant user group permissions.
9. Create the tenant's Forgejo org (auto-mapped from the groups claim); seed the Ansible inventory skeleton.
10. Send the tenant the user guide (including the half-screen doctrine).

### 9.2 BMC Intake Hardening Baseline (scripted, output logged)

- Rotate default credentials (unique per BMC; recorded in Vault)
- Create the read-only monitoring role for Zabbix
- Disable IPMI-over-LAN where the BMC will only be driven via web/Redfish/SSH; where IPMI must remain, disable cipher suite 0
- Point NTP and syslog at enclave services; disable all other BMC egress features (SNMP trap targets, email alerts) or point them at enclave services
- Verify firmware currency against the library catalog; update from the internal mirror if behind
- Prefer and enable Redfish

### 9.3 Offboarding (inverse, as a checklist)

Remove Keycloak group members (or the group), Guacamole connections, Zabbix hosts + user-group grants, Forgejo org (archive), jump VM (archive/destroy), DHCP reservations, switch-port config, VLAN (last), and rotate any credentials the tenant held. A dissolved research group leaves no residue.

### 9.4 Periodic Attestations

- Operator mesh peer list — quarterly review against current authorized operators
- Tenant rosters — annual PI re-attestation, plus continuous detective reconciliation (§4.3)
- Firewall ruleset — annual review against this specification's egress table
- Library manifest reconciliation — continuous (§5.3)

---

## 10. System Security Plan Skeleton

Mapped to **NIST SP 800-171** (an 800-53 moderate crosswalk may be appended for audiences that prefer it). Section numbering below is the SSP's own.

### 10.1 System Identification and Description

Name, owner, operational status, system type: **management enclave / security support system**. Key framing sentence: *this system processes no research data; it carries administrative access to systems that may.*

### 10.2 System Boundary and Environment

The boundary diagram is the heart of the document. **In scope:** pfSense pair, Proxmox hosts (hypervisor substrate), the six utility VMs, tenant jump VMs, switch management-VLAN configuration, the BMCs as connected components, the Headscale termination. **Out of scope:** research servers' production interfaces and data planes. **Interconnections:** campus network (user ingress), institutional IdP, institutional smarthost, enterprise log service, campus NTP.

### 10.3 Security Categorization

Confidentiality of enclave *contents* is low (no research data). **Integrity and availability of the management plane are high**: compromise of the enclave is compromise-by-proxy of every connected research system; loss of the enclave removes out-of-band recovery capability precisely when it is needed. Categorize on the impact of the systems the enclave controls, not the data it carries.

### 10.4 Roles and Responsibilities

System owner; platform operators; tenant "designated technical contacts" and the scope of what they are entrusted with; PIs as access requesters/attesters; the enterprise log service as independent audit witness; the escrow custodian.

### 10.5 Data Flows (each diagrammed)

(a) tenant → Guacamole → jump VM / SSH → BMC; (b) operator → Headscale (pfSense) → any enclave segment including the hypervisor-BMC tier; (c) BMC → internal services (NTP/DNS via pfSense; monitoring poll inbound from Zabbix) → nothing else; (d) audit: components → syslog aggregator → enterprise log service. Plus the push-in library refresh flow.

### 10.6 Control Implementation by Family

| Family | Strongest evidence |
|---|---|
| **AC** (3.1.x) | Per-tenant VLAN + deny-by-default pfSense rules as *technical* enforcement of least privilege; the testable invariant "no tenant can reach another tenant's segment"; groups-claim authorization at every service; explicit named grants narrower than research-group membership; IdP-backstop deprovisioning layered under local group hygiene |
| **IA** (3.5.x) | OIDC → enclave Keycloak → institutional IdP with institutional MFA on the tenant path; SSO-backed, key-expiring operator identities; ePPN identifiers; no shared accounts except escrowed break-glass |
| **AU** (3.3.x) | Full session recording of every management action; unified TLS syslog to an independent enterprise witness; local working-copy retention; session metadata correlation; recordings as nonrepudiable evidence |
| **MA** (3.7.x) | **The system *is* the MA family's implementation.** 3.7.5 (multifactor for nonlocal maintenance) is satisfied for every connected research system by the enclave itself. Frame the enclave as a **control provider from which connected systems' SSPs inherit** — research groups' own compliance documents point here rather than re-deriving maintenance controls |
| **SC** (3.13.x) | Boundary protection with deny-all egress and four enumerated exceptions; IPMI protocol containment (RMCP+ never crosses the boundary; cipher 0 disabled; Redfish preferred); TLS on the audit stream; WireGuard on the operator path; CARP HA as availability protection |
| **CM** (3.4.x) | BMC intake hardening baseline (scripted, logged); pfSense configuration and golden-image builds under version control in Forgejo; controlled intake as the sole path onto the network; monthly jump-VM rebuilds from a version-controlled baseline; library allowlist curation with commit-recorded changes |
| **SI** (3.14.x) | Manifest↔share hash reconciliation; fleet firmware-currency triggers from Zabbix vs. library catalog; new-MAC detection; push-in refresh with outside-the-boundary human verification |
| **IR / CP** (3.6.x) | BMC credential-compromise playbook; layered break-glass down to physical console; full-cluster-loss recovery narrative; tenant-dissolution decommission checklist |

### 10.7 Risk Acceptances

§8.1 (bidirectional clipboard/SFTP), stated with rationale, compensating control, and residual-risk assessment. Any others accumulated during deployment.

### 10.8 Continuous Monitoring

The attestation schedule (§9.4), the detective controls (§4.3, §5.3, §7.4), and firmware currency as an ongoing metric.

### 10.9 Contingency and POA&M

The layered break-glass narrative (§8.3), the recovery walk-through (§8.4), and known gaps with remediation dates.

---

## Appendix A: Deployment Parameters

Every institution-specific value a deployer must supply. The Northwinds **Mudroom** column is the worked example; an institutional deployment reproduces this table with real values in its private repository.

| Parameter | Description | Northwinds Mudroom (example) |
|---|---|---|
| Deployment name | Human name for the deployment | Mudroom |
| Base domain | DNS zone for enclave services | `mudroom.northwinds.edu` |
| Institutional IdP | OIDC/SAML endpoint brokered by enclave Keycloak | `idp.northwinds.edu` |
| MFA source | Where multifactor is enforced | Institutional IdP (Duo) |
| SMTP smarthost | Destination for Zabbix alerts; auth basis | `smtp.northwinds.edu`, accepts by source IP |
| Enterprise log service | Syslog/TLS destination; custodian | `logs.northwinds.edu:6514`, Enterprise IT SOC |
| Campus NTP | pfSense upstream time | `ntp.northwinds.edu` |
| Escrow custodian | Holder of break-glass credentials | Enterprise IT credential vault |
| Subnet plan | Enclave supernet; per-tenant /26 map; cluster /24s; utility and operator VLANs | `10.60.0.0/16` (worked map in `northwinds/mudroom`) |
| VLAN plan | VLAN IDs per tenant/utility/operator segment | worked map in `northwinds/mudroom` |
| Certificate authority | TLS issuance for enclave services | Institutional ACME (InCommon) |
| Hypervisor platform | Substrate + HA specifics | 2× Proxmox + QDevice |
| Fleet vendors | Determines firmware shelves + tooling | Dell (primary), Supermicro |
| Distro set | Installation media shelf | Ubuntu LTS, Rocky, Debian |
| Research-group source of truth | Upstream for detective reconciliation (§4.3), if any | Globus Groups (Root Cellar / Northstar) |
| Recording retention | Session-recording schedule | 1 year hot, archive per institutional schedule |
| Log retention (local) | Aggregator working-copy window | 180 days |

---

## Appendix B: Decisions Ledger

The load-bearing decisions and why. (Full discussion history lives with the project.)

| # | Decision | Rationale (compressed) |
|---|---|---|
| 1 | Per-tenant VLAN, /26 groups, /24 clusters, MAC-reserved DHCP | Least privilege as network fact; reservation table doubles as inventory; no renumbering |
| 2 | Per-group jump VM behind Guacamole as the **sole tenant path** | A real browser inside the segment handles every BMC UI/KVM/virtual-media quirk; automation moves inside and gets recorded; eliminates the tenant-VPN use case |
| 3 | Guacamole session recording, mandatory | Nonrepudiable record of every management action; the AU family's centerpiece |
| 4 | No tenant VPN; **Headscale on pfSense for operators only** | Two identity-backed doors beat "everyone has a VPN option"; SSO + key expiry fix raw WireGuard's lifecycle problem; short attestable peer list |
| 5 | Dedicated enclave Keycloak; institutional IdP upstream; **no research-platform identity chain** | Boundary independence; the enclave must work during a research-platform incident; separate failure/incident domains |
| 6 | Local Keycloak groups authoritative; explicit named enrollment; research rosters demoted to **detective cross-check** | Management access is narrower than data access; roster-mirroring is privilege inflation; drift detection without dependency |
| 7 | One groups claim drives Guacamole + Forgejo + Zabbix | Single authorization source; zero per-service rosters |
| 8 | Forgejo over Gitea over GitLab CE | OIDC group→org mapping free (Premium-gated in GitLab); lightweight; community governance; Actions suffice for CI |
| 9 | Library split: Forgejo metadata + plain HTTP/NFS media share + hash reconciler | BMCs need dumb URLs; git stays lean; manifest↔share drift becomes a detection event |
| 10 | Zero egress; four enumerated exceptions; **push-in** library refresh with outside verification | Kills exfil, C2, and supply-chain drift at once; trivially auditable; human verification happens outside the boundary |
| 11 | Bidirectional clipboard/SFTP as documented risk acceptance | No research data in the boundary; blocking punishes the high-stress legitimate user; screen-reading defeats the control anyway; session recording compensates |
| 12 | Audit = one syslog/TLS stream to the enterprise witness; recordings local; **local retention kept** | Downstream offered custody — take it; their filter may drop your incident's key line, so keep the working copy |
| 13 | pfSense serves NTP/DNS; **BMC egress = zero** | All BMC dependencies satisfied internally; deny-all stays clean |
| 14 | Zabbix central; tenant RW on own host group minus host lifecycle; **Vault-backed read-only BMC creds** | Self-service monitoring without CM collision; the biggest secret-holder holds only read-only creds fetched at use-time |
| 15 | Separate small utility VMs; QDevice; CARP+XMLRPC+DHCP-failover | Blast radius; quorum on two nodes; stateful failover |
| 16 | Break-glass escrowed with independent custodian; hypervisor BMCs on operator-only tier; physical-console floor | Circularity named and answered in layers; independent escrow > self-escrow |
| 17 | Monthly golden-image jump-VM rebuilds; durable tenant material in Forgejo orgs | Anti-snowflake; CM evidence generates itself |
| 18 | Naming: Posternity (project) / per-institution deployment names / boring internal slugs | Reference-architecture layering mirrors the codebase split; collision-resistant; SSP cover pages stay professional |

---

## Appendix C: User Guide Fragments (tenant-facing language)

- **The half-screen doctrine:** *The management window is for managing; your local browser is for searching.* When troubleshooting, put your Guacamole session on one half of your screen and your own browser on the other. Copy error text out of the session freely — that's what it's for.
- **Where your stuff lives:** your jump VM is rebuilt monthly. Anything you want to keep — playbooks, inventories, notes — belongs in your group's Forgejo org, which is permanent, versioned, and pre-configured on the VM.
- **Need a tool we don't have?** File a library request. Almost everything is approved; the process exists so the environment stays curated, not to say no.
- **Reimaging a server?** Use the iPXE ISO from the library over virtual media and chainload from the deploy server — it is dramatically faster than mounting a full installer ISO through your BMC.

---

*Posternity is developed openly at `atmarx/posternity`. The Northwinds Mudroom reference deployment lives at `northwinds/mudroom`. Contributions of generalizable improvements are welcome; institution-specific logic belongs in your deployment repository.*
