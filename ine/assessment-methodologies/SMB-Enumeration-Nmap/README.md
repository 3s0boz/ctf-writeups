# SMB Enumeration with Nmap - INE eJPT

INE lab focused exclusively on SMB enumeration using Nmap NSE scripts against a Windows target. The objective is to extract protocol details, security configuration, active sessions, shares, users, groups, services, and filesystem content - both with guest access and with valid credentials.

Target: `demo.ine.local` - Credentials provided: `administrator : smbserver_771`

---

## Enumeration

### Host Discovery and Initial Port Scan

```bash
ping -c5 demo.ine.local
nmap demo.ine.local
```

SMB exposed on port 445.

### Protocol Identification

```bash
nmap -p445 --script smb-protocols demo.ine.local
```

Identifies SMB dialect versions supported by the target.

### Security Mode

```bash
nmap -p445 --script smb-security-mode demo.ine.local
```

Result: guest login enabled - a critical misconfiguration allowing unauthenticated enumeration.

### Session Enumeration

Without credentials:

```bash
nmap -p445 --script smb-enum-sessions demo.ine.local
```

User `bob` identified as an active guest session.

With credentials:

```bash
nmap -p445 --script smb-enum-sessions \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

### Share Enumeration

Without credentials:

```bash
nmap -p445 --script smb-enum-shares demo.ine.local
```

Guest access to multiple shares. `IPC$` accessible with read/write permissions.

With credentials:

```bash
nmap -p445 --script smb-enum-shares \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

`C$` with full access for administrator.

### User Enumeration

```bash
nmap -p445 --script smb-enum-users \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

Users found: `Administrator`, `bob`, `Guest`

### Server Statistics

```bash
nmap -p445 --script smb-server-stats \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

Exposes failed login counts, open file handles, and error rates - useful for post-exploitation situational awareness.

### Domains, Groups, and Services

```bash
nmap -p445 --script smb-enum-domains \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local

nmap -p445 --script smb-enum-groups \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local

nmap -p445 --script smb-enum-services \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

### Share Content Listing

```bash
nmap -p445 --script smb-enum-shares,smb-ls \
  --script-args smbusername=administrator,smbpassword=smbserver_771 \
  demo.ine.local
```

Lists directory contents of accessible shares - direct input for lateral movement planning.

---

## Key Takeaways

- NSE scripts allow deep SMB enumeration without triggering exploitation. The entire attack surface is visible from a single authenticated scan.
- Guest login enabled on SMB is a critical misconfiguration. Active sessions, share permissions, and usernames are all exposed without credentials.
- `IPC$` with read/write permissions is frequently overlooked but is a lateral movement enabler.
- Credentials amplify visibility significantly. The same scripts with authentication expose `C$`, user lists, domain membership, and running services.
- SMB enumeration is a mandatory pre-exploitation step. The data collected here feeds directly into brute force targeting, PsExec exploitation, and lateral movement planning.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
