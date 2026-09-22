# TurnKey Fileserver — Proxmox Homelab

Cheatsheet for deploying the TurnKey Linux Fileserver LXC template on Proxmox VE, for Samba-based network file sharing.

## Stack

- **Host:** Proxmox VE
- **Container:** LXC, built from the official `turnkey-fileserver` template
- **File sharing protocol:** Samba (SMB), accessible from Windows via File Explorer
- **Admin UI:** Webmin, port 12321

---

## 1. Download the TurnKey Fileserver template

`Proxmox UI → local storage (on your node) → CT Templates → Templates`

Search for **`turnkey-fileserver`** and download it — this is the official, pre-built TurnKey Linux image with Samba already configured.

---

## 2. Create the LXC container

`Create CT` (top right)

- **Hostname:** e.g. `nas`
- **Password:** root password for the container
- **Template:** select the `turnkey-fileserver` template just downloaded
- **Disk:** size according to how much storage the fileserver needs
- **CPU / Memory:** sized to your homelab's budget
- **Network:** static IP on the pfSense LAN subnet, e.g. `10.11.12.201/24`, Gateway = pfSense's LAN IP

---

## 3. Run the TurnKey first-boot wizard

Start the container, then open its **Console** from the Proxmox UI. TurnKey's blue setup wizard walks through:

1. **Root password** — for Linux system administration.
2. **Samba password** — separate from root, used for file share access from other computers.
3. **Skip/Cancel** the TurnKey Hub API key and Cloud Backup prompts — not needed for a homelab setup.
4. Let it apply security updates and reboot.

---

## 4. Access and configure

Once boot completes, the console displays the relevant addresses:

- **Webmin (Web UI)** — for creating users and shared folders: `https://10.11.12.201:12321`
- **File access from Windows** — just type the IP directly into File Explorer's address bar: `\\10.11.12.201`

---

## Notes

- The Samba password is independent from the root password — set it deliberately, don't assume it matches.
- Webmin's self-signed certificate will trigger a browser warning on first visit — expected for a local admin panel, safe to proceed for homelab use.
- Referenced from the AdGuard Home setup as the static-IP file server whose DNS needed manual updating after the AdGuard DNS rollout (it doesn't pick up DHCP-pushed DNS changes).
