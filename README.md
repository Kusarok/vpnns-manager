<h1 align="center">vpnns-manager</h1>

<p align="center">
  <b>Run one app behind OpenVPN/NordVPN without breaking SSH — give a single Linux server two public IPs using a network namespace.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/shell-bash-4EAA25?logo=gnubash&logoColor=white">
  <img src="https://img.shields.io/badge/VPN-OpenVPN%20%2F%20NordVPN-EA7E20">
  <img src="https://img.shields.io/badge/linux-network%20namespace-FCC624?logo=linux&logoColor=black">
  <img src="https://img.shields.io/badge/systemd-ready-30A0E0">
  <img src="https://img.shields.io/badge/tested%20on-Ubuntu%2022.04%20%2F%2024.04-E95420?logo=ubuntu&logoColor=white">
</p>

`vpnns-manager` is a small, dependency-free Bash tool that puts **one application inside an isolated Linux network namespace** and routes only that namespace through an OpenVPN (or NordVPN) tunnel. Your SSH session, package updates, and everything else on the host keep using the server's real IP — only the app you choose exits through the VPN.

It is **interactive** (asks for your `.ovpn` path, VPN username and password, then wires everything up for you), **idempotent** (safe to re-run), and **systemd-ready** (survives reboots). Tested on Ubuntu 22.04 / 24.04 with OpenVPN 2.5+.

---

## How do I use a VPN on a server without breaking my SSH connection?

The usual problem: you start OpenVPN on a remote server, it sets `redirect-gateway`, the default route changes, and your SSH session dies. `vpnns-manager` avoids this entirely. The VPN runs **inside a separate network namespace**, so the host's main routing table is never touched. SSH stays up, and only processes you launch inside the namespace use the tunnel.

## How can one server have two public IP addresses?

After setup you effectively have two exits:

- **Host side** → the server's original public IP (SSH, host traffic).
- **Namespace side** → the VPN's public IP (the app you run inside it).

`vpnns.sh status` prints both side by side so you can confirm they differ.

## How do I run only one app/process through a VPN on Linux?

Launch it inside the namespace:

```bash
sudo ./vpnns.sh run "python main.py"
# or drop into a shell that is fully inside the VPN:
sudo ./vpnns.sh shell
```

Everything started from there uses the VPN IP; everything else on the box does not.

---

## Quick start

```bash
# 1) Install prerequisites
sudo apt update
sudo apt install -y openvpn iproute2 curl iptables

# 2) Get the tool
git clone https://github.com/Kusarok/vpnns-manager.git
cd vpnns-manager
chmod +x vpnns.sh

# 3) Interactive setup — asks for .ovpn path, VPN username & password, etc.
sudo ./vpnns.sh init

# 4) Build the namespace + routing, then connect
sudo ./vpnns.sh setup
sudo ./vpnns.sh start

# 5) Confirm the two IPs differ
sudo ./vpnns.sh status

# 6) Run your app behind the VPN
sudo ./vpnns.sh run "python main.py"
```

During `init` you are asked for:

- Path to your `.ovpn` file (NordVPN or any OpenVPN provider)
- VPN **username** and **password** (stored in `/etc/vpnns/auth.txt`, `chmod 600`)
- Namespace name, private subnet, DNS servers
- Optional app directory and command

The tool **copies** your `.ovpn` to `/etc/vpnns/`, injects `auth-user-pass` pointing at your credentials file, and adds `pull-filter ignore "dhcp-option DNS"` so the namespace DNS stays correct. **Your original `.ovpn` is never modified.**

---

## Example output

`status` proves the isolation at a glance — the host keeps its real IP while the namespace exits through the VPN:

```text
$ sudo ./vpnns.sh status
Namespace          : vpnns
OpenVPN            : 48213
Tunnel             : tun0
Host public IP     : 203.0.113.10      # your VPS / SSH stays here
Namespace public IP: 185.244.214.7     # the app exits via the VPN
Result             : OK — namespace traffic is going through the VPN.
----- last log lines (/var/log/vpnns-vpnns.log) -----
... Initialization Sequence Completed
```

---

## Commands

| Command | What it does |
|---------|--------------|
| `init` | Interactive setup: writes config, `auth.txt`, and the edited `.ovpn` copy |
| `setup` | Create namespace, veth pair, routing and NAT (idempotent) |
| `start` | Start OpenVPN inside the namespace and wait for the tunnel |
| `stop` | Stop OpenVPN |
| `status` | Show tunnel state + **host public IP vs namespace public IP** |
| `run [cmd]` | Run a command inside the namespace (defaults to `APP_CMD`) |
| `shell` | Open a shell inside the namespace |
| `up` | `setup` + `start` in the foreground (used by systemd) |
| `destroy` | Tear everything down (idempotent) |
| `install-service` | Install + enable the systemd unit |

---

## Run on boot (systemd)

```bash
sudo ./vpnns.sh install-service
sudo systemctl start vpnns.service
sudo systemctl status vpnns.service
```

The unit runs `vpnns.sh up` (setup + OpenVPN in the foreground) and restarts on failure.

---

## Works on many servers

Nothing is hard-coded. Every path, IP, interface, and the OpenVPN server come from the generated config at `/etc/vpnns/vpnns.conf` (template: [`vpnns.conf.example`](vpnns.conf.example)). The external interface is auto-detected via `ip route get 1.1.1.1`, so you can drop the same script on any box and just run `init`.

---

## Security

> **Never commit your `auth.txt`, `.ovpn`, or `vpnns.conf`.** They contain credentials.

- `auth.txt` and `vpnns.conf` are written with `chmod 600`.
- The repo's [`.gitignore`](.gitignore) blocks `*.ovpn`, `auth.txt`, `*.conf` (except the example), keys and logs.
- Secrets live under `/etc/vpnns/`, never inside the repository.

---

## Troubleshooting

**The two IPs are the same / "NOT isolated".**
The VPN didn't come up. Run `sudo ./vpnns.sh status` and check the log shown at the bottom. You want to see:

```
Initialization Sequence Completed
```

**`start` times out.**
Inspect the log (default `/var/log/vpnns-<ns>.log`). Common causes: wrong username/password in `auth.txt`, wrong `.ovpn` path, or the provider rejecting the protocol/port.

**No internet inside the namespace before connecting the VPN.**
`setup` pings `10.200.1.1` and `1.1.1.1`. If the second fails, check `EXT_IF` in the config and that `net.ipv4.ip_forward=1`.

**DNS not resolving inside the namespace.**
The namespace uses `/etc/netns/<ns>/resolv.conf`. `vpnns-manager` re-asserts it after connecting; make sure your provider config didn't strip the `pull-filter` line from the copied `.ovpn`.

**Everything is stuck / want a clean slate.**
`sudo ./vpnns.sh destroy` removes the namespace, veth and iptables rules safely (idempotent), then re-run `setup`.

---

## How it works

```
                 ┌────────────────────────── host (real IP) ──────────────────────────┐
   SSH / host ───┤  default route ── eth0 ── Internet (real public IP)                 │
                 │                                                                      │
                 │   veth (vh-vpnns) 10.200.1.1  ── NAT/MASQUERADE ── eth0              │
                 └───────────────┬──────────────────────────────────────────────────────┘
                                 │  veth pair
                 ┌───────────────┴────── netns "vpnns" ─────────────────────────────────┐
   your app  ───►│ veth (vp-vpnns) 10.200.1.2 ── default ── OpenVPN tun0 ── VPN exit IP │
                 └──────────────────────────────────────────────────────────────────────┘
```

## License

MIT.
