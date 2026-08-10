# Audit and monitoring

## Unified syslog

Every component — pfSense, Keycloak, Guacamole, Forgejo, Zabbix, jump VMs (preconfigured in the golden image), media share — logs to the **syslog aggregator**, which forwards a single TLS-encrypted, destination-pinned stream to the enterprise log service. The enterprise copy is the **independence witness**; downstream filtering and SIEM routing are the enterprise's concern.

## Local retention

The aggregator retains **90–180 days locally**. The enterprise service filters before its SIEM, and its filter may drop exactly the line needed during the enclave's own incident: their copy is the witness, the local copy is the working record.

## Session recordings

Guacamole recordings are binary artifacts, **stored locally only** (dedicated volume; suggested retention: 1 year hot, then archive per institutional schedule). Only session *metadata* enters the syslog stream. Recordings are the nonrepudiable record of every human action taken against a BMC — the strongest single artifact in the AU narrative.

## Zabbix

- Central instance; multi-tenant via **user groups granted permissions on host groups**. Each tenant sees only their own host group: dashboards, availability history, alerting on their BMCs.
- **Tenant permission model:** read-write on their host group for trigger tuning, maintenance windows, and alert routing — but **host lifecycle stays with the platform team** as part of [controlled intake](../operations/onboarding.md). Self-service monitoring without collision with configuration management.
- **Credentials:** Zabbix holds read credentials for every BMC in every segment — the enclave's largest cross-tenant secret-holder. Two mitigations convert this from liability to evidence: (a) **Vault as the Zabbix secrets backend** — BMC monitoring credentials live in Vault, referenced at use-time, never at rest in the Zabbix DB; (b) a **dedicated read-only monitoring role** on each BMC (iDRAC role separation), so the credential can observe but never actuate.
- **Detective functions beyond availability:** a BMC that stops answering is an availability event; a **new MAC appearing in DHCP that is not in the reservation inventory is a detection event**; firmware versions polled from BMCs compared against the library's current catalog yields a "fleet currency" trigger — flaw-remediation evidence that generates itself.
