# Firewall / HA layer

- Two **pfSense** instances, virtualized on a two-node Proxmox cluster, one instance affined per node, **CARP** for HA with **XML-RPC configuration sync** and **DHCP failover** (a CARP failover that loses reservation state must never gamble a BMC's lease).
- Virtualization notes (build-doc material): Proxmox bridges require MAC-spoofing/promiscuous allowances for the CARP VIP to float; multicast must pass on relevant bridges; disable hardware checksum offload in the pfSense guests.
- **Quorum:** a two-node Proxmox cluster has none. A **corosync QDevice** on any off-cluster host (utility NAS, single-board computer) is required so a single node loss does not strand the surviving node's ability to manage VMs.
- pfSense provides **DNS and NTP** to the entire enclave and is the only component that syncs upstream (campus NTP). BMCs and VMs never leave the enclave for time or names.
- pfSense terminates the [**operator mesh VPN**](../access/doors.md#operator-path-headscale) natively, so the operator door survives the death of every VM on the cluster.
