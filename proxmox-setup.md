# Proxmox — First Settings After Installation

## 1. Disable Enterprise Repository & Enable No-Subscription Repository

By default, Proxmox looks for a paid subscription repository. We need to switch it to the free update channel:

1. In the left tree, click on your node (e.g. `pve-lab`).
2. In the middle column, go to **Repositories** (under the **System** section).
3. Find the `pve-enterprise` line (marked with a yellow/orange icon), click it, and press **Disable**.
4. Click **Add** (top left).
5. In the dialog, set **Repository** to **No-Subscription**.
6. Click **Add**.

## 2. Run Updates

1. Right above **Repositories**, click the **Updates** tab.
2. Click **Refresh** (top left) and wait a few seconds for the package lists to download (close the popup window once it shows `TASK OK`).
3. Click **Upgrade** (next to Refresh). A terminal window will open. If prompted `[Y/n]`, type `y` and press Enter. Close the window once finished.
