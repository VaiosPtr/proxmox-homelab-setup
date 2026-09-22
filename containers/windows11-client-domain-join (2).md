# Windows 11 Client VM — Domain Join (Proxmox Homelab)

Cheatsheet for provisioning a Windows 11 Enterprise (evaluation) client VM on Proxmox and joining it to the `lab.local` Active Directory domain, for practice with domain-joined client administration.

## Stack

- **Host:** Proxmox VE
- **Guest OS:** Windows 11 Enterprise (Evaluation, 90-day) — General Availability Channel, not LTSC
- **Domain:** `lab.local` (NetBIOS `LAB`), served by the Windows Server 2022 Domain Controller
- **Disk/Network:** VirtIO SCSI + VirtIO NIC

---

## 1. Download the ISO

[Microsoft Evaluation Center — Windows 11 Enterprise](https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise) — free, 90-day evaluation.

**Enterprise over Pro:** Enterprise supports the full range of Group Policy features (advanced GPOs, BitLocker management, AppLocker) that Pro restricts — more realistic for domain administration practice.

**General Availability Channel over LTSC:** LTSC is a stripped-down, minimal-update channel meant for locked-down devices (ATMs, medical/industrial systems) — missing standard apps and feature updates. GA Channel matches what real corporate environments actually run.

Upload the ISO to Proxmox the same way as the Windows Server ISO (`Storage → ISO Images → Upload`, or `wget` via SSH into `/var/lib/vz/template/iso/`).

---

## 2. Create the VM

| Tab | Setting |
|---|---|
| OS | Select the Windows 11 ISO; attach the VirtIO drivers ISO as well |
| System | Machine = `q35`, BIOS = `OVMF (UEFI)` |
| Disk | Bus = `VirtIO SCSI`, size 20GB is enough for a test client |
| CPU | 2 cores |
| Memory | **4096MB minimum — Windows 11 Setup hard-requires at least 4GB RAM**, unlike Server Core which ran fine on 3GB. Ballooning min ~2048MB (desktop OS needs more idle baseline than Server Core). |
| Network | Model = `VirtIO`, bridge = `vmbr0` |

**Note:** Windows 11 Setup will refuse to proceed ("This PC doesn't currently meet Windows 11 system requirements — There must be at least 4 GB of system memory") if Memory (max) is set below 4096MB. If you hit this mid-setup with no QEMU Guest Agent installed yet, use **Stop** (not Shutdown) to power off the VM before adjusting memory — Shutdown depends on the guest agent, which isn't installed yet at this stage.

---

## 3. Install Windows, load VirtIO disk driver

Same pattern as the Server VM: at the "Select location to install Windows 11" screen, click **Load driver**, browse to:

```
vioscsi\w11\amd64
```

(fall back to `w10\amd64` if no `w11` folder exists in your VirtIO ISO build — drivers are usually interchangeable). You may see two identical-looking entries in the driver list — either works.

---

## 4. Skip Microsoft account — force a local account

Windows 11 Setup pushes hard toward a Microsoft account during OOBE. To get the classic local-account flow instead, **disconnect network access before reaching the "Let's connect you to a network" screen**:

`VM → Hardware → Network Device → Disconnect`

With no network detected, Setup automatically falls back to local account creation ("Who's going to use this device?"). Reconnect the network device after finishing OOBE.

---

## 5. Install VirtIO drivers via Device Manager (GUI, not PowerShell)

Unlike the Server Core VM, this is a full desktop — use Device Manager instead of `pnputil`.

Under **Other devices**, you'll see the same three unrecognized devices seen on the Server VM:

| Device Manager entry | Driver folder |
|---|---|
| Ethernet Controller | `NetKVM\w11\amd64` |
| PCI Simple Communications Controller | `vioserial\w11\amd64` |
| PCI Device | `Balloon\w11\amd64` |

For each: right-click → **Update driver** → **Browse my computer for drivers** → point to the folder above → Next.

---

## 6. Set static IP and DNS (pointing at the Domain Controller)

Network adapter → IPv4 properties:

```
IP address:       <client-ip>        (pick an unused address in your LAN)
Subnet mask:       255.255.255.0
Default gateway:   <gateway-ip>        (pfSense LAN IP)
Preferred DNS:     <dc-ip>        (Domain Controller — required for AD SRV record lookups)
Alternate DNS:     <adguard-ip>        (AdGuard, as fallback only)
```

**Important:** DNS must point at the Domain Controller (not directly at AdGuard) for domain-join to succeed — AD relies on DNS SRV records that only the AD-integrated DNS server provides. AdGuard as the *primary* DNS here would break domain discovery.

---

## 7. Join the domain

`Right-click Start → System → Domain or workgroup → Rename this PC (advanced)` → **Change...** → select **Domain**, enter `lab.local`.

When prompted for credentials, either a Domain Admin or a regular domain user account works:

```
Username: LAB\<domain-admin>    (or LAB\testuser)
Password: (account password)
```

**Note:** Domain Admin rights are *not* required for this by default. Active Directory's `ms-DS-MachineAccountQuota` attribute defaults to **10** — any authenticated domain user can join up to 10 computers to the domain out of the box. This only requires elevated/delegated rights if an admin has deliberately lowered that quota to 0 (a common production hardening step, but not the default).

Confirm the "Welcome to the lab.local domain" message, then restart when prompted.

---

## 8. Log in with a domain account

After restart, the login screen offers both the local account (created in step 4) and domain accounts.

**Always use the full domain-prefixed username** — a bare username (`<domain-admin>`) gets interpreted as a *local* account on the client and fails with "the credentials that were used did not work":

```
LAB\<domain-admin>       (domain admin)
LAB\testuser          (regular domain user, for permission-scoped testing)
```

---

## Notes

- The 90-day evaluation license is for testing/practice only.
- Create a non-admin test account (`testuser`) in AD alongside the admin account — logging into the client with a regular user is a more realistic day-to-day scenario than always using Domain Admin credentials.
- RDP into this client also requires the `LAB\` prefix on the username, same rule as domain login — bare usernames are always interpreted as local accounts by RDP clients.
