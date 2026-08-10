# BMC intake hardening baseline

Scripted, output logged:

- Rotate default credentials (unique per BMC; recorded in Vault)
- Create the read-only monitoring role for Zabbix
- Disable IPMI-over-LAN where the BMC will only be driven via web/Redfish/SSH; where IPMI must remain, disable cipher suite 0
- Point NTP and syslog at enclave services; disable all other BMC egress features (SNMP trap targets, email alerts) or point them at enclave services
- Verify firmware currency against the library catalog; update from the internal mirror if behind
- Prefer and enable Redfish
