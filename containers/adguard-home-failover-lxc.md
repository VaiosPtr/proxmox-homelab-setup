# AdGuard Home Failover — Second Instance (Proxmox Homelab)

Cheatsheet for adding a second AdGuard Home container as a DNS failover, so ad-blocking/filtering survives a reboot or outage of the primary instance.

## Overview

- **Primary AdGuard:** existing container, DNS Server 1 in pfSense
- **Secondary AdGuard:** cloned/restored copy, DNS Server 2 in pfSense
- Goal: if the primary goes down, devices fall back to the secondary (still filtered) instead of an unfiltered public resolver

---

## 1. Create the second container

Two paths depending on whether the primary is running.

### Option A — Container is stopped

1. Proxmox UI → right-click the existing AdGuard container → **Clone**.
2. Set a new **VM ID** and **Hostname**.
3. Click **Clone**.

### Option B — Container is running

1. Right-click the AdGuard container → **Backup** → **Backup now**.
   - Storage: `local` (or preferred backup storage)
   - Mode: `Snapshot`
   - Click **Backup**.
2. Once the backup finishes, go to **Storage (local) → Backups**.
3. Select the backup just created → **Restore**.
   - Set a new **CT ID**.
   - **Uncheck** "Start after restore".
   - Click **Restore**.
4. Select the new container → **Network** → change the IP to a new, unused address on your LAN.

---

## 2. Start the new container

Start it once the IP has been changed and confirmed unique.

---

## 3. Point pfSense at the second instance

`Services → DHCP Server → LAN → DNS Servers`

```
DNS Server 1: <primary-adguard-ip>
DNS Server 2: <secondary-adguard-ip>
```

Save → Apply Changes.

---

## Notes

- Both instances filter independently — if you add blocklists or custom rules on the primary, replicate them on the secondary manually (or re-clone periodically) so filtering stays consistent.
- **DNS1/DNS2 behavior is OS-dependent, not universal:**
  - **Sequential resolvers** (classic behavior, still common on Linux/glibc): query DNS1 first, only try DNS2 after DNS1 times out or fails. True failover.
  - **Parallel resolvers** (Windows since 8, via Smart Multi-Homed Name Resolution; macOS/iOS often too): query DNS1 and DNS2 **simultaneously** and use whichever answers first — even while DNS1 is perfectly healthy. This means some queries can bypass filtering unpredictably if DNS2 happens to respond faster.
