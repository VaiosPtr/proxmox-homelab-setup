# WireGuard VPN — LXC Container Setup

## Creating the LXC

1. In the Proxmox Web UI, download the Debian 12 or Ubuntu template.
2. Click **Create CT** (top right).
3. **Network**: assign a static IP on the LAN (e.g. `192.168.1.202/24`), gateway = router IP (e.g. `192.168.1.1`).
4. **Unprivileged Container**: uncheck it (or, if you keep it unprivileged, enable TUN/TAP devices in **Options**) so WireGuard can create its virtual network interface.

## Installing WireGuard (via PiVPN)

1. Open the **Shell** of the new LXC and install PiVPN (the fastest way to set up WireGuard):
   ```bash
   curl -L https://install.pivpn.io | bash
   ```
2. During setup, set the VPN subnet to `10.10.10.0/24`.
3. Enable NAT / IP forwarding so clients on `10.10.10.x` can reach the LAN (`192.168.1.x`).

## Port Forwarding on the Router

On the router, forward:
- **Protocol**: UDP
- **Port**: `51820` (default WireGuard port)
- **Internal IP**: `192.168.1.202` (the LXC's IP)

## Generating Client Credentials

Run inside the LXC:
```bash
pivpn add
```

This generates a `.conf` file (or QR code for mobile) for the client device — ready to connect.
