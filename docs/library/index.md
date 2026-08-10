# Software and media library

The enclave provides all supported tooling so tenants never need to bring their own — the mechanism that makes [zero egress](../security/egress.md) livable. The library is split along its natural seam: **Forgejo holds the metadata** (everything text, everything diffable) and the [**media share holds the binaries**](media-share.md) (dumb endpoints for dumb clients).

## Forgejo — metadata

- Checksum manifests for every binary on the media share
- [`LICENSES.md`](licenses.md)
- Firmware catalog definitions (e.g., Dell Repository Manager catalogs)
- [BMC intake hardening script](../operations/bmc-intake.md), SoL helpers, support-bundle collectors
- Runbooks: reimage (iPXE and full-ISO paths), disk sanitization, "my BMC is unreachable" triage
- [Allowlist request records](curation.md)
- Golden image build definitions; per-group Ansible inventory skeletons

**Why Forgejo:** OIDC group-claim → org/team mapping is in the free product (GitLab CE requires Premium for group sync, recreating the roster-drift problem the architecture exists to avoid); it is lightweight; Forgejo Actions covers the modest CI needs (image builds, manifest verification, playbook lint); and it is the community-governed hard fork of Gitea — a defensible provenance choice for a published reference architecture. Structure: a **`library` org** (platform writes, all tenants read) and a **per-tenant org** mapped from each tenant's Keycloak group, giving tenant automation a version-controlled home that survives monthly jump-VM rebuilds.
