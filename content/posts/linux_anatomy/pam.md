---
date: '2026-04-19T22:11:48-07:00'
draft: true
title: 'Chapter 9: PAM'
weight: 90
---
## What PAM is and why it exists

In early Unix, authentication was hardcoded. The `login` program read `/etc/passwd`, compared a crypted password, let you in or didn't. `su` did the same check. `ftpd` had its own copy of the check. Every program that needed to authenticate users re-implemented the same logic against the same files.

This broke as soon as sites wanted anything beyond `/etc/passwd`:

- **NIS** (Yellow Pages) — network-shared user databases.
- **Kerberos** — ticket-based auth.
- **LDAP** — directory services.
- **Shadow passwords** — moving the hash out of the world-readable `/etc/passwd`.
- **One-time passwords** — S/Key, SecurID tokens.
- **Smartcards, biometrics, FIDO2** — hardware auth.

For each new mechanism, you had to patch *every* authenticating program. `login`, `su`, `ftpd`, `sshd`, `cron`, `passwd`, `screen`, the display manager — dozens of programs, each with their own recompile-the-world to support a new auth backend. Worse, sites wanted to *combine* mechanisms: "require password AND smartcard", "try Kerberos then fall back to local password", "rate-limit failed attempts", "log to audit".

**PAM** (Pluggable Authentication Modules) is the answer, dating from 1995 (Sun originally, adopted by Linux-PAM in 1996). The core idea: **applications stop implementing authentication themselves**. Instead, they call the PAM library, which reads a config file and runs a chain of plugin modules that do the actual work. Add a new auth mechanism? Write a PAM module. Every PAM-aware program gets it for free.

On modern Linux, essentially every program that authenticates users — `login`, `sshd`, `sudo`, `su`, `passwd`, `systemd-logind` (via `pam_systemd`), display managers, `cron`, `cockpit` — is a PAM client. The list of installed PAM modules on your system shapes how all of them behave.

## The central abstraction: four stacks and a chain

PAM groups authentication-related operations into **four independent management groups** (sometimes called "facilities" or "types"):

| Group | Question it answers |
|---|---|
| **`auth`** | "Is this person who they claim to be?" (Verify identity.) |
| **`account`** | "Is this account allowed to log in right now?" (Policy: expired? locked? time restrictions?) |
| **`password`** | "Change this account's credentials." (Invoked by `passwd`, etc.) |
| **`session`** | "Set up / tear down the login session." (Mounts, limits, logging, environment.) |

These are *separate concerns*. A Kerberos PAM module handles `auth` (check ticket) and maybe `session` (refresh ticket), but doesn't care about `account` (LDAP might). A pam_time module handles `account` (deny login outside work hours) but says nothing about `auth`. Each management group is configured independently.

For each group, an application can run *a stack of modules in order*. This is the chain: each module returns a success/failure, and control flags determine whether the chain continues, aborts, overrides.

When `sshd` authenticates you, it makes four PAM calls in sequence (roughly):

```
pam_authenticate(handle)       → runs the 'auth' stack
pam_acct_mgmt(handle)          → runs the 'account' stack
pam_chauthtok(handle)          → (only if password expired, runs 'password' stack)
pam_open_session(handle)       → runs the 'session' stack at session start
pam_close_session(handle)      → runs the 'session' stack at session end
```

Same four calls for every PAM app. What the calls *do* is whatever the config file for that application says.

## Config file anatomy

Per-application config lives in `/etc/pam.d/<application-name>`. On modern distros, `/etc/pam.conf` (the monolithic old form) isn't used; it's per-file under `/etc/pam.d/`.

```bash
ls /etc/pam.d/
# chpasswd          gnome-screensaver  polkit-1   sshd       systemd-user
# chsh              gnome-shell        postlogin  su         ...
# fingerprint-auth  login              remote     sudo       ...
# gdm-password      passwd             runuser    system-auth
```

Each file is a list of lines, one rule per line. The format:

```
<group>  <control>  <module>  [module-args...]
```

Example, simplified `/etc/pam.d/sshd`:

```
auth       substack     password-auth
auth       include      postlogin
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
```

Four groups, each with its own stack of rules.

- `<group>` is `auth`, `account`, `password`, or `session`.
- `<control>` governs what happens on module success/failure (next section).
- `<module>` is a `.so` file, looked up in `/lib64/security/` (or `/lib/x86_64-linux-gnu/security/`).
- `[args...]` are passed to the module.

The `include` and `substack` directives let you pull in other config files as chain fragments — that's how you get shared auth policy across many apps. `/etc/pam.d/password-auth` is typically *the* file that defines the real auth logic, and every app that wants standard behavior just `include`s it.

On Fedora, the main shared files are:

- **`/etc/pam.d/system-auth`** — used by console/local auth (login, GDM, su).
- **`/etc/pam.d/password-auth`** — used by network/remote auth (sshd, cockpit).
- **`/etc/pam.d/postlogin`** — stuff done after successful auth regardless of source (motd, last-login display).
- **`/etc/pam.d/fingerprint-auth`**, **`/etc/pam.d/smartcard-auth`** — specialized pulls.

Debian/Ubuntu use `common-auth`, `common-account`, `common-password`, `common-session` instead.

**These shared files are managed by `authselect`** on Fedora (or `authconfig` on older systems) — a tool that renders them from templates based on which auth scheme you've picked (local, SSSD, Winbind, etc.). Don't hand-edit `system-auth` / `password-auth`; your changes get blown away on next `authselect apply-changes`. Either write a custom authselect profile or modify the per-app files to bypass the shared include.

```bash
authselect current                      # which profile / features are active
authselect list                         # available profiles
authselect list-features sssd           # features available in a profile
```

## Control flags: how the chain flows

The control flag is the trickiest part of PAM syntax. It governs how a module's result affects the overall chain.

### The simple form (sufficient for most configs)

Four classical keywords:

- **`required`** — module must succeed. If it fails, the chain still runs to the end (don't want to leak information about *which* module failed), but the final answer is fail.
- **`requisite`** — module must succeed. If it fails, the chain aborts immediately, and the final answer is fail.
- **`sufficient`** — if the module succeeds and no prior `required` failed, the chain short-circuits with success. If it fails, no effect — keep trying.
- **`optional`** — module's result doesn't affect the final answer unless it's the *only* module in the stack.

A typical `auth` stack:

```
auth  required    pam_env.so             # set environment vars
auth  required    pam_faillock.so preauth # check lockout state
auth  sufficient  pam_unix.so try_first_pass  # try local password
auth  sufficient  pam_sss.so              # try SSSD (LDAP/Kerberos)
auth  required    pam_deny.so             # explicit deny if we got here
```

Logic: try local password; if it works, succeed (sufficient). Otherwise try SSSD; if it works, succeed. If nothing succeeded, pam_deny forces failure. pam_faillock is `required` so it always runs (to record the attempt).

**`required` vs `requisite` distinction:** `required` failing still runs remaining modules; `requisite` aborts. Why keep running after failure? To hide *which* module failed — an attacker can't tell from timing whether the username was wrong or the password was wrong, because the rest of the stack always runs.

### The extended form (`[...]=...`)

Modern PAM supports much finer control:

```
auth [success=1 default=ignore] pam_unix.so nullok
auth requisite                  pam_deny.so
auth required                   pam_permit.so
```

Syntax: `[VALUE1=ACTION1 VALUE2=ACTION2 ...]`. Module returns a `VALUE` (success, user_unknown, auth_err, etc.), and the corresponding `ACTION` fires:

- **`ok`** — record the result, keep going.
- **`done`** — record the result, short-circuit to end with success.
- **`bad`** — record as failure, keep going.
- **`die`** — record as failure, short-circuit to end with failure.
- **`ignore`** — don't count this module's result.
- **`reset`** — reset to a neutral state.
- **`N` (a number)** — skip the next N modules.

So `[success=1 default=ignore]` means "on success, skip the next 1 rule; on anything else, ignore."

The example above translates to: if `pam_unix` succeeds, jump past `pam_deny` to `pam_permit` (which always succeeds); if `pam_unix` fails, fall through to `pam_deny` (which always fails).

You'll see these in real configs, especially Debian/Ubuntu `common-auth`. They're more expressive than `required`/`sufficient` but harder to read. The classic keywords are equivalent to specific extended forms.

## Common modules

A tour of modules you'll encounter.

### Authentication (`auth`)

| Module | Purpose |
|---|---|
| `pam_unix.so` | Local password check against `/etc/shadow`. The workhorse. |
| `pam_sss.so` | SSSD — LDAP, Kerberos, FreeIPA, AD. |
| `pam_krb5.so` | Direct Kerberos. |
| `pam_ldap.so` | Direct LDAP (older than SSSD). |
| `pam_winbind.so` | Samba Winbind for AD. |
| `pam_google_authenticator.so` | TOTP. |
| `pam_oath.so` | OATH HOTP/TOTP. |
| `pam_u2f.so` | FIDO U2F / FIDO2 / YubiKey. |
| `pam_fprintd.so` | Fingerprint reader via `fprintd`. |
| `pam_smartcard.so` / `pam_pkcs11.so` | Smartcards, PIV. |
| `pam_faillock.so` | Track and lock out failed attempts. |
| `pam_wheel.so` | Restrict access (commonly for `su`) to members of a group. |
| `pam_rootok.so` | Auto-succeed if calling UID is 0 (why root doesn't need password for `su`). |
| `pam_nologin.so` | Deny logins if `/etc/nologin` exists. |
| `pam_deny.so` | Always fail. |
| `pam_permit.so` | Always succeed. |
| `pam_warn.so` | Log a warning; always succeeds. |

### Account (`account`)

| Module | Purpose |
|---|---|
| `pam_unix.so` | Check password expiration, account disabled state. |
| `pam_access.so` | Rule-based allow/deny by user/host (`/etc/security/access.conf`). |
| `pam_time.so` | Deny based on time-of-day policy (`/etc/security/time.conf`). |
| `pam_tally2.so` / `pam_faillock.so` | Enforce lockout from prior failed attempts. |
| `pam_sss.so` | Defer to SSSD's account checks. |

### Password (`password`)

| Module | Purpose |
|---|---|
| `pam_unix.so` | Set the local password. |
| `pam_pwquality.so` / `pam_cracklib.so` | Enforce password strength policy. |
| `pam_pwhistory.so` | Prevent reusing recent passwords. |
| `pam_sss.so` | Change password in directory. |

### Session (`session`)

| Module | Purpose |
|---|---|
| `pam_systemd.so` | Register session with logind (covered in logind guide). |
| `pam_limits.so` | Apply `ulimit`-style limits from `/etc/security/limits.conf`. |
| `pam_env.so` | Set environment variables from `/etc/security/pam_env.conf`. |
| `pam_lastlog.so` | Update `/var/log/lastlog`, print "last login" message. |
| `pam_motd.so` | Show the message of the day. |
| `pam_mail.so` | Tell you if you have new mail. |
| `pam_mkhomedir.so` | Auto-create home directory on first login. |
| `pam_namespace.so` | Set up polyinstantiated directories (per-user `/tmp`). |
| `pam_selinux.so` | Transition to user's SELinux context. |
| `pam_keyinit.so` | Initialize kernel keyring session. |
| `pam_loginuid.so` | Set auditable login UID (`/proc/self/loginuid`). |

`pam_systemd.so` is worth highlighting. It's the glue between PAM and logind: when the session stack runs it, it calls `CreateSession` on logind, which sets up the cgroup scope, allocates `/run/user/UID`, starts `user@UID.service` if needed, applies device ACLs based on seat/class. Without `pam_systemd.so` in your session stack, none of the session-scope machinery happens. Every PAM-aware program that creates user sessions includes it.

### Module args worth knowing

Common args that change module behavior:

- `pam_unix`:
  - `try_first_pass` — use the password already entered by a previous module.
  - `use_first_pass` — same, but fail if no prior password.
  - `nullok` — allow empty passwords.
  - `sha512` — hash algorithm (usually the default now).
  - `remember=N` — keep last N passwords.
- `pam_faillock`:
  - `preauth` — in the `auth` stack *before* the password check; deny if already locked.
  - `authfail` — in the `auth` stack *after*; count a failure.
  - `authsucc` — after success; clear counter.
  - `deny=N`, `unlock_time=N`, `even_deny_root`.
- `pam_limits`:
  - `noaudit`, `debug`, `set_all`.
- `pam_systemd`:
  - Usually args-less; it detects the session context automatically.

## The four stacks in action: what each phase does

Let's trace a typical login end to end.

### 1. `auth` — prove identity

```
auth  required    pam_env.so
auth  required    pam_faillock.so preauth deny=3 unlock_time=600
auth  sufficient  pam_unix.so try_first_pass nullok
auth  [default=die] pam_faillock.so authfail
auth  sufficient  pam_sss.so use_first_pass
auth  required    pam_deny.so
```

Logic: set env. Check lockout state (deny if already locked). Try local password; if it works, short-circuit to success. If local failed, count the failure for faillock; then try SSSD. If neither worked, pam_deny forces failure.

`pam_unix` on success also clears the faillock counter (via the `authsucc` module in some configs).

### 2. `account` — is this account usable?

```
account  required  pam_nologin.so
account  required  pam_faillock.so
account  required  pam_unix.so
account  required  pam_access.so
account  sufficient pam_localuser.so
account  required  pam_sss.so
```

Checks: is `/etc/nologin` present (deny all non-root). Is the account faillocked. Does the local account exist and is it not expired. Does access.conf allow this user from this origin. If local user, pass. Otherwise check SSSD's account policy (LDAP posixAccount attributes, Kerberos policy).

### 3. `password` — credential change (only invoked for passwd or expiry)

```
password  requisite  pam_pwquality.so retry=3
password  sufficient pam_unix.so use_authtok sha512 shadow remember=5
password  sufficient pam_sss.so use_authtok
password  required   pam_deny.so
```

Quality check on the new password. Set it in local shadow; if this is a directory user, set it in the directory.

### 4. `session` — build and tear down

```
session  required  pam_selinux.so close
session  required  pam_loginuid.so
session  required  pam_selinux.so open
session  optional  pam_keyinit.so force revoke
session  required  pam_limits.so
session  optional  pam_systemd.so
session  optional  pam_lastlog.so silent
session  optional  pam_mail.so standard
session  required  pam_unix.so
```

This runs at session start (`pam_open_session`) and in reverse at end (`pam_close_session`). pam_systemd registers with logind. pam_limits reads limits.conf and applies rlimits. pam_lastlog updates and displays. pam_selinux sets up the process's SELinux context.

## Hands-on: inspect and test

### See what an app uses

```bash
cat /etc/pam.d/sshd
cat /etc/pam.d/sudo
cat /etc/pam.d/system-auth
```

### Follow includes

```bash
# Print every rule involved in sshd auth:
grep -r "^auth" /etc/pam.d/ | grep -E "(sshd|password-auth|postlogin)"
```

### Watch PAM live

PAM operations log to the journal via syslog. For verbose output, enable debug:

```bash
# Temporarily enable PAM debug:
sudo sh -c 'echo "*.* /var/log/pam_debug.log" > /etc/rsyslog.d/90-pamdebug.conf'
sudo systemctl restart rsyslog

# or just watch authpriv:
sudo journalctl -f _TRANSPORT=syslog SYSLOG_FACILITY=10
# auth facility = 4, authpriv = 10
```

Many modules support a `debug` argument — add it to see diagnostic output in the journal.

### List installed modules

```bash
ls /lib64/security/                     # Fedora / RHEL
ls /lib/x86_64-linux-gnu/security/      # Debian / Ubuntu
```

### Read a module's man page

Almost every module has one:

```bash
man pam_unix
man pam_systemd
man pam_faillock
man pam_limits
```

These are the best reference. Short, complete, list every argument.

### `pamtester` — interactively exercise PAM

`pamtester` (package `pamtester` on Fedora/Debian) lets you run a PAM stack without having the real application. Invaluable for testing config changes.

```bash
# Test the 'login' stack for authentication of user zl:
pamtester login zl authenticate

# Test multiple operations:
pamtester login zl authenticate acct_mgmt open_session close_session

# Verbose:
pamtester -v sshd $USER authenticate
```

Change a config file, re-run pamtester, see if it behaves as expected. Much faster than constantly SSHing or logging in.

### Check account lockout state

```bash
faillock --user zl                      # see recent failures and lock state
faillock --user zl --reset              # clear
```

## Tutorial: add FIDO2 / YubiKey to sudo

A concrete customization. Goal: `sudo` should accept either your password *or* a YubiKey touch.

### 1. Install the module

```bash
sudo dnf install pam-u2f
```

### 2. Register your key

```bash
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys   # touch the key when prompted
# Add more keys by appending:
pamu2fcfg >> ~/.config/Yubico/u2f_keys
```

### 3. Edit `/etc/pam.d/sudo`

Default:

```
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    include      system-auth
```

Change to:

```
auth       sufficient   pam_u2f.so cue
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    include      system-auth
```

Logic: try U2F first; on success, bypass the rest of auth (sufficient). On failure (or no key present), fall through to system-auth (password).

`cue` makes the module print "Please touch the device." when waiting.

### 4. Test

In a *new* terminal (don't lose your existing sudo-authenticated terminal in case of misconfig):

```bash
sudo -K    # reset sudo's cached auth
sudo echo hi
# Should prompt to touch key. Touch it. "hi" should print.
```

If it's broken: check `journalctl -f` in your safe terminal. Revert by editing back. Lesson: **always keep a root-authenticated terminal open when editing PAM configs.** A broken PAM config can lock you out of `sudo`, `ssh`, and sometimes `login` entirely.

### 5. Require both password *and* key (instead of either)

Change `sufficient` to `required`:

```
auth       required     pam_u2f.so cue
auth       include      system-auth
```

Now both must succeed. Order matters: `pam_u2f` runs first, then system-auth runs regardless (because `required` doesn't short-circuit). User gets prompted to touch key, then for password.

## Things that trip people up

**Editing `system-auth` / `password-auth` on Fedora.** Don't. They're generated by `authselect`. Use `authselect apply-changes` after editing the per-service file (e.g., `sshd`, `sudo`) to bypass the include, or create a custom authselect profile. If you've already edited and authselect overwrote your changes, pull from git history or rebuild.

**Locking yourself out.** Most common ways: misspelling a module name (PAM errors out and denies), writing `required` where you meant `sufficient`, breaking the `account` stack so even correct passwords are denied, introducing a syntax error. **Always keep a privileged shell open during edits.** For SSH: use a second session. For console: switch to another VT with root logged in.

**"password required pam_unix.so" doesn't mean "require password" for login.** The `password` group is for *changing* credentials, not for *verifying* them. Verification is `auth`. New users confuse this weekly.

**Missing `pam_systemd` in session stack.** No session registered with logind → no `/run/user/UID` → no user services → user's D-Bus session bus doesn't start correctly → desktop breaks in subtle ways. Every custom session file should include `pam_systemd` or an include that does.

**`pam_env` reads only specific files.** `/etc/environment` and `/etc/security/pam_env.conf` (or `~/.pam_environment`, now often disabled for security). Not `/etc/profile`, not `.bashrc`. Setting env in bash rc files doesn't affect PAM.

**Module order matters more than you think.** `pam_faillock preauth` must come *before* the password-checking module; `pam_faillock authfail` must come *after*. Wrong order silently breaks lockout enforcement. Some distros combine these into a single module call with args; check your distro's stock config before adding.

**The `try_first_pass` / `use_first_pass` dance.** When stacking multiple `auth` modules that all want a password, you don't want the user prompted multiple times. First module prompts and stores the password in the PAM context; subsequent modules use `try_first_pass` (try the stored one, prompt if unset) or `use_first_pass` (use the stored one, fail if unset). If you see users being prompted for their password three times in a row, a module earlier in the stack isn't passing the password along properly.

**`sufficient` + shared stack = surprise.** `sufficient` short-circuits *within* the current substack. A `sufficient` success in an `include`d file doesn't short-circuit back to the parent unless you use `substack` instead of `include`. `substack` is newer (2006+) and subtly different; if you're seeing unexpected fall-through, the difference is the culprit.

**Headless servers with GUI-oriented PAM configs.** Some modules assume a TTY (fingerprint, GUI prompts). On a serverless SSH login, these either fail gracefully or hang. Test non-interactive paths.

**Per-app vs. system-wide changes.** If you only want to change behavior for one program (e.g., relax password requirements for a specific service user), edit that app's per-service file, not system-auth. Less blast radius.

**PAM and containers.** Containerized services that run `su` or auth generally have minimal PAM stacks — `/etc/pam.d/` in the container is what matters, not the host's. Containers that don't install `libpam-modules` have broken auth.

**Kerberos ticket refresh** — `pam_krb5` needs to run in both `auth` and `session` stacks. Auth gets the ticket; session refreshes it. Missing the session step means tickets expire during the user's session.

**Password policy not being enforced.** `pam_pwquality` / `pam_cracklib` is in the `password` stack. If users aren't running `passwd` (because passwords come from LDAP/AD), the policy in your local PAM config doesn't matter; directory-side policy does.

## Security considerations

A broken PAM config is a security hole. A few principles:

**Fail closed.** End every stack (`auth`, `account`) with `pam_deny.so` or `pam_unix.so required` as a backstop. If you removed a terminal `pam_deny`, make sure the last module in the stack is `required`, so an empty stack doesn't accidentally succeed.

**Don't use `nullok` unless you mean it.** `pam_unix.so nullok` allows empty passwords. This is rarely what you want on real systems. Audit your configs for it.

**Don't skip `pam_faillock` (or equivalent).** Without it, brute-force attempts against auth have no rate limit.

**Modules run as the user who invoked the app.** Well, mostly. `login` and `sshd` start as root and run PAM as root; they drop privilege after. Apps that are *not* setuid (e.g., `passwd` in some configs) run as the user, so modules requiring root access (like pam_unix reading `/etc/shadow`) use helper binaries.

**PAM config files are root-owned, 0644.** Anyone can read them, including service definitions. Passwords, keys, etc. should *never* appear in PAM config; use separate files in `/etc/security/` with restrictive permissions for module data.

**Audit logs.** `pam_loginuid.so` sets the immutable "loginuid" that auditd uses to track actions back to the originating login session even across `su`, `sudo`. It's in session stacks for a reason; don't remove it.

## How PAM fits with everything else

- **vs. NSS (Name Service Switch).** Distinct layers, often confused. **NSS** answers "who is user zl? what's their UID, home dir, shell?" via `/etc/nsswitch.conf` and modules like `nss_files`, `nss_sss`, `nss_ldap`. **PAM** answers "can user zl log in right now?". Both modules often share backends (SSSD does both, via pam_sss + nss_sss), but they're independent. A user must exist in NSS *and* pass PAM's auth stack to log in.

- **vs. logind.** PAM's session stack calls logind via pam_systemd. Logind then owns the session lifecycle. PAM does the auth; logind does session tracking.

- **vs. polkit.** PAM is for login-time authentication. polkit is for runtime authorization of privileged operations by already-logged-in users. polkit actually calls PAM when it needs to verify a password in its auth dialog — so they stack: polkit asks "can this user do X?"; if answer requires credentials, polkit runs a PAM stack to verify them.

- **vs. sudo.** `sudo` is a PAM client. `/etc/pam.d/sudo` defines what auth it requires. Independently, `/etc/sudoers` defines *which commands* a user may run. Authentication via PAM; authorization via sudoers. Both must pass.

- **vs. SSSD.** SSSD is a daemon that centralizes LDAP/Kerberos/AD integration. PAM talks to SSSD via `pam_sss.so`; NSS talks to SSSD via `nss_sss.so`. SSSD caches credentials, handles offline login, manages Kerberos tickets. It's the modern replacement for direct `pam_ldap` / `pam_krb5` stacks.

- **vs. SSH keys.** Critically: **SSH key authentication bypasses the PAM `auth` stack.** If `PubkeyAuthentication yes` in `sshd_config` and your key matches, sshd does not run the auth stack. It *does* still run the `account` and `session` stacks (so pam_nologin, pam_access, pam_systemd, pam_limits all apply). This means: "restrict sshd auth via PAM" doesn't restrict key-based logins. Use `AllowUsers`, `AuthorizedKeysCommand`, or `UsePAM yes` with extra `account` checks for that.

- **vs. the kernel.** PAM is entirely userspace. The kernel has its own unrelated auth concepts (capabilities, credentials, keyrings, LSMs), which are the layer below PAM's UID/GID model.

## Quick reference card

```bash
# Config files
/etc/pam.d/<app>                    # per-application stack
/etc/pam.d/system-auth              # shared, local auth (Fedora)
/etc/pam.d/password-auth            # shared, network auth (Fedora)
/etc/pam.d/common-*                 # Debian/Ubuntu equivalents
/etc/security/*.conf                # module data files (limits, time, access, env)

# Management groups
auth      # verify identity
account   # is account usable
password  # change credentials
session   # set up / tear down session

# Control flags (classic)
required    # must succeed; keep running on fail
requisite   # must succeed; abort on fail
sufficient  # success short-circuits to end
optional    # result ignored unless only module

# Authselect (Fedora)
authselect current
authselect list
sudo authselect select sssd with-mkhomedir
sudo authselect apply-changes

# Inspect
ls /lib64/security/                 # installed modules
man pam_<module>                    # per-module docs
faillock --user zl                  # lockout state

# Test
sudo dnf install pamtester
pamtester -v <service> <user> authenticate

# Debug logs
sudo journalctl -f SYSLOG_FACILITY=10       # authpriv
# Many modules accept 'debug' argument for extra logging

# Common modules
pam_unix       pam_sss        pam_systemd     pam_faillock
pam_limits     pam_env        pam_nologin     pam_access
pam_u2f        pam_pwquality  pam_time        pam_deny
```

## Where to learn more

- **`man 8 pam`** — the overview.
- **`man 5 pam.conf`** — config syntax reference. The authoritative source for control flag semantics, both classic and extended forms.
- **`man 7 PAM`** — introduction.
- **Per-module man pages.** `man pam_unix`, `man pam_faillock`, etc. These are where the real documentation is.
- **The Linux-PAM Application Developer's Guide** — online at `http://www.linux-pam.org/Linux-PAM-html/`. Verbose, but covers every return code and module call.
- **`/etc/pam.d/` on your system** — reading real configs teaches a lot.
- **Red Hat / Fedora's `authselect` docs** — for anything touching the Fedora-managed shared stacks.

The mental model to carry: **PAM is a system-wide framework where authenticating programs delegate four distinct concerns — identity verification, account usability, credential change, and session setup/teardown — to a chain of pluggable modules, configured per-application in `/etc/pam.d/`, so that every auth-aware program gains new auth capabilities (Kerberos, FIDO2, lockout tracking, audit logging) just by adjusting the config.** Once the four-groups idea and the control-flag semantics click, PAM configs stop looking like arcana and become what they are: small, declarative chains that compose well. And once you know which modules exist, extending your system's auth — fingerprint, YubiKey, time-of-day restrictions, SSO — is a matter of dropping a line into the right `/etc/pam.d/` file and testing with `pamtester`.