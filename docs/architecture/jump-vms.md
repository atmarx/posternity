# Tenant jump VMs

One **low-resource, dedicated VM per tenant group**, inside the tenant's own VLAN. The jump VM is the universal client: a real browser on the management network reaches every BMC web UI, HTML5 KVM console, and virtual-media function natively — no reverse-proxy fragility, no vendor-console quirks.

- Built from a **golden image** (Proxmox template) whose build definition (cloud-init / Packer + Ansible) lives in Forgejo. **Rebuilt monthly** from the current baseline; jump VMs are semi-disposable by design, which discourages snowflake accumulation. Durable tenant material belongs in the tenant's [Forgejo org](../library/index.md), not on the VM.
- Baseline toolkit: `ipmitool`, `freeipmi`, `racadm`, Redfish tooling (DMTF `redfishtool`, vendor Redfish scripting collections, `python-redfish`), **Ansible** with `dellemc.openmanage`, `community.general.redfish_*`, and IPMI modules preinstalled, a pinned browser known to behave with the fleet's BMC consoles, and standard kit (openssh, tmux, curl, jq, rsync, tcpdump, minicom/screen for Serial-over-LAN, git preconfigured against the internal Forgejo).
- **Automation lives here.** Tenants run Ansible/Redfish/racadm from the jump VM via Guacamole SSH — which means automation sessions pass through session recording like everything else.
