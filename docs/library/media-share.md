# Media share

Binaries never enter git (even LFS): the primary consumers are **BMCs mounting virtual media and pulling firmware catalogs**, which want plain unauthenticated HTTP/NFS URLs. A boring directory tree:

```text
/isos/
  systemrescue/  gparted/  memtest86plus/  memtest86-free/
  shredos/  clonezilla/  ipxe/
  ubuntu-lts/  rocky/  alma/  debian/    (current + previous, incl. netboot minis)
  vendor-live/
/firmware/
  dell/<model>/       (DRM offline repository + standalone packages + DSU)
  supermicro/<model>/  hpe/<model>/  ...
  lvfs-mirror/        (optional, where fleet hardware is LVFS-covered)
```

Notable inclusions:

- **SystemRescue** — the free rescue-environment workhorse (GParted, ddrescue, testdisk, network tools)
- **ShredOS** — nwipe-based sanitization boot image; the data-destruction SOP points here
- **The iPXE bootstrap ISO** — arguably the most important item: mount the tiny ISO via virtual media, chainload everything else over HTTP from the deploy server. Recommended reimage path (virtual-media performance becomes irrelevant); full ISOs are the fallback.
- **Dell Repository Manager offline repo** — iDRAC "update from network share" pointed at the internal mirror: BMC firmware updates with zero BMC egress.
