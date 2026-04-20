---
date: '2026-04-19T22:05:08-07:00'
draft: true
title: 'Polkit'
---
## What polkit is and why it exists

Classical Unix privilege is binary: you're root (UID 0) or you're not. Programs needing privilege were traditionally setuid-root — a terrifyingly blunt mechanism where a single bug in the program elevates any user to full root. setuid binaries are notorious for escalation vulnerabilities; every one is a potential foothold.

As Linux became a desktop, the setuid model scaled horribly. Consider what a regular user legitimately needs to do:

- Connect to a WiFi network (NetworkManager, which runs as root).
- Mount a USB stick (udisks, which runs as root).
- Install a software update (PackageKit, running as root).
- Suspend the laptop (logind, running as root).
- Change the timezone (timedated, running as root).
- Start/stop a systemd service (systemd PID 1).

Every one of these involves a *system service* running as root that needs to decide whether the *unprivileged caller* is allowed to ask for this. Options that existed:

- **setuid wrappers** — make a small program setuid-root that calls the service. Multiplies attack surface per operation.
- **sudo with NOPASSWD** — open-ended; a user with sudo for `systemctl` can start *any* service, not just the ones they should.
- **Group-based ACLs** — `plugdev`, `networkmanager`, `video`, `audio`. Coarse; being in a group is all-or-nothing and non-contextual.
- **Service-specific ad-hoc auth** — each daemon invents its own. This was where Linux was headed: PulseAudio has X, NetworkManager has Y, HAL has Z, all different.

**polkit** (formerly **PolicyKit**, renamed in 2012) is the consolidated answer. Started ~2007 by David Zeuthen at Red Hat.

The core insight: **authorization is a separate question from authentication.** Authentication says "who are you" (PAM, login). Authorization says "are you allowed to do this specific thing right now" — and that question depends on a *lot* of context: your UID, your session state, whether your session is active on the local seat, whether you're remote, whether you're in a group, what time it is, what the specific operation is.

polkit is a system daemon that answers authorization questions on behalf of any service that asks. Services (running as root) hand polkit the caller's identity and the operation name; polkit consults its rules and returns allow / deny / ask-for-auth. If auth is needed, polkit talks to a user-facing **agent** that prompts the user. The service proceeds or denies based on the answer.

This design has two big payoffs:

1. **Centralized policy.** One place to decide "who can suspend the system?" Every service that cares asks the same daemon the same way.
2. **Context-aware.** "Allow if local active session" is a polkit primitive. "Allow only the user who's currently at the keyboard" is a five-word rule. No ad-hoc reinventions per service.

Almost every privileged system daemon on modern Linux — NetworkManager, systemd, logind, udisks, timedated, hostnamed, localed, upower, fwupd, PackageKit, gnome-control-center's backends, Cockpit — asks polkit before performing caller-requested privileged operations. You don't see it because when policy allows, it's silent. When policy requires auth, you see the polkit dialog.

## The central abstraction: actions, subjects, results

polkit's model has three core nouns.

### Action

An **action** is a named privileged operation, declared by a service. Reverse-DNS naming convention:

- `org.freedesktop.login1.suspend`
- `org.freedesktop.NetworkManager.settings.modify.system`
- `org.freedesktop.systemd1.manage-units`
- `org.freedesktop.UDisks2.filesystem-mount`
- `org.freedesktop.packagekit.package-install`

Actions are defined in XML files under `/usr/share/polkit-1/actions/*.policy`. Each service ships one file declaring its action vocabulary. A typical action declaration:

```xml
<action id="org.freedesktop.login1.suspend">
  <description>Suspend the system</description>
  <message>Authentication is required to suspend the system.</message>
  <defaults>
    <allow_any>auth_admin_keep</allow_any>
    <allow_inactive>auth_admin_keep</allow_inactive>
    <allow_active>yes</allow_active>
  </defaults>
</action>
```

The three `<allow_*>` elements say what polkit returns for three categories of caller (covered below). The values come from a small fixed vocabulary:

| Value | Meaning |
|---|---|
| `no` | Deny, no prompt. |
| `yes` | Allow, no prompt. |
| `auth_self` | Prompt for the *caller's* password. |
| `auth_admin` | Prompt for an administrator's password (root, or member of `wheel` depending on distro). |
| `auth_self_keep` | Same as `auth_self`, but remember for ~5 minutes. |
| `auth_admin_keep` | Same as `auth_admin`, remember for ~5 minutes. |

The `_keep` variants are what make GNOME Settings not prompt you twenty times in a row while adjusting system settings.

### Subject

A **subject** is who's asking. polkit identifies subjects in one of three ways:

- **Unix process** — a PID plus start-time token (to avoid PID reuse exploits). Rarely used directly; subjects are usually identified as…
- **Unix user** — by UID.
- **System bus name** — for D-Bus callers, the sender's bus name (`:1.42`). polkit can map this to the connecting UID and process.

When a service asks polkit "is this caller allowed to do action X?", polkit gets the subject's attributes: UID, groups, session, seat, active-or-not, local-or-remote. These attributes are what rules reason about.

### Result

What polkit returns: `yes`, `no`, or one of the `auth_*` variants. If an `auth_*` is returned, polkit also launches the authentication flow via the agent; the service waits for the answer, then gets `yes` or `no` as the final result.

## The three implicit categories: any / inactive / active

This is polkit's cleverest idea, and where most real policy is expressed.

When you look at an action's `<defaults>`, you see three slots:

- **`allow_any`** — the baseline for *any* subject (logged in or not, local or remote).
- **`allow_inactive`** — for subjects in a session that is not currently active on its seat (e.g., switched to another VT).
- **`allow_active`** — for subjects in a session that *is* active on its seat (i.e., the user physically at the keyboard right now).

These categories are defined in terms of **logind's session state**: polkit queries logind over D-Bus to determine whether the calling subject has an active local session. No active local session (SSH from across the internet, or a background daemon, or a screen-locked session) means `allow_inactive` or `allow_any` applies.

The typical pattern:

```
allow_active = yes          "the person at the keyboard can do this freely"
allow_inactive = auth_admin "someone on a locked session or other VT needs admin password"
allow_any = auth_admin      "remote SSH users need admin password"
```

For "suspend the system", this makes sense: the local user can just do it, other users need credentials to prove they're allowed. For more dangerous operations (`org.freedesktop.packagekit.package-install`), `allow_active` might still be `auth_admin_keep` because you want the user to confirm they really want to install software, but you don't want them annoyed with a second prompt for every subsequent install.

This simple three-way split handles a huge fraction of real policy without any custom rules.

## Rules: JavaScript for custom policy

Defaults in `.policy` files cover the common case. For anything more nuanced, polkit supports **rules written in JavaScript** (yes, really — using the Duktape embedded engine). Rules files live in:

```
/usr/share/polkit-1/rules.d/    distro-shipped rules, don't edit
/etc/polkit-1/rules.d/          admin overrides, edit here
```

Files are named `NN-description.rules`, sorted, evaluated in order. The first rule to return a definite result wins.

A rule file is a JavaScript file that registers one or more functions via `polkit.addRule(...)`. Each rule gets the `action` and `subject` objects and returns a `polkit.Result` value (or nothing, to pass to the next rule).

### Example: let wheel members suspend without password

```javascript
// /etc/polkit-1/rules.d/49-wheel-suspend.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.login1.suspend" &&
        subject.isInGroup("wheel")) {
        return polkit.Result.YES;
    }
});
```

Drop this in, reload is automatic (polkit watches the directory), subsequent suspend requests from wheel members return `yes` without any prompt.

### Example: let a specific user manage NetworkManager

```javascript
// /etc/polkit-1/rules.d/50-nm-netops.rules
polkit.addRule(function(action, subject) {
    if (action.id.indexOf("org.freedesktop.NetworkManager.") == 0 &&
        subject.user == "netops") {
        return polkit.Result.YES;
    }
});
```

### Example: require admin for any systemd unit management, period

```javascript
// /etc/polkit-1/rules.d/40-strict-systemd.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") {
        return polkit.Result.AUTH_ADMIN;
    }
});
```

This overrides the default for that action.

### Example: deny a specific dangerous action from non-local sessions

```javascript
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.packagekit.package-install" &&
        !subject.local) {
        return polkit.Result.NO;
    }
});
```

### What's in `action` and `subject`

`action` object:
- `action.id` — the action name.
- `action.lookup("key")` — look up a detail passed by the caller (some actions include e.g. the target unit name, the package being installed, the filesystem being mounted).

`subject` object:
- `subject.user` — username string.
- `subject.groups` — array of group names. Also `subject.isInGroup("wheel")`.
- `subject.pid` — caller's PID.
- `subject.seat` — e.g., `"seat0"` or `null`.
- `subject.session` — logind session ID.
- `subject.local` — boolean.
- `subject.active` — boolean (active on its seat).

### Rules are procedural

Rules can do anything JavaScript can. Read files. Parse environment. Call `polkit.log("message")` for debug logging. You can write stateful-ish rules that tolerate lookup failures, apply time-of-day checks, anything. Power and footgun.

The JavaScript choice has been controversial — many sysadmins would prefer pkla-style declarative config. The upside is flexibility; the downside is that a bad rule is a programming bug. A thrown exception in a rule is equivalent to "deny." Syntax errors in a rule file make polkit ignore the file with a warning in the journal.

## Agents: how prompts reach the user

When polkit wants to prompt for authentication, it calls the **authentication agent** registered for the subject's session. Every desktop environment ships one:

- **polkit-gnome-authentication-agent-1** (GNOME)
- **polkit-kde-authentication-agent-1** (KDE)
- **lxpolkit**, **mate-polkit**, **xfce-polkit**, etc. for other DEs
- **pkexec** itself, via its own terminal prompt, for console sessions

The agent registers itself with polkit at session start, targeting the caller's logind session. When polkit needs a prompt, it calls `BeginAuthentication` on the agent, which shows the dialog, runs the PAM stack internally, and reports back.

You can see your session's agent:

```bash
ps auxf | grep -i polkit-agent
# or look in your session scope:
systemctl status session-$XDG_SESSION_ID.scope | grep -i polkit
```

For headless systems or SSH sessions, no agent is running. If you try to do a polkit-gated operation, it fails because there's no one to prompt.

You can **temporarily register a terminal agent** for your current shell:

```bash
# Running a long-form PM command interactively from SSH?
pkttyagent --process $$ &
# Now polkit prompts appear in this terminal
# Kill it when done:
kill %1
```

`pkttyagent` is the TTY-based agent. It's what gets invoked automatically by `pkexec` (see below) for console auth.

## `pkexec`: the modern setuid replacement

`pkexec` runs a program as another user (root by default) with polkit mediating the authorization. It's the sanctioned replacement for setuid binaries and most `sudo` use cases on desktops.

```bash
pkexec some-program arg1 arg2
```

Flow:
1. `pkexec` asks polkit: "can UID X run `some-program` as root?"
2. polkit looks up the action: by default `org.freedesktop.policykit.exec`, with action attributes including the program path.
3. polkit applies policy. Default is `auth_admin_keep` — prompt for admin password, remember briefly.
4. Agent prompts; user enters password; PAM validates.
5. polkit returns `yes`; `pkexec` execs the program with root privs.

The key advantages over setuid:

- **No setuid bit** — the program isn't elevated per se; `pkexec` elevates it at run time after auth.
- **Per-program policy** — you can write a rule allowing one specific path to run without password for specific users.
- **Audit trail** — polkit logs the authorization decision to the journal.
- **Sandboxed child environment** — pkexec scrubs the environment (PATH, LD_*, etc.) to prevent obvious injection.

Example: allow a specific maintenance script to run without password for members of `ops`:

```javascript
// /etc/polkit-1/rules.d/60-ops-maintenance.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.policykit.exec" &&
        action.lookup("program") == "/usr/local/sbin/rotate-logs" &&
        subject.isInGroup("ops")) {
        return polkit.Result.YES;
    }
});
```

Now `pkexec /usr/local/sbin/rotate-logs` works without prompting for ops members.

A common `pkexec` use case is GUI apps that need root: the app re-execs itself via `pkexec` when the user clicks "Authorize," showing a GUI auth dialog. Cleaner than root-owned GUI processes.

### `pkexec` vs `sudo`

- **`sudo`** — older, ubiquitous, rule-based, config in `/etc/sudoers`. Works in headless contexts natively. Can grant very open-ended powers.
- **`pkexec`** — desktop-integrated (GUI prompts), polkit rules apply, scrubbed environment, per-program granularity. Less useful in headless contexts without an agent.

Both coexist. `sudo` is still the right tool for interactive admin shells and scripted pipelines. `pkexec` is the right tool for GUI apps that need to elevate, and for anywhere you want polkit's context-aware logic.

## Hands-on: inspecting policy

### List every registered action

```bash
pkaction                          # all action IDs, one per line
pkaction | grep -i network
```

### See details of one action

```bash
pkaction --action-id org.freedesktop.login1.suspend --verbose
# org.freedesktop.login1.suspend:
#   description:       Suspend the system
#   message:           Authentication is required to suspend the system.
#   vendor:            The systemd Project
#   vendor_url:        https://systemd.io
#   icon:
#   implicit any:      auth_admin_keep
#   implicit inactive: auth_admin_keep
#   implicit active:   yes
```

The three "implicit" values are the defaults from the `.policy` file. Custom rules may override these at runtime.

### Check whether a specific subject can do something

```bash
pkcheck --action-id org.freedesktop.login1.suspend \
        --process $$ \
        --allow-user-interaction
# exits 0 if allowed, nonzero otherwise
```

This is for debugging: "could I do this right now?" Also used inside services as the actual polkit query, though services usually go through D-Bus calls to `org.freedesktop.PolicyKit1` directly.

### Watch polkit decisions live

```bash
journalctl -u polkit -f
# When a policy rule is loaded with errors, this shows it.
# When rules call polkit.log(), messages appear here.
# When auth happens, polkit logs the decision.
```

### See who's registered as agents

```bash
busctl --system call org.freedesktop.PolicyKit1 \
    /org/freedesktop/PolicyKit1/Authority \
    org.freedesktop.PolicyKit1.Authority EnumerateTemporaryAuthorizations s "$XDG_SESSION_ID"
# lists temporary auths from _keep variants
```

## The D-Bus API

polkit runs as `org.freedesktop.PolicyKit1` on the system bus, with one object at `/org/freedesktop/PolicyKit1/Authority`. Key methods:

- **`CheckAuthorization(subject, action_id, details, flags, cancellation_id)`** — the core query. Returns authorization result.
- **`RegisterAuthenticationAgent(subject, locale, path)`** — agents call this at startup.
- **`AuthenticationAgentResponse2(serial, cookie, identity)`** — agent reports back.
- **`EnumerateTemporaryAuthorizations(subject)`** — see what's cached from `_keep` results.
- **`RevokeTemporaryAuthorizations(subject)`** — flush the cache.

```bash
busctl introspect org.freedesktop.PolicyKit1 /org/freedesktop/PolicyKit1/Authority
busctl monitor org.freedesktop.PolicyKit1
```

Watching the bus as you click around in GNOME Settings is educational — you see every `CheckAuthorization` fire.

## Tutorial: debug why an action is being denied

You expected something to work, and it didn't. Here's the methodical trace.

### 1. What action is involved?

The service's documentation is the first stop — `man NetworkManager`, `man systemd-logind`, etc. Or:

```bash
pkaction | grep -i <service>
```

Or drive the service with logging, for NetworkManager:

```bash
journalctl -u NetworkManager -f
# in another shell, do the thing that fails
```

You'll see messages like "Not authorized: org.freedesktop.NetworkManager.settings.modify.system".

### 2. Look up the action's defaults

```bash
pkaction --action-id <full.action.id> --verbose
```

Cross-reference the three implicit values against your subject's context:

```bash
loginctl show-session $XDG_SESSION_ID
# Check: Active, Remote, State
```

If you're `Remote=yes`, you're hitting `allow_any`, not `allow_active`.

### 3. Check for overriding rules

```bash
ls /etc/polkit-1/rules.d/
ls /usr/share/polkit-1/rules.d/
```

Read through them. Default distro rules are in `/usr/share/polkit-1/rules.d/`; admin rules in `/etc/`. If there's a rule covering your action, it takes precedence over the defaults.

### 4. Test the actual check

```bash
pkcheck --action-id <action.id> --process $$ --allow-user-interaction
echo $?                           # 0 = allowed, 1 = not allowed, 2 = challenge issued
```

### 5. If needed, write a rule to fix

```javascript
// /etc/polkit-1/rules.d/90-mycustom.rules
polkit.addRule(function(action, subject) {
    // your logic
});
```

Save; polkit auto-reloads; try again.

### 6. Check the journal

```bash
journalctl -u polkit --since "5 min ago"
```

Rule parse errors, loaded-file messages, `polkit.log()` output all go here.

## Things that trip people up

**"I ran `sudo` and it still prompts twice / fails via GUI" / "polkit prompt doesn't appear."** You may have no agent in your session. SSH sessions have no polkit agent; GUI apps run from SSH can't show prompts. Either register `pkttyagent` in your terminal, or use `sudo` for those workflows.

**"I added a rule and nothing changed."** Check `journalctl -u polkit` for a parse error. If the JS throws, polkit ignores that file. Also check file ordering — a lower-numbered file may match first and win. And remember: you need to call `polkit.addRule(...)`, returning a value from a bare expression does nothing.

**"Rule returns work but keep prompting."** You returned `polkit.Result.AUTH_ADMIN` when you meant `YES`, or vice versa. Easy to confuse.

**"My rule matches but nothing is returned."** Rules that *don't* return a value fall through to the next rule, ending at the action's defaults. If no rule returns, defaults apply. This is usually what you want, but means "I wrote a matching rule, the defaults still run" is the expected behavior when your rule doesn't `return`.

**`auth_admin` prompts for root password vs. wheel member password.** This is distro configuration in `/etc/polkit-1/rules.d/` — on Fedora / most modern distros, admins are wheel members; on some older ones, it's literal root. `pkaction --verbose` shows which identity groups count as admin on your system.

**`allow_active` requires an *active* session, not just "local".** Screen-locked sessions are often still active; switched-to-other-VT sessions are inactive. Your session is active right now if you can see this terminal on your own display.

**Headless servers: polkit mostly doesn't matter.** Without an agent, any `auth_*` result becomes effectively denied. If you're managing a headless box, use `sudo`/root for admin and don't rely on desktop-style polkit flows. Pre-authorizing specific actions via rules is the fix when you do need unprivileged services to work headless.

**Docker/Podman containers.** polkit inside a container is almost never useful — the container doesn't have a session, a seat, an agent. Containerized services that need privileged operations usually have to run as root inside the container, or use capabilities, or mount the host's D-Bus and hope.

**`pkexec` with GUI programs under Wayland.** `pkexec` scrubs the environment, including `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR`. GUI programs run through `pkexec` may fail to find the display. For GUI elevation, the recommended pattern is: the unprivileged GUI app calls a privileged helper over D-Bus (which polkit checks), rather than `pkexec`-ing the GUI itself.

**Changes to `.policy` files require nothing special** — polkit re-reads them on next use (or you can `systemctl reload polkit`). Changes to `rules.d/` are watched automatically.

**Two rules that both match.** First match that returns a value wins. If you want to override a distro rule, use a lower-numbered filename in `/etc/polkit-1/rules.d/` than the distro rule in `/usr/share/polkit-1/rules.d/` (lower numbers run first).

**"Temporary auth keep" confuses people.** After an `auth_*_keep`, polkit caches the positive result for ~5 minutes against that (subject, action). Next call within the window returns `yes` without prompting. Revoke with:

```bash
# Revoke this session's temporary auths
pkcheck --revoke-temp   # if your polkit is recent
# or via busctl
busctl --system call org.freedesktop.PolicyKit1 \
    /org/freedesktop/PolicyKit1/Authority \
    org.freedesktop.PolicyKit1.Authority RevokeTemporaryAuthorizations s "$XDG_SESSION_ID"
```

## How polkit fits with everything else

- **vs PAM.** PAM handles authentication (who are you). polkit handles authorization (what are you allowed to do). polkit uses PAM internally when it prompts for a password, but they're distinct layers.
- **vs sudo.** Both solve "allow unprivileged user to do privileged thing." sudo is rule-based (`/etc/sudoers`) and interactive/scripted shell-oriented. polkit is context-aware (session state) and D-Bus-oriented. Both are needed on most systems; neither fully replaces the other.
- **vs setuid.** polkit + `pkexec` is the modern replacement for many setuid binaries. Smaller attack surface, per-program policy, auditable.
- **vs logind.** polkit asks logind "is this subject's session active, local, remote?" to classify. Rules reason about those attributes. logind is polkit's primary source of session context.
- **vs systemd.** systemd's D-Bus API (`org.freedesktop.systemd1`) uses polkit to gate unit management. That's why `systemctl start foo` from your desktop prompts, but `sudo systemctl start foo` doesn't — the first goes through polkit, the second bypasses it by already being root.
- **vs NetworkManager, logind, udisks, etc.** Every D-Bus service with privileged methods is a polkit client. Their `.policy` files declare the actions; polkit arbitrates.
- **vs AppArmor / SELinux.** Different layer. MAC systems enforce what a *process* can do based on its label (regardless of who asked). polkit decides whether a *request* is allowed based on who's asking. They're complementary; you can use both.
- **vs Flatpak / xdg-desktop-portal.** Portals mediate sandboxed apps' access to host resources. Portals often call polkit internally for the "is this user allowed?" part, but their main job is sandbox-to-host mediation, a different concern.

## A brief note on the naming

You'll see both "PolicyKit" and "polkit" in docs and code. "PolicyKit" was the original project name; "polkit" became the preferred spelling around 2012 (when the project moved to freedesktop.org and the main binary was renamed). D-Bus names and some older docs retain "PolicyKit1" in the interface names for backward compatibility. Same thing either way.

## Quick reference card

```bash
# Inspect actions
pkaction                                       # list all
pkaction --action-id <id> --verbose            # details of one

# Check an authorization
pkcheck --action-id <id> --process $$ --allow-user-interaction
echo $?                                        # 0=yes, 1=no, 2=challenge

# Run a program as root via polkit
pkexec <program> [args...]

# Start a TTY auth agent (for SSH / headless interactive use)
pkttyagent --process $$ &

# Rule files
/etc/polkit-1/rules.d/          # admin rules (yours go here)
/usr/share/polkit-1/rules.d/    # distro rules
/usr/share/polkit-1/actions/    # action declarations

# Format: NN-description.rules (lower N = earlier, wins)
polkit.addRule(function(action, subject) {
    if (action.id == "..." && subject.isInGroup("...")) {
        return polkit.Result.YES;
    }
});

# Result values
polkit.Result.YES / NO
polkit.Result.AUTH_SELF / AUTH_SELF_KEEP
polkit.Result.AUTH_ADMIN / AUTH_ADMIN_KEEP
polkit.Result.NOT_HANDLED                      # fall through to next rule

# Debug
journalctl -u polkit -f
busctl monitor org.freedesktop.PolicyKit1

# Subject attributes in rules
subject.user                                    # username
subject.isInGroup("wheel")
subject.groups                                  # array
subject.seat, subject.session
subject.local, subject.active                   # booleans
subject.pid

# Common actions to know
org.freedesktop.login1.suspend
org.freedesktop.login1.hibernate
org.freedesktop.login1.power-off
org.freedesktop.login1.reboot
org.freedesktop.NetworkManager.network-control
org.freedesktop.NetworkManager.settings.modify.system
org.freedesktop.systemd1.manage-units
org.freedesktop.UDisks2.filesystem-mount
org.freedesktop.UDisks2.filesystem-mount-system
org.freedesktop.packagekit.package-install
org.freedesktop.hostname1.set-hostname
org.freedesktop.timedate1.set-timezone
```

## Where to learn more

- **`man polkit`, `man polkit.8`, `man pkaction`, `man pkcheck`, `man pkexec`, `man pkttyagent`** — the reference. Short and accurate. `polkit(8)` is the best single overview.
- **`man polkit`** section on the JavaScript API — documents every available method on `action`, `subject`, and `polkit.*`.
- **`/usr/share/polkit-1/actions/*.policy`** — read a few to see real action declarations.
- **`/usr/share/polkit-1/rules.d/*.rules`** — read the distro-shipped rules, especially `50-default.rules`, which typically defines what "admin" means on your system.
- **freedesktop.org polkit project page** — historical and current.

The mental model to carry: **polkit is a system daemon that answers authorization questions ("can this subject perform this action?") on behalf of any service that asks, consulting declarative action defaults and administrator-provided JavaScript rules, with context including the caller's identity, group membership, and logind session state (local/remote, active/inactive), and — if authentication is required — prompting via a per-session agent.** It's the piece that lets unprivileged desktop users do legitimately-privileged things without handing them root, with policy centralized instead of scattered across every service's ad-hoc mechanism. Once the triangle (action → subject → result) and the local/active session concept are in your head, the entire "why did this prompt appear / why didn't it appear" mystery becomes a tractable thing you can trace with `pkaction` and `pkcheck`.