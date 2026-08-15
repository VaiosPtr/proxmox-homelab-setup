# AdGuard Home Setup — Proxmox Homelab

Cheatsheet for deploying AdGuard Home as an LXC container on Proxmox VE and wiring it up as the network-wide DNS resolver behind pfSense.

## Stack

- **Host:** Proxmox VE
- **Container:** LXC (Debian 13, via community-scripts helper)
- **Router/Firewall:** pfSense
- **ISP:** Starlink (CGNAT — noted for context, not relevant to this guide)

---

## 1. Fix host IPv6 issues (if template downloads hang)

If `pveam update` or template downloads hang indefinitely, the host is likely trying to resolve over a broken IPv6 route. Confirm with:

```bash
ping -4 -c 3 download.proxmox.com   # should succeed
ping -c 3 download.proxmox.com      # fails if IPv6 is broken
```

Fix by preferring IPv4 system-wide:

```bash
nano /etc/gai.conf
```

Uncomment this line:

```
precedence ::ffff:0:0/96  100
```

Save and retry — no reboot required.

---

## 2. Install AdGuard Home (LXC)

Run on the **Proxmox host shell**:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/adguard.sh)"
```

Choose **Advanced** to control networking:

| Setting | Recommendation |
|---|---|
| OS | Debian 13 |
| Disk | 2–4GB (AdGuard itself is lightweight) |
| Static IP | Yes — DNS servers need a stable address |
| IPv6 | None, unless your LAN actually uses IPv6 |
| SSH keys | Optional — mostly manage via web UI |
| Nesting | No — not needed for a native install |
| APT-Cacher | No, unless you already run one |
| Filesystem mounts | No — AdGuard doesn't need network mounts |

If prompted to upgrade the host's LXC stack, accept — it's safe and doesn't touch existing containers/VMs.

---

## 3. First-run setup wizard

Visit `http://<container-ip>` (port shown at end of install; commonly `:80` or `:3000`).

1. **Admin Web Interface** — leave on all interfaces, default port.
2. **DNS server** — all interfaces, port `53`. If it warns the port is in use, `systemd-resolved` needs to be disabled on the container.
3. **Create admin account** — set a strong username/password (no easy reset flow).
4. **Static IP confirmation** — confirm the detected IP.
5. Finish → lands on the dashboard.

---

## 4. Configure upstream DNS

**Settings → DNS settings → Upstream DNS servers**

Example (Quad9 DNS10, no ECS, malware blocking):

```
https://dns10.quad9.net/dns-query
```

Optional secondary:

```
https://dns.cloudflare.com/dns-query
```

Click **Test upstreams** before saving.

---

## 5. Add blocklists

**Settings → Filters → DNS blocklists**

Keep the default AdGuard list enabled, and layer on additional lists as desired, e.g. OISD Big:

- Name: `OISD Big`
- URL: `https://big.oisd.nl`

AdGuard deduplicates overlapping entries automatically — running multiple lists is safe.

---

## 6. Point pfSense at AdGuard

**Disable pfSense's built-in resolver** (avoids port 53 conflict):

`Services → DNS Resolver` (or `DNS Forwarder`) → uncheck **Enable** → Save.

**Set DHCP to hand out AdGuard's IP:**

`Services → DHCP Server → LAN → DNS Servers`

```
DNS Server 1: <adguard-container-ip>
DNS Server 2: 1.1.1.1   (fallback if AdGuard is down)
```

Save → Apply Changes.

**(Optional) Update pfSense's own system DNS:**

`System → General Setup → DNS Server Settings` → set first entry to the AdGuard IP.

---

## 7. Update the Proxmox host's own DNS

Containers/VMs set to **"use host settings"** inherit DNS from the Proxmox host, not from pfSense DHCP.

`Datacenter → node → System → DNS`

```
DNS server 1: <adguard-container-ip>
DNS server 2: 1.1.1.1
```

This updates `/etc/resolv.conf` on the host immediately — no restart needed for the host itself.

---

## 8. Update static-IP devices manually

Static IP devices never pick up DHCP changes. Update DNS on each individually:

- **Linux (netplan):** edit `/etc/netplan/*.yaml` → `nameservers: addresses:` → `sudo netplan apply`
- **Linux (systemd-resolved):** `sudo resolvectl dns <interface> <adguard-ip>`
- **Windows:** Network adapter → IPv4 properties → DNS servers
- **LXC using "host settings":** inherits from Proxmox host automatically, but the container's cached `/etc/resolv.conf` may need a reboot to pick up host changes:
  ```bash
  pct reboot <vmid>
  ```

---

## 9. Verify it's working

1. Force a DHCP renewal on a test device (toggle Wi-Fi, or `ipconfig /release && ipconfig /renew`).
2. Open AdGuard's **Query Log** — browse on the test device, confirm entries appear.
3. Test blocking with an ad-test page (e.g. `d3ward.github.io/toolz/adblock.html`).

---

## Notes

- DNS1/DNS2 fallback behavior: most devices only fall back to DNS2 if DNS1 is fully unreachable, not just slow — so AdGuard being down is the main scenario where 1.1.1.1 kicks in.
- Quad9 DNS10 = malware/phishing blocking **without** ECS (slightly more private, marginally less optimal CDN routing).
- Restarting/rebooting AdGuard's LXC does not affect pfSense or other containers — only devices lose filtering temporarily until it's back up (falls back to secondary DNS).
