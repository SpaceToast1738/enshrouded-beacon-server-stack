# Port forwarding

The Enshrouded dedicated server uses **three UDP ports** for
gameplay + Steam server-list discovery. All three need to be:

- Open on the **host's firewall** (Ubuntu's `ufw` by default).
- Forwarded through your **router** to the host's LAN IP (so
  players outside your network can connect).

## Ports

| Port  | Protocol | Purpose |
|------:|:--------:|---------|
| 15636 | UDP      | Gameplay traffic (player movement, world events) |
| 15637 | UDP      | Query port (server browser / Steam list visibility) |
| 15638 | UDP      | Some server builds use this too — open it for safety |

All three are exposed in `docker-compose.yml`'s
`enshrouded-server` service. Docker handles the
container-to-host port mapping; you handle the
host-to-router-to-internet forwarding.

## Host-level firewall (Ubuntu `ufw`)

Open the three ports inbound:

```bash
sudo ufw allow 15636/udp comment 'Enshrouded gameplay'
sudo ufw allow 15637/udp comment 'Enshrouded query'
sudo ufw allow 15638/udp comment 'Enshrouded misc'
sudo ufw reload
sudo ufw status verbose | grep 1563
```

You should see all three listed as `ALLOW IN` with `Anywhere`
or your operator policy.

## Router port-forwarding

This step depends on your router brand. The general pattern:

1. Log into your router's admin page (typically
   `http://192.168.1.1` or `http://192.168.0.1`).
2. Find a **Port Forwarding** / **NAT** / **Virtual Server**
   section.
3. Create three rules — one per port:

| External port | Internal IP    | Internal port | Protocol |
|--------------:|----------------|--------------:|:--------:|
| 15636         | <your-host-IP> | 15636         | UDP      |
| 15637         | <your-host-IP> | 15637         | UDP      |
| 15638         | <your-host-IP> | 15638         | UDP      |

Where `<your-host-IP>` is the Ubuntu host's LAN address
(e.g. `192.168.1.42`). To find it:

```bash
ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v 127.0
```

**Static IP / DHCP reservation recommended.** If the host's
LAN IP changes after a reboot, the port-forward rules break.
Use a router-side DHCP reservation or set a static IP on the
host.

## Verifying from outside your network

Easiest: ask a friend on a different network to connect to
your **public IP** (find it via `curl ifconfig.me`) in-game.

For a quick CLI check from outside (or from a phone on
cellular), `nc -zuv <public-ip> 15636` won't work cleanly for
UDP, but several online tools (`canyouseeme.org`,
`portchecktool.com`) test specific UDP ports against your
public IP.

If the port isn't open externally but IS open on the host
firewall, the router's port-forward rule is the problem.

## Cloud / VPS hosting

If you're running this stack on a cloud VM (DigitalOcean,
Hetzner, Linode, etc.) rather than a home host:

1. The VM provider's **firewall** / **security group** also
   needs UDP 15636-15638 inbound. Check the provider's
   dashboard.
2. There's no router-level forwarding step — the VM has a
   direct public IP. Just the host-level `ufw` rules above
   are sufficient.

## ISP-level caveats

- **CGNAT** (carrier-grade NAT, common on mobile / starter
  fibre plans): you can't accept inbound connections on a
  public IP because you don't have one. Solutions: ask your
  ISP for a real IPv4 ($5-10/mo upsell typically), use a
  reverse-VPN tunnel (e.g. Tailscale Funnel, ngrok TCP,
  Cloudflare Tunnel — though CF doesn't proxy UDP well), or
  host on a small VPS instead.
- **Symmetric NAT** (some mobile networks): same problem;
  STUN-based workarounds don't help for a server use case.
- **ISP blocks game-server traffic**: rare but happens on
  some residential plans. If `ufw status` shows the port
  open + router-side forwarding is configured + the
  external port test STILL fails, your ISP may be
  inspecting + dropping. Switch to a VPS.

## After opening the ports

Restart the stack so the `enshrouded-server` container
re-registers with Steam's server list now that traffic is
flowing:

```bash
docker compose restart enshrouded-server
```

Within a minute, the server should appear in the in-game
**Multiplayer → Server browser** under your `SERVER_NAME`.
