# Identity and authorization

## Identity

- A **dedicated enclave Keycloak** — deliberately *not* shared with any research data platform — brokers the **institutional IdP** as its sole upstream, with a local realm existing only for break-glass and development. A research-platform identity incident does not automatically become a management-plane incident, and vice versa.
- **ePPN** (eduPersonPrincipalName) is the stable identifier carried in tokens.
- The research data platform's identity chain (e.g., federated research IdPs, Globus Auth) does **not** appear anywhere in this system. The enclave must function precisely when the research platform is mid-incident.
- **Deprovisioning backstop:** because authentication brokers through the institutional IdP, separation from the institution kills login automatically even before any local group is touched. Local group removal is therefore about *timeliness and least privilege*, not about whether departed users retain access at all. State this layering explicitly in the AC narrative.

## Authorization

- **Enclave Keycloak groups are authoritative and local**: `enclave-tenant-<group>`, `enclave-operators`, `enclave-admins`.
- Membership is an **explicit, requested, named grant**: the PI requests access for named individuals; the platform team enrolls them via the onboarding procedure; the grant is a recorded artifact. Management-plane access is deliberately **narrower than research-group membership** — everyone in a lab touches the lab's data; only the PI and one or two designated technical people should hold BMC access. Auto-mirroring the research roster into the management plane would be privilege inflation by automation.
- **One groups claim drives every enforcement point:** Guacamole connection-group visibility, Forgejo org/team mapping, Zabbix user-group → host-group permissions. No per-service rosters exist.

## Detective reconciliation (optional module)

Where the institution operates a research-platform group source of truth (e.g., Globus Groups), a periodic **read-only** job compares each `enclave-tenant-<group>` roster against the corresponding research group and flags enclave members who are no longer in the research group ("left the lab, still holds BMC access — review"). Failure-tolerant: if the upstream is unreachable, the check skips and alerts; nothing about access changes. Drift detection without dependency.
