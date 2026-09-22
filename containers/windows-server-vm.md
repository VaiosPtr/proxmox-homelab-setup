# Windows Server 2022 VM — Proxmox Homelab

Cheatsheet for provisioning a Windows Server 2022 VM on Proxmox VE with VirtIO drivers for optimal disk/network performance, plus QEMU Guest Agent for clean shutdowns and IP reporting.

## Stack

- **Host:** Proxmox VE
- **Guest OS:** Windows Server 2022 (Evaluation, 180-day)
- **Disk/Network:** VirtIO SCSI + VirtIO NIC (paravirtualized drivers)
- **Machine type:** q35 + OVMF (UEFI)

---

## 1. Download the required ISOs

**Windows Server 2022:**
[Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022) — free, 180-day evaluation, ideal for homelab/practice use.

**VirtIO drivers:**
[virtio-win-pkg-scripts](https://github.com/virtio-win/virtio-win-pkg-scripts) — grab the **stable** `virtio-win.iso`, not the latest/dev build.

Windows has no native VirtIO storage or network drivers — without this ISO, the installer won't even see the VirtIO disk during setup.

---

## 2. Upload both ISOs to Proxmox

**Via UI:**
`Storage (e.g. local) → ISO Images → Upload`

**Via SSH (faster if downloading directly on the host):**
```bash
cd /var/lib/vz/template/iso/
wget <direct-download-link>
```

---

## 3. Create the VM

`Proxmox UI → Create VM`

| Tab | Setting |
|---|---|
| General | VM ID + name (e.g. `winserver2022`) |
| OS | Select the Windows Server ISO; Guest OS type `Microsoft Windows`, version `Windows 10/11/2022` |
| System | Machine = `q35`, BIOS = `OVMF (UEFI)` — add an EFI disk when prompted |
| Disk | Bus = `VirtIO SCSI`, size 60–80GB, storage `local-lvm` |
| CPU | 2+ cores, type = `host` for best performance |
| Memory | 4096MB minimum, 8192MB if running AD DS or other services later |
| Network | Model = `VirtIO (paravirtualized)`, bridge = `vmbr0` (or wherever your other VMs live) |

**Notes:**
- `CPU type = host` gives better performance but makes the VM less portable to different host hardware — fine for a permanent homelab node.
- If the VM fails to boot with OVMF, check **Secure Boot** under the VM's EFI settings — disable it if unsigned drivers get blocked.

---

## 4. Attach the VirtIO drivers as a second CD/DVD drive

After VM creation:
`VM → Hardware → Add → CD/DVD Drive` → select the `virtio-win.iso`

You'll now have two CD/DVD drives: one with the Windows Server ISO, one with VirtIO drivers.

---

## 5. Install Windows Server

Start the VM and proceed through Windows Setup. Choose **Custom install** (not Upgrade — there's no existing OS to upgrade from on a fresh VM).

**Pick the edition:** choose **Standard Evaluation** (no "Desktop Experience") if you're RAM-constrained — this is Server Core, CLI-only. It runs comfortably on 3-4GB versus 4GB+ needed for the Desktop Experience GUI build. Datacenter edition isn't needed unless you specifically want Storage Spaces Direct/Shielded VMs features.

**At the disk selection screen** (shows "We couldn't find any drives" — expected):
1. Click **Load driver**.
2. Browse the VirtIO CD → `vioscsi` → your Windows version folder (e.g. `2k22`) → `amd64`.
3. Select the driver — the VirtIO SCSI disk should now appear. Continue installation normally.

---

## 6. Fix networking (Server Core, PowerShell)

After first boot, `ipconfig` will show no IP — the NetKVM driver isn't built into Windows.

`wmic` is deprecated on newer Server 2022 builds — use PowerShell instead to find the VirtIO CD's drive letter:

```powershell
Get-Volume
```

or

```powershell
Get-PSDrive -PSProvider FileSystem
```

Once you've identified the CD drive letter, confirm its contents and install the network driver:

```powershell
dir <drive-letter>:\
pnputil /add-driver <drive-letter>:\NetKVM\2k22\amd64\*.inf /install
ipconfig
```

You should now see an IPv4 address.

---

## 7. Install QEMU Guest Agent

Enables clean shutdowns/reboots from the Proxmox UI and reports the VM's IP address in the summary tab.

**In Proxmox first:**
`VM → Options → QEMU Guest Agent → Enable` — requires a full **Stop → Start** of the VM to take effect (a plain in-guest reboot is not enough; the virtio-serial channel only initializes on a cold start).

**Inside the Windows guest (PowerShell):**

```powershell
dir <drive-letter>:\guest-agent
msiexec /i <drive-letter>:\guest-agent\qemu-ga-x86_64.msi /quiet /norestart
Get-Service QEMU-GA
```

### Troubleshooting: "Guest Agent not running" in Proxmox despite the service showing Running

Check the Proxmox host side:

```bash
qm agent <vmid> ping
qm config <vmid> | grep agent
```

If `agent: 1` is set but ping fails with "QEMU guest agent is not running", the **VirtIO Serial driver** is likely missing — the service can run without it, but has no channel to talk to the host. Check for devices with driver errors inside the guest:

```powershell
Get-PnpDevice | Where-Object {$_.Status -ne 'OK'}
```

Look for **"PCI Simple Communications Controller"** — this is the unrecognized VirtIO Serial port. Install it (and the Balloon driver, needed for memory ballooning):

```powershell
pnputil /add-driver <drive-letter>:\vioserial\2k22\amd64\*.inf /install
pnputil /add-driver <drive-letter>:\Balloon\2k22\amd64\*.inf /install
Restart-Computer
```

After restart, re-check `Get-PnpDevice` — one or two generic "PCI Device"/"ACPI\QEMU..." entries with errors can remain (unrelated to the guest agent, safe to ignore). Confirm the agent responds:

```bash
qm agent <vmid> get-osinfo
```

If this returns OS details instead of an error, the guest agent is fully working — a bare `qm agent <vmid> ping` can return nothing visible even on success, so `get-osinfo` (or checking the Summary tab's IP field after a refresh) is a clearer test.

---

## 8. Set ballooning memory limits

With only 8GB total across the homelab, avoid a fixed memory allocation for the VM — use ballooning so it only takes what it needs.

`VM → Hardware → Memory → Edit`

- **Memory (max):** `4096` MiB
- **Minimum memory:** `2048` MiB (must be lower than Memory — setting them equal disables ballooning in practice even with the checkbox on)
- **Ballooning Device:** checked

Ballooning only works once the QEMU Guest Agent is installed and communicating (see step 7) — without it, the VM sits at its max allocation regardless of actual usage.

---

## 9. Enable Remote Desktop (Server Core, via sconfig)

From the Server Core menu (`sconfig`), choose the Remote Desktop option, then **Enable**, and select:

**1) Allow only clients running Remote Desktop with Network Level Authentication (more secure)**

NLA requires authentication before a full RDP session opens, protecting against brute-force attempts against the login screen. All modern Windows clients (10/11, Server 2016+) support NLA — only pick option 2 if connecting from a legacy client that doesn't.

---

## 10. Fix VM clock (RTC/UTC mismatch)

Proxmox/QEMU sets the VM's hardware clock (RTC) to UTC, but Windows assumes the RTC is already local time by default — causing the guest's clock to be off by your UTC offset. This can cascade into other failures (e.g. DNS forwarding breaking, TLS/cert validation issues).

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1
Restart-Computer
```

After restart, verify:

```powershell
Get-Date
w32tm /resync /force
```

If the DNS server role is already installed and forwarding fails with timeouts/`RCODE_SERVER_FAILURE`, check the clock **before** troubleshooting DNS zones — a wrong clock is a common but easy-to-miss root cause.

---

## 11. Install AD DS and promote to Domain Controller

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "YourStrongP@ssw0rd" -AsPlainText -Force)
```

- Replace `lab.local` with your chosen domain — `.local`/`.lab` is fine for a lab, avoid it in production.
- The SafeModeAdministratorPassword is for DSRM (Directory Services Restore Mode) recovery only — keep it different from your admin password and store it somewhere safe.
- `InstallDns:$true` installs the DNS role automatically (required for AD DS).
- The VM reboots automatically once promotion completes. After reboot, the login screen expects the domain account (`LAB\Administrator`) instead of the local one.

**Verify:**

```powershell
Get-ADDomain
Get-Service ADWS,DNS,Netlogon,NTDS
```

---

## 12. Configure DNS forwarding (AD DNS → AdGuard)

The AD DNS role only knows about its own domain by default — it needs a forwarder to resolve anything external (and to keep ad-blocking/filtering for domain-joined devices).

```powershell
Import-Module DnsServer
Set-DnsServerForwarder -IPAddress <adguard-ip>
```

**Troubleshooting checklist if external names still don't resolve** (e.g. `Resolve-DnsName aka.ms` times out or returns `RCODE_SERVER_FAILURE`), work through these in order:

1. **Confirm the VM's clock is correct first** (see step 10) — a wrong clock has been observed to break DNS forwarding entirely; fixing the clock alone resolved it in testing here, though the exact mechanism wasn't conclusively identified.
2. **Test the forwarder target directly**, bypassing local DNS logic:
   ```powershell
   Resolve-DnsName -Name aka.ms -Server <adguard-ip>
   ```
   If this works but a plain `Resolve-DnsName -Name aka.ms` doesn't, the problem is in the Windows DNS server's forwarding/recursion logic, not the forwarder target itself.
3. **Check the network adapter's own DNS setting** — it should point to itself, not skip the local DNS service:
   ```powershell
   Get-DnsClientServerAddress
   Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
   ```
4. **Restart the DNS service** after any forwarder change — it doesn't always pick up a new forwarder live:
   ```powershell
   Restart-Service DNS
   ```
5. **Rule out an auto-created root `.` zone**, which blocks all forwarding (the DNS server believes it's authoritative for the whole root instead of forwarding):
   ```powershell
   Get-DnsServerZone | Format-Table -AutoSize
   Remove-DnsServerZone -Name "." -Force   # only if a "." zone is present
   ```

---

## 13. Install Windows Admin Center (WAC)

Gives a browser-based GUI for managing this Server Core box remotely — since Server Core has no desktop shell, WAC (or RSAT from another Windows machine) is the practical way to get visual management without installing the much heavier Desktop Experience.

```powershell
Invoke-WebRequest -Uri "https://aka.ms/WACdownload" -OutFile "C:\WindowsAdminCenter.msi"
```

**⚠️ Known gotcha:** despite the `.msi` extension, this link may actually return an `.exe` bootstrapper (Microsoft has moved some products from MSI to EXE installers without updating the well-known download link's apparent format). Installing it as-is with `msiexec` fails with:

```
Error 2203 / 1620 — "This installation package could not be opened."
```

**How to confirm this is what's happening** — check the file's actual binary signature:

```powershell
$stream = [System.IO.File]::OpenRead("C:\WindowsAdminCenter.msi")
$buffer = New-Object byte[] 8
$stream.Read($buffer, 0, 8)
$stream.Close()
[BitConverter]::ToString($buffer)
```

- `D0-CF-11-E0-...` = genuine MSI (OLE Compound File) — the 2203 error lies elsewhere (corrupted download, permissions, TrustedInstaller ACLs, `Unblock-File`).
- `4D-5A-...` (`MZ`) = this is actually a **Windows EXE**, mislabeled with a `.msi` extension. This was the actual cause here, despite the file downloading at the expected size both times.

**Fix — just rename and run it as an EXE:**

```powershell
Rename-Item -Path "C:\WindowsAdminCenter.msi" -NewName "WindowsAdminCenter.exe"
C:\WindowsAdminCenter.exe
```

Proceed through the installer (Express install is fine — same defaults as the command-line flags would set: port 443, self-signed certificate, automatic updates). The self-signed cert is valid for 60 days; re-run the installer to regenerate it once it expires.

**After install, the service name differs from older docs/guides:**

```powershell
Get-Service | Where-Object {$_.DisplayName -like "*Admin Center*"}
# Real service name observed: WindowsAdminCenter (not ServerManagementGateway)

Start-Service WindowsAdminCenter
Set-Service -Name WindowsAdminCenter -StartupType Automatic
```

**Access from another machine's browser:**

```
https://<server-ip>
```

Accept the self-signed certificate warning, then log in with the local (or domain) Administrator account.

---

## Notes

- The 180-day evaluation license is for testing/practice only — not for production use. Reinstall or convert to a licensed copy if this VM becomes permanent infrastructure.
- If cloning this VM later for additional practice environments, run `sysprep` inside Windows first to avoid SID/hostname conflicts between clones.
- In the Proxmox noVNC/xterm.js console, physical Ctrl+Alt+Del is captured by your own host OS, not the VM — use the console's **Send Key → Ctrl+Alt+Del** button instead to unlock the guest's login screen.
- Total homelab RAM budget matters: check `free -h` on the Proxmox host if things feel slow — swap usage there is a clear sign of memory overcommit across VMs/containers. The rest of this homelab (pfSense, AdGuard, and a TurnKey Linux LXC file server) is lightweight; a Windows Server + AD DS VM is the heaviest thing likely to be added.
- Before troubleshooting a "mysterious" networking/DNS/service issue at length, check the VM's clock first (step 10) — it's an easy, fast thing to rule out and has caused cascading, hard-to-diagnose failures here.
- When a download from a well-known Microsoft `aka.ms` link fails to install despite matching the expected file size, check the file's actual binary signature (step 13) before assuming the download itself is broken — the file extension isn't always trustworthy.
