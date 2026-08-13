# pfSense VM on Proxmox — Setup Guide
*Includes ISP router passthrough (bridge mode) timing*

## 1. Add a Second NIC
- Proxmox: **System → Network → Create → Linux Bridge**
- Under **Bridge ports**, type your 2nd NIC's name (e.g. `enp1s0`). Leave everything else blank.

## 2. Get the pfSense ISO

### Option A — Upload from your PC
1. Download pfSense and extract it (Proxmox needs a `.iso`, not `.iso.gz`).
2. In Proxmox: **pve (node) → local → ISO Images → Upload**. Select the file and upload.

### Option B — Download directly on Proxmox
1. Go to **pve → local → ISO Images → Download from URL**.
2. Right-click the pfSense download link on their site, copy the link, paste it into the URL field, and click Download.
3. Extract it via the Shell console:
   ```bash
   cd /var/lib/vz/template/iso/
   gunzip pfsense*.iso.gz
   ```

## 3. VM Creation

| Setting | Value |
|---|---|
| **OS** | Select the uploaded ISO. Guest OS type: Other. |
| **System** | Machine type `q35`, BIOS: Default (SeaBIOS). |
| **Disks** | 32 GB (VirtIO Block or SCSI). |
| **CPU** | 2 cores, type `host`. |
| **Memory** | 2048–4096 MB — disable ballooning. |
| **Network (at creation)** | Bridge `vmbr0` (WAN), Model: VirtIO (paravirtualized). |

After the VM is created, add the second NIC: **VM → Hardware → Add → Network Device → Bridge `vmbr1` (LAN)**, Model: VirtIO (paravirtualized).

## 4. Install pfSense
1. Start the VM and open the Console.
2. Accept terms → **Install pfSense**.
3. Partitioning: Auto (ZFS) or Auto (UFS).
4. When finished, select **Reboot** — then unmount the ISO under the VM's Hardware options so it doesn't boot the installer again.

## 5. Assign Interfaces
On reboot, pfSense prompts you to set up interfaces in the console:
- Set up VLANs now? → `n`
- WAN interface → `vtnet0` (maps to `vmbr0`)
- LAN interface → `vtnet1` (maps to `vmbr1`)
- Press Enter to apply.

## 6. Web GUI — First-Time Setup
1. Connect a laptop directly to physical NIC 2 (the LAN port).
2. Browse to `https://192.168.1.1`
3. Log in with default credentials — Username: `admin`, Password: `pfsense`
4. Complete the Setup Wizard: hostname/domain, WAN connection type (DHCP/PPPoE), and change the default admin password.

## 7. Autostart
In the VM's **Options** tab, enable **Start at boot** and set **Start/Shutdown order** to `1`, so internet comes back automatically after a power loss or reboot.

## 8. ISP Router Passthrough (Bridge Mode) — Timing

> **Do this LAST — only after pfSense is fully built and working.**

Build the entire pfSense VM first (Sections 1–7) while the ISP router is still in its normal, non-bridge mode. pfSense's WAN port doesn't need to be doing anything special yet — if it's plugged in, it just grabs an IP from the ISP router like any other device. That's fine.

Only once pfSense is fully installed, has interfaces assigned, and the LAN GUI is reachable and configured, do the switch:

1. Plug pfSense's WAN NIC into the ISP router.
2. Log into the ISP router and switch it to bridge / passthrough mode.
3. Reboot the ISP router (most models require this for bridge mode to apply).
4. On pfSense, renew the WAN interface (or reboot the VM) so it picks up the real public IP.

> **Note:** Doing this out of order causes problems either way — too early, and no device on the network is doing NAT/DHCP, so everything drops offline. Too late (leaving the ISP router in normal mode after pfSense is live) causes double NAT, which breaks port forwarding, VPNs, and some game/VoIP traffic.

Doing the switch last also means that if pfSense is misconfigured, you can just flip the ISP router back to normal mode and be instantly back online while you troubleshoot.

*Also confirm in advance that your specific ISP router model supports true bridge/passthrough mode — some budget ISP-supplied gateways don't, and you may need it swapped for a modem-only unit.*
