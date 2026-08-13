# Mapping a TrueNAS Network Drive on Windows

## Method 1: Right-click "This PC"

1. In File Explorer's left sidebar, find **This PC**.
2. Right-click on **This PC**.
   - On Windows 11: click **Show more options** (or press `Shift + F10`).
3. Select **Map network drive...**

## Method 2: Via the "..." menu

1. Left-click **This PC** to open it.
2. In the toolbar at the top, click the **...** (three dots).
3. Select **Map network drive** from the menu.

## Settings

- **Drive**: pick any letter (e.g. `Z:`)
- **Folder**: enter the path:
  ```
  \\172.31.240.132\SharedFiles
  ```
  or, over VPN:
  ```
  \\<VPN_IP_OF_TRUENAS>\SharedFiles
  ```
- Check **Reconnect at sign-in** so it stays connected after reboot.
- Click **Finish**.

## Extra Tips for VPN Access

- **Firewall / Routing**: make sure the VPN allows traffic on port `445` (SMB).
- **Credentials**: when connecting from outside the local network, Windows will ask for credentials again (username & password). Check **"Remember my credentials"** so it doesn't ask every time the VPN connects.
- **DNS / Name Resolution**: hostname resolution (e.g. `\\truenas\SharedFiles`) often doesn't work over VPN, so using the VPN IP directly is the most reliable way to connect instantly.
