# Curation and refresh

## Allowlist by request

The library is **allowlist by request, not free-for-all**: a tenant needing an additional tool files a request; the platform team vets and adds it; the addition is a Forgejo commit. In practice nearly everything is approved — the point is that the library's contents are themselves an auditable, change-controlled artifact rather than an unmanaged file dump.

## Refresh — push-in, never pull-from-inside

Operators download ISOs and firmware on workstations **outside the boundary**, verify checksums against vendor-published values **there**, then push the binary to the share and the manifest commit to Forgejo over the [operator path](../access/doors.md#operator-path-headscale). Who, what, and which hash are recorded in the commit. The verification step happens outside, by a human, before anything crosses the boundary — a stronger supply-chain posture than allowlisted vendor-domain egress.

## Manifest ↔ share reconciliation

A scheduled job (Forgejo Actions or cron on the media VM) hashes everything on the share against the manifests in git. **An unmanifested file or a hash mismatch is a detection event.** The share cannot silently diverge from the record; the reconciler is itself an integrity control (SI/CM evidence).
