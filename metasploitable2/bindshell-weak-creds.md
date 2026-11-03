# Metasploitable 2: Unauthenticated Root Bind Shell & Weak Credentials

**Target:** Metasploitable 2 (`10.0.130.10`)
**Environment:** Isolated home lab (Proxmox, pfSense-segmented VLAN)
**Date:** November 2, 2026
**Tester:** Russ

## Summary

During enumeration of a Metasploitable 2 instance, Nmap identified a service on TCP port 1524 fingerprinted as `bindshell`. Connecting to the port with netcat returned an unauthenticated root shell with no exploitation required. From that shell, `/etc/shadow` was read and a password hash for the `msfadmin` account was cracked in under one second, revealing a classic weak-credential pattern (`msfadmin:msfadmin`).

This chain demonstrates two separate but related findings: an unauthenticated backdoor providing full system compromise, and a weak/reused credential that would independently compromise the account on any service where it was valid.

## Scope and environment

All testing was performed against a vulnerable-by-design VM (Metasploitable 2) running in an isolated lab VLAN with no route to production systems, management infrastructure, or the internet. No real-world systems were tested.

## Methodology

1. Host discovery and port scanning with Nmap
2. Service and version enumeration (`-sC -sV`)
3. Manual connection to an anomalous open port
4. Local privilege/credential enumeration from the resulting shell
5. Offline hash cracking to validate credential strength

## Finding 1: Unauthenticated Root Bind Shell (TCP/1524)

### Discovery

A full port sweep followed by service/version detection flagged port 1524 with an unusual fingerprint:

```
1524/tcp open bindshell Metasploitable root shell
```

Nmap's service-detection probes recognized the response pattern as a raw command shell rather than a standard protocol, and its fingerprint database labeled it accordingly.

### Exploitation

Connected directly with netcat, no credentials or exploit required:

```
nc 10.0.130.10 1524
```

The connection immediately returned an interactive shell. Verified access level:

```
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

### Impact

Complete, unauthenticated compromise of the host. Any party able to reach TCP/1524 has full root-level command execution with no authentication barrier at all. This is the most severe class of finding possible, equivalent to a system with no access control whatsoever on that port.

### Remediation

- Identify and remove the backdoor process and its persistence mechanism
- Treat the host as fully compromised; do not trust its integrity going forward, rebuild from a known-clean image
- Audit how the backdoor was introduced (supply chain, prior compromise, insider threat) before returning the system to service
- Restrict and monitor all listening ports; alert on any unexpected open port

## Finding 2: Weak / Reused Credentials (`msfadmin:msfadmin`)

### Discovery

With root access from Finding 1, local user accounts were enumerated via `/etc/passwd`, identifying three interactive (non-service) accounts: `msfadmin`, `user`, and `service`. The corresponding password hash was then read from `/etc/shadow`:

```
msfadmin:$1$XN10Zj2c$Rt/zzCW3mLtUWA.ihZjA5/:14684:0:99999:7:::
```

The `$1$` prefix identifies the hash as MD5-crypt, with the following `$`-delimited segment as the salt and the remainder as the hash output.

### Exploitation

The hash was extracted into a local file and attacked offline with John the Ripper using its default wordlist:

```
john /home/zooka/Desktop/hash
```

Result: cracked in under one second.

```
john --show /home/zooka/Desktop/hash
# msfadmin:msfadmin
```

### Impact

The `msfadmin` account uses its own username as its password. Beyond being trivially guessable, credential reuse means this password should be tested against every other authenticated service identified in the initial port scan (SSH, FTP, MySQL, PostgreSQL, Samba, etc.), since a single weak credential often compromises multiple services at once.

### Remediation

- Enforce a password policy that rejects username-as-password and other low-entropy patterns
- Apply account lockout after a small number of failed authentication attempts
- Use salted, modern hashing algorithms (the host's use of MD5-crypt is itself outdated; SHA-512-crypt or stronger is preferred)
- Avoid credential reuse across accounts and services

## Lessons learned

- Service enumeration can surface findings (like an open bind shell) that require no exploitation at all; always investigate unusual fingerprints rather than only well-known ports.
- Algorithm weakness (e.g., MD5) and password weakness are independent variables. A fast algorithm only matters if the underlying password is itself guessable or in a common wordlist; this account would have cracked instantly on nearly any hashing algorithm.
- Enumeration provides intelligence beyond the immediate vulnerability. Context gathered along the way (e.g., observing login behavior) can inform which credentials are worth testing first.
