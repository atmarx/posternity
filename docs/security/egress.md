# Egress policy

**Deny all egress, with four enumerated exceptions.** This single posture addresses exfiltration (the enclave cannot stage data outward), command-and-control (a compromised BMC or VM has nowhere to phone home — most malware footholds become inert), and supply-chain drift (nothing inside can fetch unvetted software; the library is the only ingestion path).

| # | Source | Destination | Port/Proto | Purpose |
|---|---|---|---|---|
| 1 | Zabbix VM | Institutional SMTP smarthost | 25/587 TCP | Alert delivery (smarthost accepts by source IP) |
| 2 | Syslog aggregator | Enterprise log service | 6514 TCP (RFC 5425 syslog/TLS) | Audit stream |
| 3 | Keycloak VM | Institutional IdP | 443 TCP | Identity brokering during login |
| 4 | pfSense | Campus NTP | 123 UDP | Upstream time (pfSense serves NTP/DNS internally) |

Notably absent: any library-refresh egress ([push-in](../library/curation.md)), any BMC egress of any kind (NTP, DNS, syslog, and alert relay are all provided *inside* the enclave), and any general web access ([half-screen doctrine](../access/doors.md#tenant-path-guacamole)). Ingress is limited to the [two doors](../access/doors.md) plus institutional monitoring of the doors themselves if required.

pfSense does not relay SMTP (no MTA in base pfSense); exception 1 is a plain firewall rule, not a service. If institutional mail later requires authenticated submission, a minimal Postfix satellite VM on the utility tier replaces the rule — but only add it if the direct rule proves insufficient.
