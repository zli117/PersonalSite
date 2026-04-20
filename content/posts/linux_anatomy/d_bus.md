---
date: '2026-04-19T20:55:22-07:00'
draft: true
title: 'DBus'
---
## What D-Bus is and why it exists

By the early 2000s, Linux desktops had a problem: every component needed to talk to every other component. The screensaver needed to know when you're idle. The volume applet needed to ask PulseAudio for the current level. The network applet needed to talk to whatever managed networking. The media player needed to inhibit screen lock during video playback.

What existed: CORBA (heavyweight and hated), ad-hoc Unix sockets (everyone rolled their own protocol), X11 atoms (abused as IPC), DCOP in KDE and Bonobo in GNOME (desktop-specific, incompatible). Every component pair invented or picked their own IPC.

**D-Bus** (started 2002 as a freedesktop.org project) proposed: one message-passing bus that any process can join, with a standard protocol, standard type system, and a broker that routes messages by well-known names. Lightweight enough for desktop use, structured enough for system services, language-agnostic via bindings.

The bet paid off. Today D-Bus is how:

- `systemctl` talks to PID 1 (`org.freedesktop.systemd1`)
- `nmcli` talks to NetworkManager (`org.freedesktop.NetworkManager`)
- `bluetoothctl` talks to BlueZ (`org.bluez`)
- Your desktop talks to UPower, [[logind]], fwupd, colord, gnome-shell, KWin...
- [[Polkit]] enforces privilege policies for all of the above

**D-Bus is not:**

- A network protocol — it's local IPC (though there's dbus-broker-over-TCP for weird cases, rarely used).
- A high-performance transport — for bulk data (audio, video frames), use shared memory or direct sockets.
- A service discovery protocol — it's a message bus with well-known names, not Zeroconf.

## The core architecture

```
  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │  systemctl    │    │ NetworkMgr    │    │  bluetoothctl │
  │  (client)     │    │  (service)    │    │   (client)    │
  └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
          │ Unix socket        │                    │
          │ (auth + framing)   │                    │
          └────────┬───────────┴────────────────────┘
                   ▼
          ┌─────────────────┐
          │  dbus-broker    │    ← routes messages by name/path/interface
          │  (the bus)      │
          └────────┬────────┘
                   ▲
                   │
          ┌────────┴────────┐
          │    systemd      │
          │    BlueZ        │
          │    ...          │
          └─────────────────┘
```

**The broker.** One daemon process that every D-Bus participant connects to. It routes messages: "deliver this method call to whoever owns the name `org.freedesktop.NetworkManager`." Modern Fedora runs `dbus-broker` (Red Hat's reimplementation, faster); traditional systems run the reference `dbus-daemon`. Wire protocol is identical.

**Two buses per system, usually:**

- **System bus** — one instance, root-privileged, for system services. Socket at `/run/dbus/system_bus_socket`. NetworkManager, systemd, BlueZ, UPower, firewalld live here.
- **Session bus** — one per logged-in user session, for desktop apps. Socket at `$XDG_RUNTIME_DIR/bus` (typically `/run/user/<uid>/bus`). GNOME Shell, notifications, media player controls, window manager extensions live here.

Why two: security boundaries. Session bus talks on behalf of your user; system bus requires careful policy to let unprivileged users poke at root-owned services (this is where polkit comes in).

You can create **private buses** too, scoped to a container, a test harness, or a specific application. Every bus is just a dbus-broker listening on a socket; which one you connect to is up to you (env var `DBUS_SESSION_BUS_ADDRESS`, etc.).

## The object model

D-Bus exposes services using four layers of identifier, nested like a filesystem-plus-namespacing scheme:

```
  Bus name:        org.freedesktop.NetworkManager           ← "who"
  Object path:     /org/freedesktop/NetworkManager/Devices/2   ← "what"
  Interface:       org.freedesktop.NetworkManager.Device     ← "which API"
  Member:          State (property), Disconnect() (method), StateChanged (signal)   ← "which operation"
```

Stepping through each:

**1. Bus name** — who owns this endpoint. Two flavors:

- **Well-known names** like `org.freedesktop.NetworkManager` — services claim these by name, clients address them by name. Reverse-DNS convention to avoid collisions.
- **Unique names** like `:1.42` — every connection to the bus gets one automatically, assigned by the broker. Services own well-known names _in addition to_ their unique name.

When you send a message to `org.freedesktop.NetworkManager`, the broker looks up who currently owns that well-known name and forwards to their unique connection.

**2. Object path** — which object within the service. Paths look like filesystem paths but they're just namespacing. A service exposes many objects: NetworkManager has `/org/freedesktop/NetworkManager` (the manager itself), `/org/freedesktop/NetworkManager/Devices/0`, `/1`, `/2` (each device), `/Settings`, and so on.

**3. Interface** — which named API you're using. An object can implement multiple interfaces. A NetworkManager device object implements `org.freedesktop.NetworkManager.Device` (the base) plus `org.freedesktop.NetworkManager.Device.Wireless` (if it's a WiFi device) plus the standard ones `org.freedesktop.DBus.Properties`, `org.freedesktop.DBus.Introspectable`, `org.freedesktop.DBus.Peer`.

**4. Member** — the specific thing you're operating on. Three member kinds:

- **Methods** — call-and-reply. `Disconnect()`, `GetDevices()`, `ActivateConnection(settings, device, specific_object)`. Synchronous from the protocol's perspective (you get a reply or an error).
- **Properties** — typed named values. `State`, `Ipv4Address`, `Managed`. Read via the standard `org.freedesktop.DBus.Properties.Get` method; changes emit a `PropertiesChanged` signal.
- **Signals** — broadcasts. `StateChanged(oldState, newState)`, `DeviceAdded(path)`. No reply expected. Clients subscribe to signals they care about.

The combination `(bus name, object path, interface, member)` uniquely addresses any operation.

## The standard interfaces

Every D-Bus object is expected to implement a few standard interfaces:

- `org.freedesktop.DBus.Introspectable` — `Introspect()` returns XML describing the object's interfaces and members. This is how tools like `busctl introspect` learn what's available without prior knowledge.
- `org.freedesktop.DBus.Properties` — `Get(interface, prop)`, `Set(interface, prop, value)`, `GetAll(interface)`, plus the `PropertiesChanged` signal.
- `org.freedesktop.DBus.Peer` — `Ping()`, `GetMachineId()`. Lightweight liveness.
- `org.freedesktop.DBus.ObjectManager` (optional, on container objects) — `GetManagedObjects()`, `InterfacesAdded`, `InterfacesRemoved`. Lets a client enumerate a service's whole object tree in one call and subscribe to add/remove signals.

The ObjectManager pattern is important: instead of every service inventing a "list my devices" method, services that manage a collection implement ObjectManager at some root path, and clients use the standard methods to enumerate and watch.

## The type system

D-Bus is strongly typed. Every message's arguments and return values have declared types using single-character codes:

|Code|Type|
|---|---|
|`y`|BYTE|
|`b`|BOOLEAN|
|`n` / `q`|INT16 / UINT16|
|`i` / `u`|INT32 / UINT32|
|`x` / `t`|INT64 / UINT64|
|`d`|DOUBLE|
|`s`|STRING (UTF-8)|
|`o`|OBJECT PATH|
|`g`|SIGNATURE (a type code string)|
|`v`|VARIANT (any type, self-describing)|
|`a<T>`|ARRAY of T|
|`(TT...)`|STRUCT|
|`a{KV}`|DICT (really array of key-value pairs)|
|`h`|UNIX FD (file descriptor)|

So `a{sv}` is a dictionary from string to variant — the ubiquitous "properties dict" pattern. `ao` is an array of object paths. `(si)` is a struct of string and int32.

You'll see signatures in introspection XML and in tool output:

```
method org.fd.NM.ActivateConnection(ooo) -> o
#                                    ^^^    ^
#           connection path, device path, specific object path → activation path
```

## Hands-on: inspecting what's on the bus

The `busctl` tool (from [[systemd]]) is the friendliest modern D-Bus explorer. `dbus-send` is the classical alternative; `gdbus` ships with GLib.

**Who's on the bus:**

```bash
busctl list                                 # system bus services
busctl --user list                          # session bus services
```

You'll see rows like:

```
NAME                                TYPE  PID  PROCESS          USER
org.freedesktop.NetworkManager      -     987  NetworkManager   root
org.freedesktop.systemd1            -     1    systemd          root
org.bluez                           -     1234 bluetoothd       root
:1.42                               -     1234 bluetoothd       root     ← unique name
```

The dashes under TYPE mean "activatable on demand if called" vs `activatable` / `running` hints in some outputs.

**Explore a service's object tree:**

```bash
busctl tree org.freedesktop.NetworkManager
```

```
└─ /org
  └─ /org/freedesktop
    └─ /org/freedesktop/NetworkManager
      ├─ /org/freedesktop/NetworkManager/ActiveConnection
      │ └─ /org/freedesktop/NetworkManager/ActiveConnection/1
      ├─ /org/freedesktop/NetworkManager/Devices
      │ ├─ /org/freedesktop/NetworkManager/Devices/1
      │ ├─ /org/freedesktop/NetworkManager/Devices/2
      │ └─ /org/freedesktop/NetworkManager/Devices/3
      └─ /org/freedesktop/NetworkManager/Settings
```

**Introspect a specific object** — see what interfaces and members it exposes:

```bash
busctl introspect org.freedesktop.NetworkManager /org/freedesktop/NetworkManager/Devices/2
```

```
NAME                                TYPE      SIGNATURE RESULT/VALUE
org.freedesktop.NetworkManager.Device interface -        -
.Disconnect                         method    -         -
.Reapply                            method    a{sa{sv}}ut -
.ActiveConnection                   property  o         "/org/freedesktop/..."
.Interface                          property  s         "wlp194s0"
.State                              property  u         100
.StateChanged                       signal    uuu       -
```

You get the full API surface of the object, zero documentation needed.

**Get a property:**

```bash
busctl get-property org.freedesktop.NetworkManager \
  /org/freedesktop/NetworkManager/Devices/2 \
  org.freedesktop.NetworkManager.Device Interface

# s "wlp194s0"
```

**Call a method:**

```bash
busctl call org.freedesktop.NetworkManager \
  /org/freedesktop/NetworkManager \
  org.freedesktop.NetworkManager GetDevices
# ao 3 "/org/freedesktop/NetworkManager/Devices/1" "/org/freedesktop/NetworkManager/Devices/2" ...
```

Format: `busctl call <service> <path> <interface> <method> [signature args...]`.

**Watch signals and method calls live** — invaluable for debugging:

```bash
busctl monitor org.freedesktop.NetworkManager
```

Everything going to or from NetworkManager streams by in real time. Kill WiFi, reconnect, click around in the network applet — you'll see the whole dance.

Plain `busctl monitor` watches all messages on the bus, very verbose.

## Hands-on: a concrete walkthrough

Let's say you want to see your laptop's battery level and watch it change over time.

**Find the relevant service:**

```bash
busctl list | grep -i power
# org.freedesktop.UPower  ...
```

**Explore its tree:**

```bash
busctl tree org.freedesktop.UPower
# └─ /org/freedesktop/UPower
#   ├─ /org/freedesktop/UPower/devices
#   │ ├─ /org/freedesktop/UPower/devices/battery_BAT0
#   │ ├─ /org/freedesktop/UPower/devices/line_power_AC
#   │ └─ /org/freedesktop/UPower/devices/DisplayDevice
```

**Introspect the battery:**

```bash
busctl introspect org.freedesktop.UPower /org/freedesktop/UPower/devices/battery_BAT0
```

Lots of output. Notable properties: `Percentage`, `State`, `TimeToEmpty`, `TimeToFull`, `EnergyRate`.

**Read a property:**

```bash
busctl get-property org.freedesktop.UPower \
  /org/freedesktop/UPower/devices/battery_BAT0 \
  org.freedesktop.UPower.Device Percentage
# d 73.0
```

**Watch for changes** — UPower emits `PropertiesChanged` via the standard interface:

```bash
busctl monitor org.freedesktop.UPower
```

Unplug AC or plug it in, percentages ticking, and you see signals like:

```
Type=signal ... Interface=org.freedesktop.DBus.Properties Member=PropertiesChanged
  string "org.freedesktop.UPower.Device"
  array of dict entry(string, variant) {
    "Percentage" -> variant double 72.0
    "State" -> variant uint32 2
    ...
  }
```

This same pattern applies to _everything_ on D-Bus. List services, pick one, introspect, call or watch. Once you know the rhythm, most system services become directly scriptable.

## Service activation

Services don't have to be running for clients to address them. If a client sends a method call to `org.freedesktop.NetworkManager` and nobody owns that name yet, the broker checks its **service files** (`.service` files under `/usr/share/dbus-1/system-services/` and `/usr/share/dbus-1/services/`) — each declares a well-known name and how to start the service.

If found, the broker starts the service (or asks systemd to), waits for it to claim the name, then delivers the message. This is **D-Bus activation**, analogous to systemd's socket activation but for D-Bus services.

Combined with `Type=dbus` in a systemd unit, you get: "service starts when anyone on D-Bus tries to talk to it, stops when idle, transparent to clients."

## Signals and subscriptions

Signals are broadcasts — the sender fires and forgets; the broker routes them to any connections that have added a matching rule. Unlike method calls, signals have no reply.

To receive signals, a client adds a **match rule** describing which signals it cares about:

```
type='signal',interface='org.freedesktop.NetworkManager',member='StateChanged'
```

Match rules can filter on sender, interface, member, path, arguments. Clients typically use bindings that hide this — `gdbus monitor`, GLib's `g_dbus_connection_signal_subscribe`, Python's `dbus-next.add_message_handler`.

`busctl monitor <service>` adds a broad match rule behind the scenes.

## Security: polkit and the privilege boundary

Here's the tricky problem. System services on the system bus run as root. Ordinary users need to invoke some of their methods (e.g., connect to WiFi, suspend the machine, mount a disk) but not others (e.g., shut down the system from a screenshare attendee's account).

**Polkit** (formerly PolicyKit) is the solution. When a user calls a method on a privileged service, the service asks polkit: "is this user allowed to do X?" Polkit consults rule files, possibly prompts the user for authentication via an agent (GNOME's polkit dialog, KDE's equivalent, or a terminal `pkexec` prompt), and replies allow/deny.

The call chain:

```
nmcli (unprivileged)
  → D-Bus system bus
  → NetworkManager (root)
  → polkit: "may UID 1000 call ActivateConnection for admin-only profile?"
  → polkit checks /usr/share/polkit-1/actions/*.policy and /etc/polkit-1/rules.d/*.rules
  → polkit may invoke polkit-agent in user's session to prompt for password
  → allow / deny
  → NetworkManager proceeds or returns AuthorizationFailed
```

Polkit rules are JavaScript (controversial choice, but flexible). System-wide rules live in `/usr/share/polkit-1/actions/*.policy` (vendor-declared actions) and `/etc/polkit-1/rules.d/*.rules` (admin overrides).

Example rule — let anyone in the `wheel` group suspend without password:

```javascript
// /etc/polkit-1/rules.d/49-nopasswd-suspend.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.login1.suspend" &&
        subject.isInGroup("wheel")) {
        return polkit.Result.YES;
    }
});
```

See what actions a service declares:

```bash
ls /usr/share/polkit-1/actions/
pkaction --verbose --action-id org.freedesktop.NetworkManager.network-control
```

## D-Bus in practice: three patterns

**1. The service and its clients.** You write a daemon that owns a well-known name, exposes objects, accepts method calls, emits signals. Clients (other daemons, GUIs, CLIs) connect and interact. This is how NetworkManager, BlueZ, UPower, systemd itself are built.

**2. The control-plane pattern.** Something complex is happening (systemd managing units, NetworkManager managing connections, BlueZ managing devices). D-Bus is the control plane — the API other components use to drive it. The data plane (actual packets, audio frames, disk I/O) bypasses D-Bus entirely; D-Bus is for coordination and state.

**3. The portal pattern.** Flatpak apps run sandboxed and can't directly touch the filesystem or hardware. Instead, they call **xdg-desktop-portal** over the session bus, which mediates access (showing "this app wants to open a file" dialogs, etc.). This is how modern Linux sandboxing works — D-Bus is the only channel out of the sandbox, with the portal enforcing policy.

## Tutorial: writing a tiny D-Bus service

Let's build a minimal session-bus service in Python that exposes a greeter, then call it from another process. Uses the `dbus-next` library (async, modern).

**Install:**

```bash
pip install --user dbus-next
```

**Server** (`greeter-server.py`):

```python
import asyncio
from dbus_next.aio import MessageBus
from dbus_next.service import ServiceInterface, method, signal
from dbus_next import BusType

class Greeter(ServiceInterface):
    def __init__(self):
        super().__init__('com.example.Greeter')
        self._count = 0

    @method()
    def Hello(self, name: 's') -> 's':
        self._count += 1
        self.GreetingSent(name)
        return f'Hello, {name}!'

    @signal()
    def GreetingSent(self, name: 's') -> 's':
        return name

async def main():
    bus = await MessageBus(bus_type=BusType.SESSION).connect()
    iface = Greeter()
    bus.export('/com/example/Greeter', iface)
    await bus.request_name('com.example.Greeter')
    print('Service running as com.example.Greeter; Ctrl-C to stop.')
    await asyncio.Future()  # run forever

asyncio.run(main())
```

Run it:

```bash
python greeter-server.py
```

Now from another terminal, poke it with `busctl`:

```bash
busctl --user list | grep example
# com.example.Greeter ...

busctl --user introspect com.example.Greeter /com/example/Greeter

busctl --user call com.example.Greeter /com/example/Greeter \
  com.example.Greeter Hello s "ZL"
# s "Hello, ZL!"

busctl --user monitor com.example.Greeter &
# Now call again, watch the GreetingSent signal fire
```

That's the whole cycle — claim a name, export an object implementing an interface, respond to calls, emit signals. The bindings handle type marshaling, dispatching, signal delivery.

**Client** (`greeter-client.py`):

```python
import asyncio
from dbus_next.aio import MessageBus
from dbus_next import BusType

async def main():
    bus = await MessageBus(bus_type=BusType.SESSION).connect()
    introspection = await bus.introspect('com.example.Greeter', '/com/example/Greeter')
    obj = bus.get_proxy_object('com.example.Greeter', '/com/example/Greeter', introspection)
    greeter = obj.get_interface('com.example.Greeter')

    # Subscribe to the signal
    greeter.on_greeting_sent(lambda name: print(f'[signal] greeted: {name}'))

    # Call the method
    reply = await greeter.call_hello('world')
    print(reply)

    await asyncio.sleep(1)  # give signal time to arrive

asyncio.run(main())
```

In most languages, bindings turn D-Bus into "call a method on an object" with introspection handling the plumbing.

## Making it a proper session service

Right now your service only exists while you run it manually. To make it **activatable** on the session bus:

**`~/.local/share/dbus-1/services/com.example.Greeter.service`:**

```
[D-BUS Service]
Name=com.example.Greeter
Exec=/home/you/greeter-server.py
```

Now `busctl --user call com.example.Greeter ...` starts the service automatically if it's not running. For production, back it with a proper systemd user unit (`Type=dbus`, `BusName=com.example.Greeter`) and let the bus and systemd coordinate.

## Language bindings

Every mainstream language has D-Bus bindings:

- **C/C++** — `libdbus` (reference, low-level, painful). `libsystemd`'s `sd-bus` (much nicer, modern, also works without systemd). `GDBus` (GLib, if you're doing GTK anyway).
- **Python** — `dbus-next` (async, pure Python, modern). `pydbus` (GLib-backed, simpler but GLib runtime). `dbus-python` (old, still works, synchronous).
- **Rust** — `zbus` (pure Rust, excellent async, widely used).
- **Go** — `godbus/dbus` (mature, reasonable).
- **Node.js** — `dbus-next` (also the name, different package).
- **Shell** — `busctl`, `dbus-send`, `gdbus` from the command line. Awkward for anything complex but fine for scripts.

`sd-bus` and `zbus` are the nicest to use. `libdbus` directly is only for legacy code.

## Debugging: the observatory

When something's wrong, D-Bus is eminently observable.

**Who owns what:**

```bash
busctl list
busctl list --user
```

**Watch specific traffic:**

```bash
busctl monitor org.freedesktop.NetworkManager       # one service
busctl monitor                                       # everything on system bus
busctl --user monitor                                # everything on session bus
busctl capture > trace.pcap                          # pcap format, readable in Wireshark
```

**Wireshark has a D-Bus dissector** — open a capture, see messages with types, signatures, and args pretty-printed. Extremely useful for chasing sporadic issues.

**Introspect any object:**

```bash
busctl introspect <service> <path>
```

**Call methods interactively:**

```bash
busctl call <service> <path> <interface> <method> <signature> <args>
```

**Check polkit decisions** when a method call is mysteriously denied:

```bash
journalctl -u polkit -f
pkaction --action-id <id> --verbose
```

**Verify a specific property works:**

```bash
busctl get-property <service> <path> <interface> <property>
```

## Performance and limits

D-Bus is not high-throughput. The broker routes every message, so there's double the socket I/O of a direct connection, plus message marshaling. Rough ballparks:

- Fine for thousands of calls per second.
- Not fine for audio samples, video frames, or bulk file transfers. Send a file descriptor (`h` type) and communicate out-of-band.
- Message size is capped (~128 MB default, often lower in practice). Big payloads should go through shm, files, or fd passing.

Modern `dbus-broker` is substantially faster than the old `dbus-daemon`, but the design is still optimized for control plane, not data plane.

**Fd passing** is a killer feature: you can send an open file descriptor over D-Bus as an `h` arg, and the receiver gets a dup of it. This is how PipeWire negotiates audio streams, how portals hand opened files to sandboxed apps, etc. Setup happens over D-Bus; the data flow goes through the fd directly, bypassing the bus.

## Things that trip people up

**Confusing buses.** `busctl` defaults to the system bus; `busctl --user` uses session. If a service you expect isn't showing up, check both. Apps running under Flatpak may not see your session bus at all without portal plumbing.

**Activation silence.** A method call against a non-running, non-activatable service just hangs until timeout. If nothing's happening, check that the service file exists and the `Exec=` is correct.

**Type mismatches.** Passing `i` where `u` is expected (int32 vs uint32) fails with a signature error. Bindings usually help; raw `busctl call` requires precision.

**Race conditions with signals.** Subscribe _before_ you call the method that triggers the signal, or you may miss it. Good bindings provide "subscribe and get current state" helpers to avoid this.

**Polkit cache.** Polkit caches auth decisions for a short window. If you change a rule and test, expect some confusion — `systemctl restart polkit` to force a flush.

**Name squatting.** Only one connection can own a well-known name at a time. If you start your service and get "Connection :1.42 is not allowed to own the service", either the policy (`/etc/dbus-1/system.d/*.conf`) forbids it, or someone else already owns it.

**Privilege assumptions.** Services on the _system bus_ need D-Bus policy XML files (`/usr/share/dbus-1/system.d/*.conf`) allowing the right users to talk to them. Session bus services don't need this — anyone in the session can talk to anyone else in the session.

## The bigger picture

A way to see D-Bus: it's the **control plane of the Linux desktop and service ecosystem**. Most coordination between system services — who's connected to WiFi, is the lid closed, is the user idle, did a Bluetooth device just pair, is fwupd in the middle of a firmware update — flows through D-Bus. Most introspection tools and integrations are D-Bus clients.

This means **learning D-Bus gives you direct access to most of the system's live state.** Scripting idiomatic solutions on modern Linux often means: find the right service, call the right method. Want to suspend programmatically? `busctl call org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager Suspend b true`. Want a notification? `busctl --user call org.freedesktop.Notifications /org/freedesktop/Notifications ...`. Want to know network state? Read properties from NetworkManager.

It's the universal joint connecting daemons, desktops, and users.

## Quick reference card

```bash
# Inspection
busctl list                              # system bus services
busctl --user list                       # session bus services
busctl tree <service>                    # object tree
busctl introspect <service> <path>       # interfaces + members of an object
busctl status <service>                  # PID, owner, credentials

# Property access
busctl get-property <service> <path> <interface> <property>
busctl set-property <service> <path> <interface> <property> <sig> <value>

# Method call
busctl call <service> <path> <interface> <method> [<sig> <args...>]

# Signal monitoring
busctl monitor [<service>]               # live traffic
busctl capture > trace.pcap              # pcap for Wireshark

# User bus (same commands with --user)
busctl --user ...

# Polkit
pkaction                                  # list actions
pkaction --action-id <id> --verbose
pkcheck --action-id <id> --process <pid>  # test authorization

# Service registration (declarative)
/usr/share/dbus-1/system-services/*.service    # system-bus activatable services
/usr/share/dbus-1/services/*.service           # session-bus activatable services
/etc/dbus-1/system.d/*.conf                    # system-bus policy
```

## Where to learn more

- **`man busctl`, `man sd-bus`** — terse, accurate.
- **`dbus-daemon(1)`, `dbus-broker(1)`** — the brokers.
- **freedesktop.org D-Bus spec** — `https://dbus.freedesktop.org/doc/dbus-specification.html`. Longer than it looks; the first quarter is enough to understand everything above. The rest covers wire protocol details you'll never write by hand.
- **NetworkManager and systemd D-Bus APIs** — read their introspection XML in `/usr/share/dbus-1/interfaces/` or via `busctl introspect`. Great real-world examples of interface design.
- **zbus Rust docs** — one of the best-written D-Bus binding docs out there, explains concepts clearly even if you're not using Rust.

The mental model to carry: **D-Bus is a broker-mediated message bus with typed addressing (bus name, object path, interface, member), supporting method calls, properties, and signals, plus on-demand service activation and standardized introspection.** Once you can `busctl tree` and `busctl introspect` any service and know what you're looking at, the Linux control plane is legible end to end.