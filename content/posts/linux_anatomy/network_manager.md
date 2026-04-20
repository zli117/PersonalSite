---
date: '2026-04-19T22:03:50-07:00'
draft: true
title: 'Chapter 12: NetworkManager'
weight: 120
---
## What NetworkManager is and why it exists

Classical Linux networking was built around `/etc/network/interfaces` (Debian) or `/etc/sysconfig/network-scripts/` (Red Hat) — static config files read by init scripts at boot. Define your interfaces, bring them up, you're done. Perfect for servers that never move.

Desktops and laptops broke this entirely. The assumptions that failed:

- **The machine moves.** Home WiFi, coffee shop, office ethernet, phone hotspot. Network changes constantly.
- **Multiple interfaces, competing for primacy.** Ethernet *and* WiFi connected; which is the default route? What happens when you unplug the cable?
- **Authentication.** WPA2, WPA3, 802.1X, captive portals, VPNs — each with its own daemon and state machine. Coordinating them was a nightmare of custom scripts.
- **Services depend on the network.** Syncing clients, mail fetchers, chat apps. They need to know "network is up now" / "network is going down" to behave correctly.
- **Users aren't admins.** Alice shouldn't need `sudo` to connect to her phone's hotspot or run a VPN. But network config is privileged.
- **Hot-plug.** Plug in a USB ethernet dongle → it should just work. Phone tethering over USB → same.

**NetworkManager** (started ~2004 by Red Hat) is the daemon that consolidated all of this. It runs as a system service, owns network interfaces (usually), manages connection profiles, drives auxiliary daemons like wpa_supplicant, runs its own DHCP client, coordinates with systemd-resolved for DNS, talks to firewalld for zone assignment, and exposes everything on D-Bus so that CLIs (`nmcli`), GUIs (GNOME's network applet, KDE's plasma-nm), and scripts can drive it uniformly.

Most desktop Linux distros ship NM enabled by default. Servers often use `systemd-networkd` instead (simpler, static-config-friendly, no D-Bus surface). Both solve the same problem from different ends: NM is dynamic and user-facing; networkd is declarative and server-facing. You'll mostly encounter NM because you mostly use laptops.

## The central abstraction: device vs. connection

This is the single most important distinction in NetworkManager, and almost every piece of confusion traces back to not having it straight.

- **Device** — a network interface the kernel has. `wlp194s0`, `eno1`, `wg0`, a bridge, a bond. Hardware (or virtual), discovered from the kernel via netlink, lives as long as the kernel has it. NM either manages it or doesn't.

- **Connection** — a saved configuration profile. "My home WiFi, WPA3, these DNS servers, this firewall zone." Stored as keyfiles on disk. Can be activated on a compatible device. Many connections can target one device; only one is active at a time.

A connection is *not* an interface. A connection is a *config to apply* to an interface. When you "connect to WiFi X" in the GUI, what's happening is: NM finds the connection profile for X (or creates one), figures out which device can carry it (your WiFi card), and activates the profile *on* the device.

```bash
nmcli device status                  # what interfaces exist, are they NM-managed, what state
# DEVICE          TYPE      STATE        CONNECTION
# wlp194s0        wifi      connected    HomeNet
# eno1            ethernet  unavailable  --
# wg0             wireguard connected    home-vpn
# lo              loopback  unmanaged    --

nmcli connection show                # what profiles are saved
# NAME      UUID                                  TYPE      DEVICE
# HomeNet   5f...                                 wifi      wlp194s0
# Work      8a...                                 wifi      --
# home-vpn  2c...                                 wireguard wg0
# Wired     93...                                 ethernet  --
```

The `DEVICE` column on `connection show` is "what device is this profile currently active on, if any." Profiles can exist without being active (Work is saved but not connected right now).

Once this distinction clicks, every `nmcli` command makes sense. Almost every command either operates on a device (bring up/down, reassociate, scan) or on a connection (add, modify, delete, activate).

## Connection profiles: what's in one

A connection is a typed settings dictionary. Different connection *types* (wifi, ethernet, wireguard, bridge, bond, vlan, team, macvlan, vpn, gsm, bluetooth, loopback, etc.) use different settings.

The core settings are organized into **sections**, each with their own prefix:

| Section | Purpose |
|---|---|
| `connection` | Identity: name, UUID, type, autoconnect, zone, timestamp |
| `ipv4`, `ipv6` | Address config: method, addresses, gateway, DNS, routes, DHCP options |
| `802-3-ethernet` | Ethernet-specific: MAC, MTU, wake-on-LAN |
| `802-11-wireless` | WiFi: SSID, mode, band, channel, BSSID, hidden, powersave |
| `802-11-wireless-security` | WiFi auth: key-mgmt, PSK, WPA version, EAP |
| `802-1x` | Enterprise EAP: eap type, identity, CA cert, client cert, phase2 |
| `wireguard` | WireGuard: private-key, listen-port, fwmark, peers... |
| `vpn` | Plugin-based VPNs: openvpn, strongswan, openconnect |
| `bridge`, `bond`, `vlan`, `team` | Virtual interface types |
| `proxy` | PAC / HTTP proxy settings |

```bash
nmcli connection show HomeNet         # every setting on this profile
# connection.id:                          HomeNet
# connection.uuid:                        5f...
# connection.type:                        802-11-wireless
# connection.autoconnect:                 yes
# connection.zone:                        home
# ipv4.method:                            auto
# ipv4.dns:                               1.1.1.1,9.9.9.9
# ipv4.addresses:                         --
# ipv4.gateway:                           --
# 802-11-wireless.ssid:                   HomeNet
# 802-11-wireless.mode:                   infrastructure
# 802-11-wireless-security.key-mgmt:      wpa-psk
# 802-11-wireless-security.psk:           <hidden>
# ...
```

Profiles live in `/etc/NetworkManager/system-connections/<name>.nmconnection` as INI-format keyfiles, mode 0600 (contain passwords). You can edit them by hand, or use `nmcli` / the GUI.

```bash
sudo ls -la /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/HomeNet.nmconnection
```

The keyfile is the canonical state. If you edit it manually, `nmcli connection reload` picks up the changes.

## The IP method: `auto`, `manual`, `shared`, `link-local`, `disabled`

`ipv4.method` and `ipv6.method` are where most of the behavior lives. The values:

- **`auto`** — DHCP (ipv4) or SLAAC+DHCPv6 (ipv6). Normal client.
- **`manual`** — Static. Set `ipv4.addresses`, `ipv4.gateway`, `ipv4.dns`, `ipv4.routes` by hand.
- **`shared`** — *The laptop becomes a router.* This is the hotspot/NAT-sharing mode. NM spins up dnsmasq on the interface, hands out DHCP leases on 10.42.0.0/24 (ipv4) or a ULA prefix (ipv6), enables IP forwarding, sets up NAT masquerade via firewalld, allows DNS forwarding. One command and you're sharing your upstream connection.
- **`link-local`** — 169.254.x.x, no DHCP. Useful for direct peer links.
- **`disabled`** — no IP on this family. Common for `ipv6.method=disabled` if you don't want v6.

You've already used `shared` for your hotspot experiments. That's the magic word — NM does the whole NAT+DHCP+DNS-forward setup in response to `ipv4.method=shared`, no manual dnsmasq needed.

## Device states and the lifecycle

Every NM-managed device moves through a defined state machine:

```
unmanaged  → unavailable  → disconnected  → prepare  → config
    ↑                                                      │
    │                                                      ▼
    └─────  deactivating ←  activated  ←  ip-config  ←  need-auth
```

The states you'll see:

| State | Meaning |
|---|---|
| `unmanaged` | NM isn't touching this device (explicitly or by config) |
| `unavailable` | Hardware not ready (WiFi radio off, cable unplugged) |
| `disconnected` | Ready but no active connection |
| `prepare` | Bringing up the link layer |
| `config` | Authenticating / associating |
| `need-auth` | Waiting for credentials (user/password prompt, etc.) |
| `ip-config` | Running DHCP / configuring addresses |
| `ip-check` | Post-config connectivity check |
| `secondaries` | Activating dependent connections (VPN on top of WiFi) |
| `activated` | Fully up, routes installed, DNS configured |
| `deactivating` | Tearing down |
| `failed` | Activation failed |

When something's not working, the state tells you which phase is stuck.

```bash
nmcli device monitor                 # watch state transitions live
nmcli connection monitor
journalctl -u NetworkManager -f      # detailed logs
```

## nmcli: the full vocabulary

`nmcli` is NM's command-line interface. Four objects:

```bash
nmcli general        # NM-wide state, hostname, logging
nmcli networking     # master enable/disable
nmcli radio          # wifi/wwan radio enable/disable
nmcli device         # interfaces
nmcli connection     # profiles
nmcli agent          # secret agent (password prompts)
nmcli monitor        # real-time events
```

The common verbs: `show`, `status`, `up`, `down`, `connect`, `disconnect`, `add`, `modify`, `delete`, `reload`, `scan`.

### Everyday recipes

**Connect to WiFi by SSID:**
```bash
nmcli device wifi list                    # scan
nmcli device wifi connect "MyNet" password "hunter2"
# creates a profile named MyNet if it doesn't exist, activates it
```

**Connect to hidden WiFi:**
```bash
nmcli device wifi connect "HiddenNet" password "pw" hidden yes
```

**Bring a saved connection up/down:**
```bash
nmcli connection up HomeNet
nmcli connection down HomeNet
```

**Disconnect a device (puts it in `disconnected` state):**
```bash
nmcli device disconnect wlp194s0
```

**Delete a saved profile:**
```bash
nmcli connection delete HomeNet
```

**Static-IP ethernet:**
```bash
nmcli connection add type ethernet con-name StaticHome ifname eno1 \
    ipv4.method manual \
    ipv4.addresses 192.168.1.50/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "1.1.1.1,9.9.9.9"
nmcli connection up StaticHome
```

**Modify a field on an existing connection:**
```bash
nmcli connection modify HomeNet ipv4.dns "1.1.1.1"       # replace
nmcli connection modify HomeNet +ipv4.dns "9.9.9.9"      # append
nmcli connection modify HomeNet -ipv4.dns "9.9.9.9"      # remove
nmcli connection modify HomeNet ipv4.ignore-auto-dns yes # use manual DNS, not DHCP's
```

Modifications take effect on the next `connection up`. If the connection is currently active, you'll need to re-activate:

```bash
nmcli connection down HomeNet && nmcli connection up HomeNet
# or the shortcut, which reapplies compatible changes without tearing down:
nmcli device reapply wlp194s0
```

**The `modify` scope:** NM settings use dotted paths (`ipv4.dns`, `802-11-wireless.ssid`). `nmcli connection modify CONNAME SETTING VALUE` is the pattern. For list-valued settings, `+`/`-` append/remove.

**Interactive editor** (useful for complex edits):
```bash
nmcli connection edit HomeNet
# (drops you into a sub-shell with tab completion and `set`, `describe`, `save`)
```

**Create a WiFi hotspot** (the command we used earlier):
```bash
nmcli device wifi hotspot ifname wlp194s0 ssid MyAP password "somepass123"
# creates a wifi profile with mode=ap, ipv4.method=shared, fires it up
```

**WireGuard:**
```bash
nmcli connection add type wireguard con-name wg-home ifname wg0 \
    ipv4.method manual ipv4.addresses 10.9.0.2/24 \
    wireguard.private-key "$(wg genkey)" \
    wireguard.listen-port 51820
nmcli connection modify wg-home \
    +wireguard-peer.<PEER-PUBKEY> \
    wireguard-peer.<PEER-PUBKEY>.endpoint "server:51820" \
    wireguard-peer.<PEER-PUBKEY>.allowed-ips "0.0.0.0/0"
nmcli connection up wg-home
```

(Syntax is verbose; the keyfile is often easier to write directly for WG.)

### Terse output for scripting

```bash
nmcli -t -f NAME,UUID,TYPE connection show             # tab-separated, selected fields
nmcli -t -f DEVICE,STATE,CONNECTION device status
nmcli -g STATE device status                           # raw values, one per line
```

`-t` = terse; `-f` = fields; `-g` = get (one value only). Good for shell pipelines.

## Secrets, the agent model, and how passwords flow

NM splits "policy" (the connection profile) from "secrets" (passwords, keys). A profile can store secrets itself, or defer them to an **agent**.

Storage flags per secret, expressed as `802-11-wireless-security.psk-flags`:

- `0` = store plainly in system keyfile (readable by root)
- `1` = agent-owned, stored encrypted in the desktop keyring (GNOME Keyring, KDE Wallet)
- `2` = ask every time (no storage)
- `4` = not required

When a connection needs a secret, NM calls its registered agents over D-Bus. Each desktop environment ships one:

- **gnome-shell** — integrates with GNOME Keyring, prompts via GNOME's dialog.
- **plasma-nm** — integrates with KDE Wallet.
- **nm-applet** — for non-GNOME/KDE DEs with the tray applet.
- **nmcli** itself — can act as an agent on headless/TTY systems (prompts in the terminal when you run `nmcli connection up` on a connection with agent-owned secrets).

A server without a desktop session has no agent running. If you save WiFi credentials with agent-owned flags on such a system, nothing can prompt, and activation fails. Fix: `nmcli connection modify CONNAME 802-11-wireless-security.psk-flags 0` to store system-wide. Similarly, connections used at boot must store their secrets in the system keyfile.

```bash
nmcli --ask connection up HomeNet     # prompt interactively for any missing secrets
```

## The D-Bus API

NM is a D-Bus citizen: `org.freedesktop.NetworkManager` on the system bus. Every operation `nmcli` does is a D-Bus call; every GUI client speaks the same API.

```bash
busctl tree org.freedesktop.NetworkManager | less
```

The shape:

- `/org/freedesktop/NetworkManager` — the manager, with methods like `ActivateConnection`, `GetDevices`, `CheckConnectivity`, and properties like `State`, `Connectivity`, `PrimaryConnection`.
- `/org/freedesktop/NetworkManager/Devices/<n>` — one per device, with state, addresses, statistics, active connection.
- `/org/freedesktop/NetworkManager/Settings` — connection profiles.
- `/org/freedesktop/NetworkManager/ActiveConnection/<n>` — currently-active activations.
- `/org/freedesktop/NetworkManager/IP4Config/<n>`, `IP6Config/<n>`, `DHCP4Config/<n>` — applied configs.

Signals: `StateChanged`, `DeviceAdded`, `DeviceRemoved`, `PropertiesChanged` everywhere. Apps subscribe to these to redraw indicators, run "on connect" scripts, etc.

```bash
busctl --match "type=signal,sender=org.freedesktop.NetworkManager" monitor
# watch every NM signal live as you plug/unplug, connect/disconnect
```

Polkit gates the privileged methods. The actions are named `org.freedesktop.NetworkManager.*` — `network-control`, `enable-disable-network`, `wifi-share-open`, `settings.modify.system`, etc. GNOME Shell's network applet, running as your user, calls these via D-Bus; polkit decides if you're allowed.

```bash
pkaction --verbose --action-id org.freedesktop.NetworkManager.settings.modify.system
```

## Dispatcher scripts: the "run this on connect/disconnect" hook

NM runs scripts from `/etc/NetworkManager/dispatcher.d/` on every state transition, with the event type and device as arguments. Classic use: "when I connect to Work WiFi, start the corporate VPN." "When the laptop gets an address on ethernet, kick a sync."

```bash
sudo tee /etc/NetworkManager/dispatcher.d/10-log.sh >/dev/null <<'EOF'
#!/bin/bash
# Arguments: $1 = interface, $2 = event
logger -t nm-dispatcher "interface=$1 event=$2 CON_ID=$CONNECTION_ID"
EOF
sudo chmod +x /etc/NetworkManager/dispatcher.d/10-log.sh

# Events include: up, down, pre-up, pre-down, vpn-up, vpn-down, hostname,
# dhcp4-change, dhcp6-change, connectivity-change
```

Environment variables passed: `CONNECTION_UUID`, `CONNECTION_ID`, `DEVICE_IFACE`, plus DHCP info (`DHCP4_*`), IP config, routes, DNS, etc. Full list in `man NetworkManager-dispatcher`.

These run as root. They block the state transition until they complete (with a timeout). Keep them fast, and prefer launching longer work asynchronously.

## Integrations with other system services

NM doesn't do networking alone. It coordinates with several other daemons, and understanding these integrations clarifies a lot of behavior.

**wpa_supplicant** — handles WiFi association and authentication (WPA2/WPA3/EAP). NM spawns it (or uses an existing instance) and drives it over D-Bus. You don't usually interact with it directly. When diagnosing WiFi auth failures, `journalctl -u wpa_supplicant` is the second place to look after NM's own log.

**DHCP client** — NM has an *internal* DHCP client (the default on Fedora) and can also use `dhclient` or `dhcpcd` via config. Internal is lighter; dhclient is battle-tested. The choice is in `/etc/NetworkManager/NetworkManager.conf` under `[main] dhcp=`.

**systemd-resolved** — the stub resolver listening on 127.0.0.53. NM pushes per-link DNS config to resolved via D-Bus (`org.freedesktop.resolve1`). This is why `/etc/resolv.conf` on a modern Fedora box just has `nameserver 127.0.0.53`: the real per-interface DNS servers live inside resolved, queried with `resolvectl status`.

```bash
resolvectl status
# see which DNS servers are active on which interfaces, which domains each covers
```

If NM is configured without resolved integration (`/etc/NetworkManager/NetworkManager.conf` has `dns=none` or `dns=default`), it writes `/etc/resolv.conf` directly. Most modern configs use `dns=systemd-resolved`.

**firewalld** — each connection has a `connection.zone` setting. When activating, NM tells firewalld "put this interface in that zone." This is how your AP vif ended up in `trusted` after you set `nmcli connection modify myap connection.zone trusted`. Without firewalld, the setting is silently ignored.

**systemd-networkd** — can run alongside NM, but usually one or the other. NM by default will refuse to manage interfaces already being managed by networkd, and vice versa. Don't run both on the same interface.

**VPN plugins** — OpenVPN, WireGuard, strongswan, OpenConnect, libreswan all have NM plugins (`NetworkManager-openvpn`, etc.) that expose VPN config as ordinary NM connections. The plugin talks to the underlying VPN daemon; NM treats the VPN as just another connection type. WireGuard is special — its kernel support means NM manages it natively, no plugin needed.

**mDNS (Avahi)** — Avahi runs independently but cares about NM: when a new connection activates, Avahi rescans interfaces and starts advertising/listening there.

## Configuration files and the precedence rules

```
/usr/lib/NetworkManager/conf.d/*.conf    distro defaults
/run/NetworkManager/conf.d/*.conf         runtime (transient)
/etc/NetworkManager/conf.d/*.conf         admin overrides — edit here
/etc/NetworkManager/NetworkManager.conf   main config
```

NM reads all `.conf` files in those directories in sort order, later wins. Put your overrides in `/etc/NetworkManager/conf.d/` rather than editing the main file.

Common things you'll set:

```ini
# /etc/NetworkManager/conf.d/99-custom.conf
[main]
dhcp=internal
dns=systemd-resolved

[connectivity]
# used for "online" detection; fires connectivity-change dispatcher events
uri=http://fedoraproject.org/static/hotspot.txt
response=OK

[device]
# Per-device overrides by match expression
match-device=interface-name:wlp194s0
wifi.powersave=2

[keyfile]
# What interfaces should NM *not* touch?
unmanaged-devices=interface-name:veth*;interface-name:docker*
```

After editing: `sudo systemctl reload NetworkManager` (gentler than restart; doesn't drop existing connections).

## Device management scope: what NM touches

NM decides per-device whether to manage it. The rules, in order:

1. **Device is explicitly unmanaged** via config (`unmanaged-devices=` or a keyfile for the device). Won't touch it.
2. **Device is managed by another daemon** — if systemd-networkd or something else has claimed it, NM backs off.
3. **Device has `NM_UNMANAGED=1`** set via udev. Won't touch it.
4. **Device type is in NM's supported list.** Ethernet, WiFi, WireGuard, bridges, bonds, VLANs, teams, infiniband, PPP, tun/tap, macvlan, vrf, ovs-bridge, etc.
5. Otherwise: managed.

Container and VM virtual interfaces are typically excluded by default (docker0, veth*, virbr*, cali*). This is in `/usr/lib/NetworkManager/conf.d/` on most distros — take a look.

```bash
cat /usr/lib/NetworkManager/conf.d/*.conf
# Shows the unmanaged-devices rules your distro ships
```

If you need to unmanage something NM is touching, drop a config:

```bash
sudo tee /etc/NetworkManager/conf.d/99-unmanaged.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:tap0;interface-name:myveth*
EOF
sudo systemctl reload NetworkManager
```

This is exactly what you did for `ap0` earlier when building a manual hotspot.

## Activation: the full path traced

Putting it together, here's what "click Connect to WiFi X" actually does.

1. **GUI client** (gnome-shell's network indicator) is an NM D-Bus client. It calls `ActivateConnection(connection=<path>, device=<path>, specific_object=<ap-path>)`.
2. **Polkit** checks whether your session is allowed. Usually yes for local active session.
3. **NM's state machine** moves the device: `disconnected` → `prepare`.
4. **Link layer**: if ethernet, bring up and wait for carrier; if WiFi, spawn/communicate with wpa_supplicant.
5. **Authentication** (WiFi): wpa_supplicant associates with the AP, does 4-way handshake using the PSK. If it needs a secret and none is stored: `need-auth` state, NM calls registered agents over D-Bus, user is prompted, secret flows back.
6. **IP config**: NM starts its DHCP client on the interface. DISCOVER, OFFER, REQUEST, ACK. Gets IP, gateway, DNS servers, option 121 (classless static routes), option 15 (domain name).
7. **Route installation**: NM writes routes via netlink. Default gateway via the DHCP-supplied gateway. Per-interface routes. Metric prioritization between multiple active connections.
8. **DNS push**: NM tells systemd-resolved via D-Bus: "for interface wlp194s0, DNS servers are X Y, search domain is home.arpa". resolved updates its per-link config.
9. **Firewall zone**: NM tells firewalld: "put wlp194s0 in zone FedoraWorkstation" (or whatever the connection specifies).
10. **Connectivity check**: NM fetches the connectivity URL over the new connection. Result (`full`, `portal`, `limited`, `none`) is stored as the `Connectivity` property.
11. **Dispatcher scripts** in `/etc/NetworkManager/dispatcher.d/` run with `up` event.
12. **State change signal** fires on D-Bus. Everyone subscribed (gnome-shell, mail clients, sync daemons, the `systemd-networkd-wait-online.service`-like equivalents) redraws/reacts.

Every step logged in `journalctl -u NetworkManager`. This single trace is the best thing to keep in mind when debugging — when something fails, figure out which step is the one that broke.

## Special connection types worth knowing

**Ethernet** — boring by design. Plugs in, gets DHCP, works. Creates an auto-generated "Wired connection 1" profile on first use.

**WiFi client (`mode=infrastructure`)** — the normal WiFi case. SSID, security, go.

**WiFi AP (`mode=ap`)** — hotspot mode. Combined with `ipv4.method=shared` you get NAT+DHCP+DNS out of the box. The `nmcli device wifi hotspot` shortcut builds this for you.

**WiFi Ad-Hoc (`mode=adhoc`)** — peer-to-peer WiFi, largely dead in favor of WiFi Direct / hotspot sharing. Rare.

**Mesh (`mode=mesh`)** — 802.11s mesh networks. Niche but supported.

**Bond / Team / Bridge / VLAN** — virtual interfaces. You create the master connection (the bridge/bond itself), then individual slave connections that say "my `connection.master` is this bridge." NM activates them in order.

```bash
nmcli connection add type bridge con-name br0 ifname br0 \
    ipv4.method manual ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1
nmcli connection add type ethernet con-name br0-eno1 ifname eno1 master br0
nmcli connection up br0
```

**WireGuard** — natively supported (no plugin). Entire peer table is in the connection settings. Routes auto-computed from `allowed-ips` unless you override.

**VPN (plugin-based)** — OpenVPN, OpenConnect, strongswan, libreswan, etc. Install the NM plugin, the VPN type appears. Can be layered on top of another connection (the "secondary connection" mechanism: VPN-on-WiFi activates WiFi first, then VPN).

**GSM/WWAN** — cellular modems. Needs ModemManager daemon running alongside.

**PPP / PPPoE** — DSL and legacy. Still works.

**Loopback** — since NM 1.42+, NM can manage the loopback interface too (for setting extra addresses etc.).

## Tutorial: trace a real session on your machine

```bash
# 1. Current state
nmcli general status
nmcli device status
nmcli connection show --active

# 2. What's the primary connection?
nmcli -g PRIMARY-CONNECTION general status
nmcli -g connection.id connection show <uuid-of-primary>

# 3. What did DHCP give us?
nmcli -f IP4 device show wlp194s0
nmcli -f DHCP4 device show wlp194s0        # full DHCP option set

# 4. What DNS is in effect?
resolvectl status wlp194s0

# 5. What firewall zone is the interface in?
firewall-cmd --get-zone-of-interface=wlp194s0

# 6. What routes exist?
ip route
ip -6 route

# 7. Connectivity check result
nmcli -g CONNECTIVITY general status

# 8. Watch live
nmcli monitor &
sudo journalctl -u NetworkManager -f &
# Now disconnect/reconnect WiFi from the menu; watch both streams
```

This will teach you more about NM in five minutes than an hour of reading.

## Things that trip people up

**Confusing device and connection.** "I deleted the connection and now my WiFi doesn't work" — deleting the profile doesn't remove the device. The device still exists; it just has no active profile. Create a new one (or `nmcli device wifi connect` recreates one).

**Expecting `modify` to take effect immediately.** Changes to a live connection require reactivation. `nmcli device reapply <dev>` handles most changes without a full down/up cycle.

**`auto` + `manual` DNS confusion.** If `ipv4.method=auto`, you get DHCP's DNS by default. Setting `ipv4.dns` adds to the list but doesn't replace. To *replace*, also set `ipv4.ignore-auto-dns yes`. Same for routes (`ipv4.ignore-auto-routes`).

**`connection.autoconnect`.** Default is `yes`. If you want a saved profile NM knows about but doesn't grab on boot, set `autoconnect no`. Hot pain point: your hotspot profile auto-activates on boot, stealing the radio from your client profile.

**Connection priority**. With multiple autoconnect-yes profiles, NM picks by priority (`connection.autoconnect-priority`, higher wins) and timestamp (more recently used wins). Set explicit priorities if it matters.

**Hotspot vs. client on one radio.** As we spent much time on: NM's hotspot mode wants to use the radio exclusively. Whether concurrent AP+STA works depends on the driver (MT7925 says yes, but buggily; Intel ax210 is better; some are flat-out no).

**Wi-Fi powersave tanking latency.** By default NM enables `wifi.powersave=3`. For gaming or low-latency: `nmcli connection modify <name> 802-11-wireless.powersave 2` (disabled) or set globally in conf.d. Trade-off: some battery life.

**MAC randomization by default.** Since ~1.16, NM randomizes WiFi MAC per connection for privacy. Breaks static-lease setups. `802-11-wireless.cloned-mac-address permanent` disables for that connection; `wifi.scan-rand-mac-address=no` in config disables scan-time randomization globally.

**Secret storage flags on headless systems.** Agent-owned secrets need an agent. No desktop → no agent → activation fails silently. Use `psk-flags 0` for system-stored, or run `nmcli connection up --ask` for interactive prompting.

**Dispatcher scripts hanging state transitions.** Scripts run synchronously. A slow dispatcher script delays `up`/`down` from being reported. Keep them fast.

**Multiple active profiles on one device.** By design, one profile per device at a time. Attempting to activate a second deactivates the first.

**`nmcli` device reapply doesn't handle all changes.** Switching encryption types, changing from dhcp to static — these require full down/up. When in doubt, `down` then `up`.

**NM and Docker/Podman/libvirt.** Container networking creates virtual interfaces NM shouldn't touch. Distros ship unmanaged rules for `docker0`, `veth*`, `virbr*`, etc. If you add custom bridge names, extend the `unmanaged-devices=` list.

**systemd-networkd-wait-online fighting NM.** Servers sometimes have both installed; the wait-online service from networkd sits forever because networkd isn't actually managing anything. Disable the wrong one. For NM: `systemctl enable NetworkManager-wait-online.service`.

## How NM fits with everything else

- **vs kernel + `ip` / `iw` / `nft`.** NM is userspace. It drives the kernel via netlink (for links, addresses, routes), wpa_supplicant for 802.11 auth, and its own DHCP client. When NM and `ip` fight (you `ip route add` something NM thinks it owns), NM may overwrite.
- **vs systemd-networkd.** Alternative daemon. Same substrate, different philosophy — networkd is declarative, doesn't do D-Bus, wpa_supplicant isn't integrated.
- **vs wpa_supplicant.** NM uses it for WiFi auth. You can drive wpa_supplicant directly if you prefer; NM just makes it easy.
- **vs iwd.** Intel's alternative WiFi daemon, competing with wpa_supplicant. Some distros (crostini, some minimal setups) use iwd + systemd-networkd. NM can also use iwd instead of wpa_supplicant if configured.
- **vs resolved / firewalld.** Integrations, not rivals. NM pushes configuration into them.
- **vs the desktop.** GNOME / KDE ship NM applets that are D-Bus clients of NM. Nothing about the GUI is privileged; it all goes through NM, which goes through polkit.
- **vs ModemManager.** Separate daemon for cellular/WWAN. NM talks to MM over D-Bus to manage mobile connections.

## Quick reference card

```bash
# Overview
nmcli                                       # general status
nmcli general status
nmcli device status
nmcli connection show
nmcli connection show --active
nmcli radio                                 # wifi/wwan state

# Devices
nmcli device wifi list
nmcli device wifi rescan
nmcli device wifi connect "SSID" password "pw"
nmcli device disconnect <iface>
nmcli device reapply <iface>
nmcli device show <iface>                   # everything about a device

# Connections
nmcli connection add type <TYPE> con-name <NAME> ifname <IFACE> [settings...]
nmcli connection modify <NAME> <field> <value>
nmcli connection delete <NAME>
nmcli connection up <NAME>
nmcli connection down <NAME>
nmcli connection show <NAME>
nmcli connection reload                     # re-read on-disk files
nmcli connection edit <NAME>                # interactive editor

# Secrets
nmcli --ask connection up <NAME>            # prompt for missing secrets

# Hotspot shortcut
nmcli device wifi hotspot ifname <iface> ssid <NAME> password <PW>

# Monitoring
nmcli monitor
nmcli device monitor
journalctl -u NetworkManager -f
busctl monitor org.freedesktop.NetworkManager

# Config
$EDITOR /etc/NetworkManager/NetworkManager.conf
$EDITOR /etc/NetworkManager/conf.d/99-local.conf
sudo systemctl reload NetworkManager
sudo ls /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/<name>.nmconnection

# Scripting-friendly output
nmcli -t -f NAME,UUID,TYPE connection show
nmcli -g STATE device status
```

## Where to learn more

- **`man NetworkManager`, `man nmcli`, `man nm-settings-nmcli`, `man NetworkManager.conf`, `man NetworkManager-dispatcher`** — the reference. `nm-settings-nmcli(5)` in particular lists every settable field for every connection type. Comprehensive, if dense.
- **`nmcli connection show <name>`** — the fastest way to remember what settings exist; all fields shown, even the empty ones.
- **D-Bus introspection** — `busctl tree org.freedesktop.NetworkManager` reveals the API surface. For scripts, sometimes easier than `nmcli`.
- **The upstream project pages** at `https://networkmanager.dev/` (news, migration notes, blog posts from maintainers).
- **Dispatcher script examples** on the project's GitLab — useful patterns for "do X on connect".

The mental model to carry: **NetworkManager is a D-Bus daemon that owns network interfaces (devices), applies saved configuration profiles (connections) to them, coordinates with wpa_supplicant, its DHCP client, systemd-resolved, firewalld, and polkit to handle authentication, IP configuration, DNS, firewall zones, and privilege, and exposes all of this as a uniform API for GUIs, CLIs, and scripts.** Once the device/connection distinction is solid and the activation lifecycle (prepare → config → need-auth → ip-config → activated) is in your head, almost every piece of NM's sprawl slots into place, and the deep coordination with the rest of the userspace stack becomes a feature rather than a mystery.