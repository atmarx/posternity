# Architecture overview

```text
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

The architecture is four layers, each with its own page:

- **[Network layer](network.md)** — per-tenant VLANs, MAC-reserved DHCP as inventory, deny-by-default inter-VLAN policy, IPMI protocol containment.
- **[Firewall / HA layer](firewall-ha.md)** — the pfSense CARP pair on a two-node Proxmox cluster, quorum, and internal DNS/NTP.
- **[Utility tier](utility-tier.md)** — the six small VMs that provide identity, access, library, monitoring, and audit services.
- **[Tenant jump VMs](jump-vms.md)** — the per-group universal client that makes every BMC quirk livable and puts automation inside session recording.
