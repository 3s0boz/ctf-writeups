# Cap - HackTheBox

Linux machine running a web dashboard that generates network captures on demand.
An IDOR vulnerability exposes another user's packet capture containing FTP credentials
in cleartext. Password reuse gives SSH access, and a Python binary with `cap_setuid`
provides a direct path to root.

Target: `10.129.27.185`

---

## Reconnaissance

```bash
nmap -sV -sC 10.129.27.185 -T5
```

Three open ports:

- FTP (21) - vsftpd
- SSH (22) - OpenSSH
- HTTP (80) - web dashboard with a "Security Snapshot" feature

---

## Enumeration

### IDOR - Accessing Other Users' Captures

The dashboard exposes a `/data/[id]` endpoint after clicking "Security Snapshot".
The ID is an auto-incrementing integer. Changing `/data/1` to `/data/0` in the URL
returns a different user's capture with no server-side ownership validation.

This is a textbook IDOR (Insecure Direct Object Reference) - sequential IDs with
no access control.

### PCAP Analysis - FTP Credentials in Cleartext

The PCAP downloaded from `/data/0` contains an FTP session. Opening it in Wireshark
and following the TCP stream reveals the full login sequence:

```
nathan : Buck3tH4TF0RM3!
```

FTP transmits credentials in plaintext - one captured session is enough to compromise
the account.

---

## Initial Access

The FTP credentials work for SSH (password reuse):

```bash
ssh nathan@10.129.27.185
```

User flag is in `/home/nathan/user.txt`.

---

## Privilege Escalation - cap_setuid

Enumerating Linux capabilities with linPEAS (transferred via Python HTTP server)
reveals a dangerous capability on the Python binary:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` allows a process to change its own UID. On an interpreter like Python,
this means instant root:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

```
# id
uid=0(root) gid=1000(nathan)
```

Root flag is in `/root/root.txt`.

---

## Key Takeaways

- IDOR with sequential IDs and no server-side validation is one of the most common
  web vulnerabilities (OWASP Top 10 - Broken Access Control). Always test incrementing
  and decrementing ID parameters.
- FTP transmits everything in cleartext. A single intercepted session exposes
  credentials, files, and commands. In production, use SFTP or SCP.
- Linux capabilities are an underrated privilege escalation vector. `cap_setuid` on
  any interpreter (Python, Perl, Ruby) is equivalent to SUID root. `getcap -r /` should
  be part of every post-exploitation enumeration alongside `sudo -l` and SUID checks.
- Password reuse across services on the same host (FTP to SSH) is a pattern that
  appears constantly in both CTFs and real assessments.

---

## Disclaimer

This box was completed in the controlled, legal environment provided by HackTheBox.
All actions were performed strictly for educational purposes.
