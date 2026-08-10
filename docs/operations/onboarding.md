# Tenant / host onboarding

1. Allocate VLAN + /26 (or /24 for a cluster); record in the subnet plan ([deployment parameters](../reference/deployment-parameters.md)).
2. Configure switch ports (untagged tenant VLAN) for each BMC.
3. Create DHCP MAC reservations (inventory entry).
4. Run the [**BMC intake hardening script**](bmc-intake.md) per host; attach output to the intake record.
5. Create `enclave-tenant-<group>` in Keycloak; enroll the named individuals from the PI's request.
6. Create Guacamole connection group + connections (jump VM RDP/VNC, SSH targets).
7. Provision the jump VM from the golden image.
8. Create the Zabbix host group + hosts (read-only monitoring credential via Vault); grant the tenant user group permissions.
9. Create the tenant's Forgejo org (auto-mapped from the groups claim); seed the Ansible inventory skeleton.
10. Send the tenant the [user guide](../reference/user-guide.md) (including the half-screen doctrine).
