# Metasploitable 2: Privilege Escalation via Unrestricted Sudo Access

**Target:** Metasploitable 2 (`10.0.130.10`)
**Environment:** Isolated home lab (Proxmox, pfSense-segmented VLAN)
**Date:** November 2, 2026
**Tester:** Russ

## Summary

After authenticating over SSH with credentials obtained from a separate finding (see [bind shell & weak credentials writeup](bindshell-weak-creds.md)), the `msfadmin` account was found to hold unrestricted `sudo` privileges. This allowed trivial, immediate escalation from a standard user to full root access with a single command.

## Scope and environment

All testing was performed against a vulnerable-by-design VM (Metasploitable 2) running in an isolated lab VLAN with no route to production systems, management infrastructure, or the internet. No real-world systems were tested.

## Methodology

1. Authenticate as a low-privilege user via SSH
2. Enumerate current user's sudo permissions
3. Escalate to root using the permissions identified
4. Confirm access level and document the misconfiguration

## Finding: Unrestricted Sudo Access (`msfadmin`)

### Discovery

After logging in over SSH as `msfadmin` (credentials obtained via offline hash cracking, see prior writeup), sudo permissions for the account were enumerated:

```
sudo -l
```

Output:

```
User msfadmin may run the following commands on this host:
    (ALL) ALL
```

This line, read from `/etc/sudoers` syntax, means `msfadmin` may run **any command** (`ALL` in the command position) **as any user**, including root (`ALL` in the run-as position). No `NOPASSWD` flag was present, meaning the account's own password is required, but no command restrictions exist at all.

### Exploitation

With unrestricted command access confirmed, escalation required no exploit, only the intended (but misconfigured) use of `sudo`:

```
sudo bash
```

After re-authenticating with the `msfadmin` account password (sudo's standard behavior, even for an already-logged-in session, to confirm the active user is still present at the keyboard), an interactive root shell was returned:

```
root@metasploitable:~#
```

Confirmed with:

```
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

### Impact

Any account with this sudoers configuration is functionally equivalent to root, regardless of its intended purpose or restrictions elsewhere on the system. In this case, a standard user account reachable via SSH, and protected only by a weak, guessable password, provided a direct and trivial path to full administrative control. This finding compounds the severity of the separate weak-credential finding: the two combined mean a single guessed password equals total system compromise.

### Remediation

- Apply the principle of least privilege: grant sudo access only to the specific commands an account actually requires, e.g.:
  ```
  msfadmin ALL=(root) /usr/bin/systemctl restart apache2
  ```
  rather than a blanket `(ALL) ALL` grant.
- Audit `/etc/sudoers` and any files under `/etc/sudoers.d/` regularly for overly permissive rules.
- Remove sudo access entirely from accounts that do not require administrative capability.
- Pair this with strong, unique credentials and ideally key-based SSH authentication, since a sudo misconfiguration is only as dangerous as the ease of compromising the account that holds it.

## Lessons learned

- Privilege escalation isn't always a technical exploit. Checking `sudo -l` immediately after gaining any shell access costs nothing and frequently reveals a direct, intended path to full control via simple misconfiguration.
- Findings compound. A weak password alone is a serious finding; a weak password on an account with unrestricted sudo is a critical one. Report findings individually, but also call out how they combine to increase real-world risk.
- `sudo`'s password re-prompt exists to verify the person currently at the keyboard, not just the account's on-paper permissions, which is why it can still prompt mid-session even for an already-authenticated user.
