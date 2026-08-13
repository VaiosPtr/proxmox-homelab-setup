# TrueNAS SCALE — VM Setup on Proxmox

## Download the TrueNAS ISO Directly to Proxmox

No need to download the ISO to your PC and upload it — Proxmox can fetch it directly:

1. In the left tree, click **local (pve-lab)** (under your node).
2. In the middle column, select **ISO Images**.
3. Click **Download from URL**.
4. Paste the TrueNAS SCALE download link into the **URL** field.
5. Click **Query URL** (to auto-fill the filename), then **Download**.

## Creating the VM

Once the ISO download finishes, click **Create VM** (top right) and fill in each tab:

**General**
- Node: `pve`
- VM ID: `100` (leave as default or pick any number you want)
- Name: `truenas-scale`

**OS**
- ISO image: `TrueNAS-25.10.4.iso`
- Type: Linux

**System**
- Graphic card: Default
- Machine: Default (i440fx)
- BIOS: **OVMF (UEFI)** — important for TrueNAS!
- EFI Storage: `local-lvm`

**Disks** (system disk for TrueNAS itself)
- Disk size: 16 GiB (enough for the OS)
- Storage: `local-lvm`

**CPU**
- Cores: 2

**Memory**
- Memory: 4096 MiB (4 GB)

**Network**
- Bridge: `vmbr0`
- Model: VirtIO (paravirtualized)

**Confirm**
- Uncheck **"Start after created"** — we need to pass through the two extra 50 GB disks first.
- Click **Finish**.

## Passing Through the Extra Disks

1. Click **pve** in the left menu.
2. Open **Shell** (top right).
3. Identify the disks:
   ```bash
   lsblk
   ls -l /dev/disk/by-id/
   ```
   You'll see unique names (serial numbers), e.g.:
   - `scsi-360022480...`
   - `ata-VBOX_HARDDISK_...`
   - `nvme-Samsung_SSD_...`

   Tip: `ls -l /dev/disk/by-id/` shows arrows (`->`) pointing to `sdb` / `sdc` so you can tell them apart quickly.

4. Attach each disk to the VM using its full by-id path (don't use `/dev/sdb` directly, as that can change):
   ```bash
   qm set 100 -scsi1 /dev/disk/by-id/scsi-360022480XXXXXXXXXXXXX
   qm set 100 -scsi2 /dev/disk/by-id/scsi-360022480YYYYYYYYYYYYY
   ```

## Network Troubleshooting (No IP on Boot)

Once boot completes, the console should show:
```
The Web UI is at:
http://192.168.X.X
```

If instead you see:
> "The web interface could not be accessed. Please check network configuration"

This means TrueNAS booted fine but didn't get an IP via DHCP.

**Fix via Console Setup menu:**
1. Type `1` (Configure Network Interfaces) and press Enter.
2. Select your network interface (e.g. `NET1` or `vtnet0` / `eth0`).
3. **Reset network configuration?** → `n` (No)
4. **Configure IPv4?** → `y` (Yes)
5. **DHCP?** → `y` (Yes) to retry automatic IP.

**If DHCP still fails, set a static IP:**
1. **Configure IPv4?** → `y`
2. **DHCP?** → `n`
3. **IPv4 Address**: pick a free IP on your network, including the prefix, e.g. `192.168.1.250/24`
4. **IPv6?** → `n`

**Check on the Proxmox side too:**
- Go to `100 (truenas-scale)` → **Hardware** → **Network Device (net0)**
- Confirm it's connected to the `vmbr0` bridge.

## Creating a Storage Pool

1. In the TrueNAS left menu, click **Storage**.
2. Top right, click **Create Pool**.
3. Name: e.g. `main-pool` or `data`.

**Selecting disks and RAID type:**
1. Under **Data Devices**, click **Add Layout** (or check the available disks).
2. You'll see your two 50 GB disks (`vdb`, `vdc` or similar).
3. Choose a **Layout**:
   - **Mirror** (recommended) — like RAID 1. Disks mirror each other, so if one fails you don't lose data. Usable space: ~50 GB.
   - **Stripe** — like RAID 0. Combines capacity (~100 GB usable), but losing one disk loses everything.
4. Click **Save Layout / Create**.
5. Confirm the warning that the disks will be wiped, check **Confirm**, and click **Create Pool**.
